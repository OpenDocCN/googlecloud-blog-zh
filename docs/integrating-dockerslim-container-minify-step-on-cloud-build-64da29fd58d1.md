# 在云构建上集成 DockerSlim 容器缩小步骤

> 原文：<https://medium.com/google-cloud/integrating-dockerslim-container-minify-step-on-cloud-build-64da29fd58d1?source=collection_archive---------0----------------------->

每当我们在处理集装箱时，一般的建议是使它们的 ***更瘦***更小********更安全*** 。容器映像越小，应用程序第一次启动的速度就越快，扩展的速度也越快。*

*但更多关于这个话题，业界权衡拥有更小容器的好处，更安全的 ***因为减少了可用的攻击面*** 。*

*在 Cloud Run 这样的无服务器容器环境中，拥有一个更精简的容器映像有助于减少冷启动。*