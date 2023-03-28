# 制作 Newtonsoft。Json 和协议缓冲区配合得很好。

> 原文：<https://medium.com/google-cloud/making-newtonsoft-json-and-protocol-buffers-play-nicely-together-fe92079cc91c?source=collection_archive---------0----------------------->

我在尝试将一些[协议消息](https://developers.google.com/protocol-buffers/)序列化到 json 时遇到了一个问题。当 [Newtonsoft。Json](https://www.newtonsoft.com/json) 反序列化了我的协议消息，它与原始消息不匹配。我找到了一个简单的工作，可能对你也有用。

# 协议缓冲区

[协议缓冲区](https://developers.google.com/protocol-buffers/)是 Google 的语言中立、平台中立、可扩展的机制，用于序列化结构化数据——比如 XML，但是更小、更快、更简单。一旦定义了数据的结构化方式，就可以使用生成的源代码轻松地编写和读取结构化数据。协议缓冲区也知道如何像 json 一样读写自己。下面是一个协议消息定义示例:

我正在为一个新的 [Google Stackdriver](https://cloud.google.com/stackdriver/) API 编写样本。几乎所有 Google 的云 API 都使用协议缓冲区作为它们的网络格式。

我有一个名为`AlertPolicy`的协议消息。我想把它作为 json 序列化到磁盘上，这样就可以很容易地备份和恢复我所有的警报策略。协议缓冲库可以很容易地为我序列化一个`AlertPolicy`，但是它不能序列化一个`IEnumerable<AlertPolicy>`。更一般地说，协议缓冲库知道如何将协议消息序列化为 json，但是被序列化的根对象必须是协议消息。

所以，我看了 Newtonsoft.Json。

# 纽顿软件。Json

[牛顿软件。Json](https://www.newtonsoft.com/json) 是用[序列化和反序列化](https://www.microsoft.com/net/) [json](https://www.json.org/) 最流行的库。网。这也是最受欢迎的一个原因:这个库使得对象与 json 之间的相互转换变得很简单。将对象转换成 json 字符串只需要一行代码，`JsonConvert.SerializeObject(yourObject)`:

所以，我试着用 Newtonsoft.Json 序列化我的`IEnumerable<AlertPolicy>`。我发现的问题是，当我反序列化`IEnumerable<AlertPolicy>`并将其与我最初的`IEnumerable<AlertPolicy>`进行比较时，一大块数据丢失了。

之前:

之后:

幸运的是，牛顿软件。Json 允许我指定一个定制的 JSON 转换器。我实现了一个定制的 JsonConverter，它使用 Protocol Buffers 的 json 序列化函数来避免丢失数据。因此，30 行代码之后，我有了一个序列化所有协议消息的解决方案:

我是这样调用它的:

所以，耶！我可以将这个`ProtoMessageConverter`用于任何包含协议消息对象的对象，一切都会正确地序列化和反序列化。你也可以。完整的样本代码保存在 github.com 的 T4。