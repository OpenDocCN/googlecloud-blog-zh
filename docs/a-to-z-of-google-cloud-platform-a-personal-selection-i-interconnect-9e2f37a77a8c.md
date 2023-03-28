# 谷歌云平台的 a 到 Z 个人精选— I —互联

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-i-interconnect-9e2f37a77a8c?source=collection_archive---------1----------------------->

是的，我知道我让你和我以前的帖子挂在一起，但请放心，我已经写了 J，但“我”确实在下面，所以你有它。(我把它保持得很短)不能离开网络，这是你的另一个条目！

互联是一种将 GCP 与您的内部 DC 或潜在的其他云提供商直接连接的方式。这个标题下有三种产品

[运营商互连](https://cloud.google.com/interconnect/) —这是一种通过使用互连服务提供商将您的网络连接到 Google 网络边缘的方式，互连服务提供商将与您一起连接您的网络，无论是在内部还是已经在一些互连服务提供商运营的数据中心内，都可以连接到他们的(因为缺乏更好的描述)交换点，而交换点又直接连接到 Google 的网络。

[云 VPN](https://cloud.google.com/compute/docs/vpn/)——这允许你在两个独立的网络之间建立一个 IPsec VPN。这些网络可以是位于不同区域的两个独立的 GCE 网络；本地网络到 GCE 网络，甚至是 GCE 网络和另一个云之间的网络。应注意，在所有场景中，计算引擎 VPN 仅支持网关到网关场景。客户端必须有专用的物理或虚拟 VPN 网关。

[直接对等](https://cloud.google.com/interconnect/direct-peering) g -你的商业网络和谷歌的直接对等[连接。它允许在您的网络和谷歌的一个覆盖广泛的边缘网络位置之间交换互联网流量。它在 Google 和对等实体之间交换 BGP 路由](https://www.wikipedia.org/wiki/Peering)

你们当中的网络 BOD 将立即理解为什么你有选择，事实上，你可以将云 VPN 与运营商互连或直接对等相结合。原因是它们用于处理不同的任务

运营商互连和直接对等就是通过提供更高程度的可用性和更好的延迟来克服穿越互联网的各种变化，这些变化超出了谷歌或你的控制。

VPN 旨在提供与 GCP 的安全连接

好的，下周我们将回到我承诺的静态托管文章的第二部分。