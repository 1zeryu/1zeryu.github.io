---
permalink: /
title: ""
excerpt: ""
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

{% if site.google_scholar_stats_use_cdn %}
{% assign gsDataBaseUrl = "https://cdn.jsdelivr.net/gh/" | append: site.repository | append: "@" %}
{% else %}
{% assign gsDataBaseUrl = "https://raw.githubusercontent.com/" | append: site.repository | append: "/" %}
{% endif %}
{% assign url = gsDataBaseUrl | append: "google-scholar-stats/gs_data_shieldsio.json" %}

<span class='anchor' id='about-me'></span>

  I am now an algorithm researcher at an AI startup GigaAI, working under the supervision of <a href="http://www.zhengzhu.net/">Zheng Zhu</a> and <a href="https://scholar.google.com/citations?hl=zh-CN&user=a5nrMUkAAAAJ">Xinze Chen</a>. In June 2025, I obtained my Bachelor's degree from <a href="https://www.xidian.edu.cn/">Xidian University</a>. During my undergraduate studies, I worked as a research intern in <a href="https://ai.comp.nus.edu.sg/">NUS-HPC-AI-Lab</a>. I also interned at the <a href="https://www.datagrand.com">DataGrand</a> with Dr. <a href="https://normxu.github.io/about/">Nuo Xu</a>.

  My research interests include **world models**, **reinforcement learning** and **embodied intelligence**. I hope to have the opportunity to explore the integration of world models for comprehensive environmental data and reinforcement models for intelligent decision-making, and to make modest contributions to the development of AI systems.

  I am exploring potential PhD opportunities in my field of interest. Feel free to reach out at [email](mailto:ykzhou8981389@gmail.com)!


<!-- # 🔥 News
- *2025.07*: &nbsp; .  -->

# 📝 Selected Public Work

<!-- ## Projects

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">Close-Source Project</div><img src='images/opensora_demo.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[GigaBrain](https://mp.weixin.qq.com/s/qiWG2KETNnzbn0NpLeyt-w)

Project participant as Algorithm Engineer of GigaAI.

[<i class="fab fa-fw fa-github" aria-hidden="true"></i> **Preview**](https://mp.weixin.qq.com/s/qiWG2KETNnzbn0NpLeyt-w)
- Open-Sora is an open-source video generation model designed to produce high-fidelity video content. We democratizes full access to all the training/inference/data preparation codes as well as model weights
</div>
</div> -->

## Publications 

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">CVPR 2025</div><img src='images/humandreamer.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[HumanDreamer: Generating Controllable Human-Motion Videos
via Decoupled Generation](https://openaccess.thecvf.com/content/CVPR2025/papers/Wang_HumanDreamer_Generating_Controllable_Human-Motion_Videos_via_Decoupled_Generation_CVPR_2025_paper.pdf)

Boyuan Wang, et al, **Yukun Zhou**, et al.

[<i class="fa fa-solid fa-cube" aria-hidden="true"></i> **Project**](https://humandreamer.github.io/) &nbsp; [<i class="fas fa-fw fa-database" aria-hidden="true"></i> **Dataset**](https://huggingface.co/datasets/chuanshuogushi/MotionVid) &nbsp; [<i class="fas fa-fw fa-file-alt" aria-hidden="true"></i> **Paper**](https://openaccess.thecvf.com/content/CVPR2025/papers/Wang_HumanDreamer_Generating_Controllable_Human-Motion_Videos_via_Decoupled_Generation_CVPR_2025_paper.pdf)
- HumanDreamer is a decoupled human video generation framework that first generates diverse poses from text prompts and then leverages these poses to generate human-motion videos.
</div>
</div>

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">CVPR 2025</div><img src='images/Speed_demo.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[A Closer Look at Time Steps is Worthy of Triple Speed-Up for Diffusion Model
Training](https://openaccess.thecvf.com/content/CVPR2025/papers/Wang_A_Closer_Look_at_Time_Steps_is_Worthy_of_Triple_CVPR_2025_paper.pdf)

Kai Wang, Mingjia Shi, **Yukun Zhou**, et al.

[<i class="fa fa-solid fa-cube" aria-hidden="true"></i> **Project**](https://bdemo.github.io/SpeeD/) &nbsp; [<i class="fab fa-fw fa-github" aria-hidden="true"></i> **Code**](https://github.com/NUS-HPC-AI-Lab/SpeeD) &nbsp; [<i class="fas fa-fw fa-file-alt" aria-hidden="true"></i>  **Paper**](https://openaccess.thecvf.com/content/CVPR2025/papers/Wang_A_Closer_Look_at_Time_Steps_is_Worthy_of_Triple_CVPR_2025_paper.pdf)
- As a plug-and-play and architectureagnostic approach, SpeeD consistently achieves 3× acceleration across various diffusion architectures, datasets, and tasks.
</div>
</div>

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">ECCV 2024</div><img src='images/dataset_growth.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[Dataset Growth](https://link.springer.com/chapter/10.1007/978-3-031-72673-6_4)

Ziheng Qin, Zhaopan Xu, **Yukun Zhou**, et al.

[<i class="fab fa-fw fa-github" aria-hidden="true"></i> **Code**](https://github.com/NUS-HPC-AI-Lab/InfoGrowth?tab=readme-ov-file) &nbsp; [<i class="fas fa-fw fa-file-alt" aria-hidden="true"></i> **Paper**](https://arxiv.org/pdf/2405.18347)
- InfoGrowth can improve data quality/efficiency on both single modal and multi-modal tasks, with an efficient and scalable design.
</div>
</div>


## Preprints

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">Preprint</div><img src='images/gigavideo-1.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[GigaVideo-1: Advancing Video Generation via
Automatic Feedback with 4 GPU-Hours Fine-Tuning](https://arxiv.org/pdf/2506.10639)

Xiaoyi Bao, et al, **Yukun Zhou**, et al.

[<i class="fa fa-solid fa-cube" aria-hidden="true"></i> **Project**](https://gigavideo-1.github.io/) &nbsp; [<i class="fas fa-fw fa-file-alt" aria-hidden="true"></i> **Paper**](https://arxiv.org/pdf/2506.10639)
- GigaVideo-1 consistently improves performance on almost all the dimensions with an average gain of ~4% using only 4 GPU-hours.
</div>
</div>

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">Preprint</div><img src='images/p-diff.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[Neural Network Parameter Diffusion](https://arxiv.org/pdf/2402.13144v2)

Kai Wang, Zhaopan Xu, **Yukun Zhou**, et al.

[<i class="fab fa-fw fa-github" aria-hidden="true"></i> **Code**](https://github.com/NUS-HPC-AI-Lab/Neural-Network-Diffusion) &nbsp; [<i class="fas fa-fw fa-file-alt" aria-hidden="true"></i>  **Paper**](https://arxiv.org/pdf/2402.13144v2)
- We demonstrate that diffusion models can also generate high-performing neural network parameters.
</div>
</div>


<!-- # 🎖 Honors and Awards
- *2021.10* Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus ornare aliquet ipsum, ac tempus justo dapibus sit amet. 
- *2021.09* Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus ornare aliquet ipsum, ac tempus justo dapibus sit amet.  -->

## Projects

<div class='paper-box'><div class='paper-box-image'><div><div class="badge">Open-Source Project</div><img src='images/opensora_demo.png' alt="sym" width="100%"></div></div>
<div class='paper-box-text' markdown="1">

[Open-Sora: Democratizing Efficient Video Production for All](https://github.com/hpcaitech/Open-Sora)

Project participant as intern of Luchen Technology.

[<i class="fab fa-fw fa-github" aria-hidden="true"></i> **Code**](https://github.com/hpcaitech/Open-Sora) &nbsp; [<i class="fas fa-fw fa-file-alt" aria-hidden="true"></i>  **Paper**](https://arxiv.org/pdf/2412.20404) &nbsp; [<i class="fas fa-project-diagram"></i> **Model**](https://huggingface.co/collections/hpcai-tech/open-sora-66373960c8950692a4394e2d) &nbsp; [<i class="fas fa-fw fa-database" aria-hidden="true"></i> **Dataset**](https://huggingface.co/datasets/hpcai-tech/open-sora-pexels-45k)
- Open-Sora is an open-source video generation model designed to produce high-fidelity video content. We democratizes full access to all the training/inference/data preparation codes as well as model weights
</div>
</div>



# 📖 Educations
- *2021.09 - 2025.06*, School of Computer Science and Technology, Xidian University.

<!-- # 💬 Invited Talks
- *2021.06*, Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus ornare aliquet ipsum, ac tempus justo dapibus sit amet. 
- *2021.03*, Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus ornare aliquet ipsum, ac tempus justo dapibus sit amet.  \| [\[video\]](https://github.com/) -->

# 💻 Internships
- *2023.07-2023.10*, [Data Grand](https://www.datagrand.com/), China.
- *2024.05-2024.07*, [Luchen Technology](https://www.luchentech.com/), China.
- *2024.08-2025.06*, [GigaAI](https://gigaai.cc/), China.

<!-- # 👍 Friends
-  -->

 &nbsp;

<div style="max-width:300px; margin:auto;">
  <script type="text/javascript" id="mapmyvisitors" src="//mapmyvisitors.com/map.js?d=uoPj1f-PAOoHtVcnNm8UsbEN95b7HfMvqBaKGxMNv-U&cl=ffffff&w=300"></script>
</div>
<!-- <script type="text/javascript" id="mmvst_globe" src="//mapmyvisitors.com/globe.js?d=mDd3qmhvqv64vdye5bjCkzmKat0KqufhB1RJXbJ1a9w"></script> -->
<!-- <script type='text/javascript' id='clustrmaps' src='//cdn.clustrmaps.com/map_v2.js?cl=ffffff&w=400&t=tt&d=Dv2XJvfFsJgeunE6i3OYfBPY7jcGv2zSILN4rwgNYH8&co=2d78ad&ct=ffffff&cmo=3acc3a&cmn=ff5353'></script> -->