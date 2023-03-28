# gRPC 代码转换

> 原文：<https://medium.com/google-cloud/grpc-transcoding-e101cc53d51d?source=collection_archive---------1----------------------->

我正在开发一些受益于 gRPC 的代码。稍微跑题一下(提前考虑，以便于测试)，我想探索 gRPC-JSON 代码转换。

谁知道有这么多选择:

*   [特使 gRPC-JSON 转码](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/grpc_json_transcoder_filter)
*   [grpc-网关](https://grpc-ecosystem.github.io/grpc-gateway/)
*   [grpc-web](https://github.com/grpc/grpc-web)
*   谷歌云端点([链接](https://cloud.google.com/endpoints/docs/grpc/about-grpc))

salmaan rashid (和我)是 Envoy 的粉丝，他推荐我尝试它的 gRPC-JSON 转码器。它工作了，但是我花了一段时间让我的配置工作，我仍然不能让其余的调用成功。

> **更新 2019-05-19**:REST 通话现在正常。见故事结尾。

希望这能帮助你避免同样的陷阱。

总之——在 Linux 上——我为`envoy.yaml`找到了这个工作组合:

```
cluster_name: grpc
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: **0.0.0.0**
            port_value: 50051
```

> 我认为`host.docker.internal`在 Linux 上不起作用，但是这个解决方案应该可以跨平台工作。

然后，最简单的方法是使用以下命令与容器共享主机网络:

```
docker run \
--interactive --tty --rm 
--**net=host** \
--volume=${WORK_DIR}/envoy.yaml:/etc/envoy/envoy.yaml \
--volume=${WORK_DIR}/envoy/proto.pb:/tmp/envoy \
envoyproxy/envoy
```

> **注意**调整卷安装，以反映`envoy.yaml`文件的位置和原始描述符集的路径(`proto.pb`)。不要忘记生成原型描述符集，这对我来说是新的。

gRPC

为了调试这个，我使用了`[gRPCurl](https://github.com/fullstorydev/grpcurl)`,如果你和我一样，错过了卷曲 gRPC 服务的能力，这是一个有用的工具。使用它，您可以:

```
HELLOWORLD=${GOPATH}/src/google.golang.org/grpc/examples/helloworld
GOOGLEAPIS=${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapisgrpcurl \
--plaintext \
--import-path=${HELLOWORLD} \
--import-path=${GOOGLEAPIS} \
--proto=helloworld/helloworld.proto \
**${SERVER}** \
**helloworld.Greeter/SayHello**
{
  "message": "Hello "
}
```

> **NB**可以是直接的，例如`:5**0**051`或通过代理`:5**1**051`。关于增加`${GOOGLEAPIS}`的原因，见下文休息部分。

您还可以:

```
grpcurl \
...
${SERVER} \
**list**
helloworld.Greetergrpcurl
...
${SERVER} \
**list helloworld.Greeter**
helloworld.Greeter.SayHello
```

虽然这些结果不是由服务器提供的，而是通过自省提供的原型。

**休息**

嗯…..还没工作:-(

但是，我确信代码转换配置正确，因为我可以通过它路由 gRPC 流量。

我没有注意到在定义 REST 映射的 Envoy 文档中添加到原型的注释:

```
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {
 **option (google.api.http) = {
      get: "/say"
    };**  }
}
```

因此，在生成原型描述符集之前添加这些，然后在运行`protoc`时需要引用 Google API:

```
HELLOWORLD=${GOPATH}/src/google.golang.org/grpc/examples/helloworld
GOOGLEAPIS=${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapisprotoc \
--proto_path=${HELLOWORLD} \
**--proto_path=${GOOGLEAPIS}** \
--include_imports \
--include_source_info   \
--descriptor_set_out=proto.pb \
helloworld/helloworld.proto
```

但是，尽管我尽了最大努力(文档中似乎缺少这方面的内容)，我还是不能`curl``:51051`端点:

```
curl \
--request GET \
--header "Content-Type: application/json" \
--data '{"name":"Freddie"}' \
http://0.0.0.0:51051/**helloworld.Greeter/SayHello**curl \
--header "Content-Type: application/json" \
--data '{"name":"Freddie"}' \
http://0.0.0.0:51051/**say**curl \
--request GET \
--header "Content-Type: application/json" \
[http://0.0.0.0:51051/](http://0.0.0.0:51051/helloworld.Greeter/SayHello)say?name=Freddieupstream connect error or disconnect/reset before headers. reset reason: remote reset
```

鉴于:

```
grpcurl \
--plaintext \
--import-path=${HELLOWORLD} \
--import-path=${GOOGLEAPIS} \
--proto=helloworld/helloworld.proto \
-d '{"name":"Freddie"}' \
:51051 \
helloworld.Greeter/SayHello
{
  "message": "Hello Freddie"
}
```

一个相关的困难是，Envoy 代理(按照配置)没有提供太多的报告，因此我没有信息来指导我找到解决方案。

我会更新这个故事——要么当我意识到，要么当有人指出——我的错误。

**工作！**

我在另一台机器上再现了该解决方案，并且能够通过 Envoy 代理成功地发出请求:

```
curl \
--header "Content-Type: application/**json**" \
[http://localhost:51051/say?name=Frederik](http://localhost:51051/say?name=Frederik)
{
 "message": "Hello Frederik"
}curl [http://localhost:51051/say?name=Frederik](http://localhost:51051/say?name=Frederik)
{
 "message": "Hello Frederik"
}
```

以下变体*不*工作:

```
curl \
--header "Content-Type: application/**grpc**" \
[http://localhost:51051/say](http://localhost:51051/say)curl \
--**data** '{"name":"Henry"}' \
[http://localhost:51051/say](http://localhost:51051/say)
upstream connect error or disconnect/reset before headers. reset reason: remote reset
```

我确信另一台机器上有网络(配置)问题。