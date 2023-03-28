# 用负载平衡器和 CDN 在 GCP 上托管一个静态网站

> 原文：<https://medium.com/google-cloud/host-a-static-website-on-gcp-with-loadbalancer-and-cdn-e1ce71d38d07?source=collection_archive---------1----------------------->

# 介绍

在我的上一篇文章中，我们讨论了如何使用 Google 管理的 SSL 证书在 Google Cloud 上配置负载平衡器。然后有人问我有没有可能用 loadbalancer 和 CDN 托管一个静态网站？所以才会有这篇文章。

# 加拿大

你们有些人可能不知道 CDN(内容分发网络)是什么，我来简单解释一下。根据[谷歌文档](https://cloud.google.com/cdn)，云 CDN 使用谷歌的全球边缘网络来提供更接近用户的内容，这加速了…