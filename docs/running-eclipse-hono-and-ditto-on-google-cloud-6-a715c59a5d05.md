# 在谷歌云上运行 Eclipse HONO 和 Ditto)

> 原文：<https://medium.com/google-cloud/running-eclipse-hono-and-ditto-on-google-cloud-6-a715c59a5d05?source=collection_archive---------4----------------------->

现在我们已经验证了与 HTTP 客户端的端到端连接，如果我的设备使用 MQTT 会怎样？

Eclipse HONO 支持 MQTT 3.1.1，要向 MQTT 端点发布遥测数据，有一些事情需要额外注意。

*   **clientId:** 设备的客户端 Id 应该是`gcp-nnection-name` + p，比如你按照这个教程，连接名应该是`gcp-connection-org-eclipse-ditto`，那么你的 MQTT 客户端中指定的 clientId 应该是`gcp-connection-org-eclipse-dittop`
*   **用户名:**设备的用户名应该是${DEVICE-AUTH}@${TENANT}，如果您按照教程操作，应该是`demo-device-001-auth@org.eclipse.ditto`，其中`demo-device-001`是设备标识符，`org.eclipse.ditto`是 HONO 租户名称。

您的 MQTT 连接选项应该与此类似

```
const connectionArgs = {
host: ‘YOUR MQTT HOST URL’,
port: 1883,
clientId: ‘gcp-connection-org-eclipse-ditto’ + ‘p’,
username: `demo-device-001-auth@org.eclipse.ditto`, 
password: ‘my-password’,
protocol: ‘mqtt’
};
```

要将遥测发布到 Eclipse HONO，您可以发布到主题`t/?content-type=application/json`，其中

*   `t`是认证设备的主题名称。更多详情请参考 [Eclipse 文档](https://www.eclipse.org/hono/docs/user-guide/mqtt-adapter/)。
*   `/?content-type=application/json`是可选的，指定遥测是以 JSON 格式。

要订阅来自 Eclipse 的通知，例如，发送到边缘设备的命令，您可以订阅`command///req/#`