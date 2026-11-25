---
title: "视频数据的标称帧率和平均帧率不一致问题"
description: "把 Lance 源数据转成 LeRobot v3 时，save_episode 报 constant FPS 错；根因是 mp4 容器里标称帧率与真实帧率对不上，lerobot 又恰好只信那个错标的值"
slug: lerobot-reference-fps
date: 2026-11-25 10:00:00+0800
draft: false
image: ""
categories:
    - bug
tags:
    - lerobot
authors:
  - "Yukun Zhou"
type: post
---

mp4[^mp4] 不像早期的 AVI，没有一个明文写着"帧率 = xx"的字段。它把时间拆成两块存：

- 一个"时间刻度"（`timescale`），定义一秒被切成多少格。
- 一张"每帧占几格"的清单（`stts`），为了省空间，它记成"接下来 N 帧，每帧占 D 格"。

每一帧在时间轴上的位置，就是把这些格数逐帧累加、再除以时间刻度得到的。单帧不携带"我在第几秒"，只有"我这一帧延续几格"，绝对时间是从第 0 帧一路累加推出来的。

有了"每帧占几格"，就能推出第一个帧率：标称帧率（`r_frame_rate`）= 时间刻度 ÷ 每帧格数。比如时间刻度是 60，每帧占 1 格，标称帧率就是 60/1 = 60；每帧占 4 格，就是 60/4 = 15。真实文件里的时间刻度通常大得多，常见的有 90000（MPEG 系列默认的 90kHz）、15360（15fps × 1024）、12800、30000、60000，或者干脆用 1000 表示毫秒精度。刻度大，每帧占的格数 D 也会按同样比例放大，两者相除出来的帧率不变。

mp4 里还另存了一个"总时长"（`mvhd` 里的 `duration`）。拿总帧数除以总时长，又得到第二个帧率：平均帧率（`avg_frame_rate`）。它跟标称帧率是两个独立算出来的数字。

还有第三个帧率根本不在文件里：采集帧率，也就是相机传感器实际出帧的频率，只有采集端知道。正常封装时，容器应该把每帧格数 D 设成 `时间刻度 ÷ 采集帧率`，让标称帧率和平均帧率都等于采集帧率。三者对不上，就说明封装或采集的某一环出了问题。

## BUG 现象

我们想把手上所有的数据转成 LeRobot v3 格式。转格式要对每段视频重编码一遍，写进 `lerobotv3-root/videos/`，又慢又占空间。

我们想跳过这一步，就给 LeRobot 加了 `video_mode="reference"`：不把视频转存进 `lerobotv3-root/videos/`，只在数据集里记一条引用，也就是视频根目录加相对路径，训练时直接 seek 原始 mp4。

```python
# scripts/data_processing/lance_to_lerobot_v3.py
fps = int(round(float(tbl["fps"][first])))          # 采集端记录的采样帧率或平均帧率
dataset = LeRobotDataset.create(
    repo_id=repo_id,
    fps=fps,                                          # 写进数据集元数据
    ...
    video_mode="reference",
    video_root=VIDEO_ROOT,
)
...
ep_fps = float(tbl["fps"][ep_idx])
video_refs[f"observation.images.cam{k}"] = ExternalVideoReference(
    source_path=rel_video_path(src),
    from_timestamp=0.0,
    to_timestamp=n_frames / ep_fps,                   # 同样由 ep_fps 推出
)
dataset.save_episode(video_refs=video_refs)
```

reference 模式会用到两个帧率，两者来源互不相干：

- 写数据集元数据时，用的是采集端记录的采样帧率或平均帧率，也就是上面那个 `fps`，它还决定了每条视频引用的时间区间。
- 读取视频 seek 时，LeRobot[^lerobot] 用 `av` 打开 mp4，从容器头反推一个 `video.fps`，也就是上面说的标称帧率：

```python
# lerobot/datasets/video_utils.py  get_video_info
video_info["video.fps"] = int(video_stream.base_rate)  # base_rate 就是 r_frame_rate
```

大部分视频这两个帧率一致，reference 模式没事。一旦对不上，`_query_videos` 会先用元数据 fps 生成采样时间戳，再拿去 seek 原始 mp4：

```python
# lerobot/datasets/lerobot_dataset.py  _query_videos
from_timestamp = ep[f"videos/{vid_key}/from_timestamp"]
shifted_query_ts = [from_timestamp + ts for ts in query_ts]   # query_ts 由元数据 fps 生成
frames = decode_video_frames(video_path, shifted_query_ts, self.tolerance_s, ...)
```

`decode_video_frames` 拿着这些按元数据 fps 生成的时间戳去容器里 seek 帧，可容器里每一帧的时间位置是按容器头的标称帧率排的。查询沿元数据 fps 的时间轴走，帧却待在标称帧率的时间轴上，两套时间轴错开，seek 不到对应的帧，就抛错：

```python
# lerobot/datasets/video_utils.py  decode_video_frames_torchvision
if not is_within_tol.all():
    raise FrameTimestampError(
        f"One or several query timestamps unexpectedly violate the tolerance ..."
    )
```

我们手里的数据有两类踩了这个坑：

- droid[^droid] 的容器头把标称帧率错标成 60（`stts` 每帧时长写成 1/60 秒），真实是 15。元数据 15、容器头 60，seek 直接错位。
- g2[^agibot2026] 的标称帧率统一标成 30，但每一集真实平均帧率在 8~23 之间浮动，本身就是可变帧率，再加上视频帧数和 action 帧数对不齐，比 droid 更麻烦，没法一刀切。

## 小结

突然明白了，看似更难的路，实际上可能是捷径。重编码虽然慢且占额外空间，但是只需要做一次，且能保证往后的正确性。


[^agibot2026]: AgiBot World 2026，智元机器人（AgiBot）发布的大规模真实世界操作数据集（含 ImitationLearning / RichInteraction 子集）。
[^droid]: Khazatsky A, Pertsch K, Nasiriany S, et al. DROID: A Large-Scale In-The-Wild Robot Manipulation Dataset. arXiv:2403.12945, 2024. https://arxiv.org/abs/2403.12945
[^lerobot]: Cadène R, Alibert S, Soare A, et al. LeRobot: State-of-the-art Robot Learning with Open Models. arXiv:2505.09613, 2025. https://arxiv.org/abs/2505.09613
[^mp4]: ISO/IEC 14496-12: ISO Base Media File Format（MP4 的 box 结构，含 `stts` / `mvhd`）. https://www.iso.org/standard/83102.html
