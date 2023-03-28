# 我如何学会在生产中停止担心和调试

> 原文：<https://medium.com/google-cloud/how-i-learned-to-stop-worrying-and-debug-in-production-cca099a2bccb?source=collection_archive---------0----------------------->

[事故管理](https://landing.google.com/sre/sre-book/chapters/managing-incidents/)是现场可靠性工程的核心实践之一。作为其中的一部分，SRE 的书建议把重点放在事件本身的优先化上。具体来说:

> ***分清主次*** *。止血，恢复服务，并保留根源的证据。*

但是，有时您可能仍然需要尝试在生产环境中进行调试，例如，您可能正在努力在本地或开发环境中重现某个问题，而生产环境可能是发生该问题的唯一地方…