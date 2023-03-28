# 使用云运行和 C++的云构建通知

> 原文：<https://medium.com/google-cloud/cloud-build-notifications-with-cloud-run-and-c-a23294e2e959?source=collection_archive---------2----------------------->

卡洛斯·奥赖安(谷歌)

我们正在将 google-cloud-cpp 的许多持续集成构建迁移到 GCB (Google Cloud Build ),因为它提供了高吞吐量、简单的配置以及与其他 GCP 服务的安全集成。虽然我们喜欢 GCB 中的大多数特性，但是我们缺少一种在完全构建(与拉请求构建相反)失败时通知我们的方式。本文描述了我们如何使用 Google 云服务和 C++客户端库解决这个问题。这里描述的所有东西的实际工作代码都可以在我们的 GitHub repo 中找到。

一些观察帮助我们总结出了这个解决方案:

*   GCB 为构建生命周期中的每个有趣事件(开始、成功完成、取消等)发送发布/订阅通知。)
*   我们可以使用 C++的函数框架来捕获云构建状态的变化，因为这些变化是通过云发布/订阅发送的
*   我们可以发送一个 HTTP POST 到一个特定的 URL，它将在 Google 聊天室中发布一条消息来提醒我们失败(而且仅仅是失败)。

这似乎也很有趣，因为我们将使用自己的功能框架来组合我们需要的功能。

# C++函数

这段代码中的主要入口点函数是 SendBuildAlerts()(关于如何配置该函数的名称，请参见下文)。当接收到新的 CloudEvent 时，函数框架会自动调用这个函数。它需要做的第一件事是获取聊天消息将被发布的秘密 URL(称为 webhook)。我们使用 [Google Secret Manager](https://cloud.google.com/secret-manager) 将这个 URL 注入到流程的环境中。

```
void SendBuildAlerts(google::cloud::functions::CloudEvent event) {
  static auto const webhook = [] {
    std::string const name = "GCB_BUILD_ALERT_WEBHOOK";
    auto const* env = std::getenv(name.c_str());
    if (env) return std::string{env};
    throw std::runtime_error("Missing environment variable: "
        + name);
  }();
… … … 
}
```

接下来，我们进行一些解析，从云发布/订阅消息中提取代表构建结果的 JSON 对象。你可以在 GitHub 上看到[的全部细节](https://github.com/googleapis/google-cloud-cpp/blob/9ce4ca8509e1be20d1a11badf8770f62f94aa3e2/ci/cloudbuild/notifiers/alerts/function/function.cc#L34-L50)，但是它非常简单，唯一不明显的地方是 Pub/Sub 消息有一个消息字段，包含 GCB 的[构建资源](https://cloud.google.com/build/docs/api/reference/rest/v1/projects.builds#Build)的 base64 编码表示。

```
void SendBuildAlerts(google::cloud::functions::CloudEvent event) {
… … … 
  auto const bs = ParseBuildStatus(std::move(event));
… … … 
}
```

然后，我们过滤掉不值得发出警报的事件，例如成功的构建或手动启动的构建:

```
void SendBuildAlerts(google::cloud::functions::CloudEvent event) {
… … …
  if (bs.status != "FAILURE") return;
  auto const substitutions = bs.build["substitutions"];
  auto const trigger_type = substitutions.value(
      "_TRIGGER_TYPE", "");
  auto const trigger_name = substitutions.value("TRIGGER_NAME", "");
  // Skips PR invocations and manually invoked builds (no trigger
  // name).
  if (trigger_type == "pr" || trigger_name.empty()) return;
… … … 
}
```

现在只需要向 webhook URL 发出 POST 请求:

```
void SendBuildAlerts(google::cloud::functions::CloudEvent event) {
… … … 
  auto const chat = MakeChatPayload(bs);
  std::cout << nlohmann::json{{"severity", "INFO"}, {"chat" : chat}}
            << "\n";
  HttpPost(webhook, chat.dump());
}
```

聊天消息本身是一个 JSON 对象:

```
nlohmann::json MakeChatPayload(BuildStatus const& bs) {
  auto const trigger_name = bs.build["substitutions"].value("TRIGGER_NAME", "");
  auto const log_url = bs.build.value("logUrl", "");
  auto text = fmt::format(
      "Build failed: *{}* {}", trigger_name, log_url);
  return nlohmann::json{{"text", std::move(text)}};
}
```

HTTP POST 是使用 libcurl 实现的:

```
void HttpPost(std::string const& url, std::string const& data) {
  static constexpr auto kContentType =
      "Content-Type: application/json";
  using Headers =
      std::unique_ptr<curl_slist, decltype(&curl_slist_free_all)>;
  auto const headers = Headers{
     curl_slist_append(nullptr, kContentType), curl_slist_free_all};
  using CurlHandle =
      std::unique_ptr<CURL, decltype(&curl_easy_cleanup)>;
  auto curl = CurlHandle(curl_easy_init(), curl_easy_cleanup);
  if (!curl) {
      throw std::runtime_error("Failed to create CurlHandle");
  }
  curl_easy_setopt(curl.get(), CURLOPT_URL, url.c_str());
  curl_easy_setopt(curl.get(), CURLOPT_HTTPHEADER, headers.get());
  curl_easy_setopt(curl.get(), CURLOPT_POSTFIELDS, data.c_str());
  CURLcode code = curl_easy_perform(curl.get());
  if (code != CURLE_OK) {
      throw std::runtime_error(curl_easy_strerror(code));
  }
}
```

# 获取和存储 Webhook

要为您的 Google 聊天室设置一个传入的 webhook，您可以按照[以下步骤进行](https://developers.google.com/hangouts/chat/how-tos/webhooks)。一旦你完成了这些，你就有了从你的云函数发布到的 URL(上面显示为 GCB_BUILD_ALERT_WEBHOOK 环境变量的内容)。您可以使用 curl(1)向该 URL 发送一篇测试文章，命令如下:

```
curl -X POST -sSL "<YOUR_WEBHOOK_URL>" \
    --data-binary "{'text': 'hello world'}"
```

这个网址应该保密，因为它允许任何人在你的谷歌聊天室发帖——**把它当作密码**。我们按照这个 [Secret Manager 快速入门指南](https://cloud.google.com/secret-manager/docs/quickstart#secretmanager-quickstart-web)中的说明将这个 URL 存储在 Google Secret Manager 中。接下来，我们需要将这个秘密安全地注入到我们的函数环境中。我们通过配置 [Google Cloud Run 来使用这个秘密](https://cloud.google.com/run/docs/configuring/secrets)来进行配置。瞧啊。

注意:我们在这里使用了一个 Google Chat Webhook，但是我们相信类似的方法也适用于 [Slack Webhooks](https://api.slack.com/messaging/webhooks) 或 [Discord Webhooks](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks) 。

# 部署 C++函数

functions framework for C++集成了 Cloud Run，这是一个托管的无服务器平台，您可以在其中将代码部署为一个容器。该平台调用您的代码来响应 web 请求或发布/订阅事件。该平台负责启动服务器，随着负载的增加增加服务器的数量，并在处理失败时重新传递消息。我们需要做的就是编写 SendBuildAlerts()函数，并将其包装到 Docker 映像中。

这里我们将逐个显示每个命令，但是我们有一个[小 shell 脚本](https://github.com/googleapis/google-cloud-cpp/blob/main/ci/cloudbuild/notifiers/alerts/deploy.sh)来运行它们。

方便的是，Google 的 build pack 支持 C++的函数框架，因此创建 Docker 映像是一个简单的命令:

```
readonly IMAGE="gcr.io/${GOOGLE_CLOUD_PROJECT}/send-build-alerts"
pack build --builder gcr.io/buildpacks/builder:latest \
  --env "GOOGLE_FUNCTION_SIGNATURE_TYPE=cloudevent" \
  --env "GOOGLE_FUNCTION_TARGET=SendBuildAlerts" \
  --path "function" "${IMAGE}"
```

这将检测我们函数的依赖项，下载、编译和安装它们，然后将我们的函数编译成 HTTP 服务。此外，依赖项是缓存的(本地，在您的工作站上)，所以额外的构建相当快。

一旦图像被创建，我们可以把它推到 GCR(谷歌容器注册):

```
docker push “${IMAGE}:latest”
```

并部署代码:

```
gcloud run deploy send-build-alerts \
    --project="${GOOGLE_CLOUD_PROJECT}" \
    --image="${IMAGE}:latest" \
    --region="us-central1" \
    --platform="managed" \
    --no-allow-unauthenticated
```

现在我们的服务器已经“运行”了，我们需要设置一个触发器来向它发送云构建消息，这是通过 Eventarc 完成的:

```
PROJECT_NUMBER=$(gcloud projects list \
    --filter="project_id=${GOOGLE_CLOUD_PROJECT}" \
    --format="value(project_number)" \
    --limit=1)gcloud beta eventarc triggers create send-build-alerts-trigger \
    --project="$GOOGLE_CLOUD_PROJECT" \
    --location="us-central1" \
    --destination-run-service="send-build-alerts" \
    --destination-run-region="us-central1" \
    --transport-topic="cloud-builds" \
    --matching-criteria="type=google.cloud.pubsub.topic.v1.messagePublished" \
    --service-account="${[PROJECT_NUMBER}-compute@developer.gserviceaccount.com](mailto:PROJECT_NUMBER}-compute@developer.gserviceaccount.com)"
```

注意，在这个例子中，我们使用默认的 Google Compute 服务帐户向函数发送通知。为了这个目的，你可能想要使用一个特定的服务帐户，遵循最小特权的原则，我们只是不想因为创建服务帐户和权限分配而使这篇文章变得混乱。

# 后续步骤

一旦你有了在每次构建后运行的代码，你就可以真正发挥创造力了:

*   也许在一个数据库中捕获所有的构建结果并跟踪剥落
*   创建仪表板来查看构建的长期趋势(构建时间增长？)
*   考虑自动归档失败/脆弱版本的 GitHub 问题

如果您在 Google Cloud 社区 Slack ( [#cpp](https://googlecloud-community.slack.com/archives/C8CDDF81H) )中发现这很有用，请告诉我们。