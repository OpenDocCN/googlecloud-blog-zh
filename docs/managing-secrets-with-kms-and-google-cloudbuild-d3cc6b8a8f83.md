# 用 KMS 和谷歌云构建管理秘密

> 原文：<https://medium.com/google-cloud/managing-secrets-with-kms-and-google-cloudbuild-d3cc6b8a8f83?source=collection_archive---------1----------------------->

![](img/4284362d6d6403d8e56c2daf57c7465a.png)

来源:谷歌云

本文旨在帮助开发人员在将应用程序部署到 Google Application Engine 时管理他们的秘密，期望如下:

*   Github 存储库(在我们的例子中是 Typescript express 应用程序)
*   可以使用 Docker 在本地构建和运行应用程序
*   谷歌云开发账户

# 设置

*   按照上一篇文章中[的描述设置你的谷歌云账户。](/coinmonks/continuous-integration-with-google-application-engine-and-travis-d822b751fb47)
*   克隆回购

```
git clone [git@github.com](mailto:git@github.com):cipherzzz/ts-cloudbuild-secrets-example.git
```

# 使用谷歌 KMS 进行秘密管理

Google Cloud 有一个密钥管理系统或 KMS 的概念，可以通过 gcloud 作为命令行工具使用，并集成到 cloudbuild 工具中。我们将使用它来加密密码和敏感字段等秘密，并在构建时将解密后的值作为环境变量提供给 docker 容器。

# 钥匙圈/钥匙

我们真的应该只需要一个密匙环，每个实例(集成、测试和生产)都有自己的密匙环。

检查 kms api 是否已启用—[https://console . developers . Google . com/APIs/library/cloud kms . Google APIs . com](https://console.developers.google.com/apis/library/cloudkms.googleapis.com)

```
# If we do not have a keyring
gcloud kms keyrings create vmi-integration-secrets --location global# If we already had a keyring or just created one, take a look at what keys are on it
gcloud kms keys list --location global --keyring vmi-integration-secrets# To add a key - one per application
gcloud kms keys create vertigo-js-node-api --location global --keyring vmi-integration-secrets --purpose encryption# Verify that your keyring has the keys you expect
gcloud kms keys list --location global --keyring vmi-integration-secrets
```

密匙环和密匙将用于在 cloudbuild 过程中加密和解密值。当您的应用程序在云中发生变化时，您可以添加和删除密钥。如果你还有其他问题，这篇文章很棒——[https://cloud.google.com/kms/docs/quickstart](https://cloud.google.com/kms/docs/quickstart)

# 加密秘密

```
# Create a local file with the secret
echo "MyRedisPassword1234" > redis_pw.txt# To encrypt a secret using KMS
gcloud kms encrypt \
  --plaintext-file=redis_pw.txt \
  --ciphertext-file=redis_pw.enc.txt \
  --location=global \
  --keyring=vmi-integration-secrets \
  --key=vertigo-js-node-api# Encode the binary encoded secret as base64 string
base64 redis_pw.enc.txt -w 0 > redis_pw.enc.64.txt
```

您将使用上一步中获得的字符串放入您的 cloudbuild 文件，如下一节所述。

# 解密秘密

我们最终想要的是一种方法来解密我们的编码秘密的 base64 值，并将其作为 env 变量注入到我们的应用程序中。有多种方法可以做到这一点，但使用 cloudbuild 只有一种方法。

# 云构建

为了让 cloudbuild 解密我们的值，它必须是 base64 编码的，并表示为如下秘密(为了清楚起见，我包括了整个 cloudbuild):

```
steps: # Building image
# Note: You need a shell to resolve environment variables with $$
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: 'bash'
  args: [ 
          '-c',
          'docker build -t gcr.io/$PROJECT_ID/appengine/ts-cloudbuild-secrets-example:latest -f Dockerfile --build-arg REDIS_PASS=$$REDIS_PW .' 
        ]
  secretEnv: ['REDIS_PW'] # Push Images       
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/appengine/ts-cloudbuild-secrets-example:latest'] # Deploy to GAE
- name: 'gcr.io/cloud-builders/gcloud'
  args: 
  - 'app'
  - 'deploy'
  - 'app.yaml'
  - '--image-url'
  - 'gcr.io/$PROJECT_ID/appengine/ts-cloudbuild-secrets-example:latest' secrets:
- kmsKeyName: projects/vmi-integration/locations/global/keyRings/vmi-integration-secrets/cryptoKeys/vertigo-js-node-api
  secretEnv:
    REDIS_PW: CiQAkmpYKP7L1ELHIrdvp/J43k1w6EN/l4wgVZnBMMhbEr/dFxYSPQBMN3wJgwxNRTNmNpaif4rSOSHKy7gHTamaxsxo3la2qCLJfVSHz8jUA4jERssiMZAeKhHvfp5LBTDvjxk=
```

注意底部的`secrets:`部分。这为我们的 cloudbuild 提供了要解密的秘密、用于解密的密钥以及我们可以用来引用它的 env 变量。

在 cloudbuild 中回到 docker build 指令。注意我们是如何声明`secretEnv`属性以及期望的`REDIS_PW`键的。这告诉 cloudbuild，我们希望将解密的值传递到这个构建步骤中，而不是加密的 base64 值。还要注意，为了使用`$$REDIS_PW`语法，我们必须在`bash` shell 中手动运行命令。

# Dockerfile 文件

我们将`--build-arg REDIS_PASS=$$REDIS_PW`从我们的 cloudbuild 传入我们的 docker build 指令。我们需要告诉我们的`Dockerfile`这个参数，以便它接受它的值。

```
FROM node:8 as native-build
COPY . .
RUN npm install
RUN npm run buildFROM node:carbon-alpineARG REDIS_PASS
ENV REDIS_PW=${REDIS_PASS}WORKDIR /home/node/app
COPY --from=native-build /dist dist/
COPY --from=native-build /package.json .
COPY --from=native-build /node_modules node_modules/EXPOSE 8080USER node
CMD ["npm", "start"]
```

注意`Dockerfile`中的`ARG REDIS_PASS`——这使我们能够处理 cloudbuild 给出的值。还要注意`ENV REDIS_PW=${REDIS_PASS}`——这将设置`REDIS_PW` env 值，该值将被放入 docker 容器中。每当我们运行构建的映像时，`REDIS_PW` env 变量就会出现并被填充。

# 以打字打的文件

我们通过引用`process.env.REDIS_PW`变量来访问应用程序中的解密变量。请注意，在这种情况下，我们只是检查值，但是您也可以轻松地连接到带有密码的服务器。参见`src/logic.ts`

```
export function checkSecret(): Payload {
    let message = process.env.REDIS_PW==='MyRedisPassword1234'?"Secret Correct":"Secret Wrong";
    return { message }
}
```

# 部署

遵循 [cloudbuild-local 安装说明](https://cloud.google.com/cloud-build/docs/build-debug-locally)以让您的 gcloud sdk、docker 和本地构建环境工作。

从项目的根目录运行以下代码，`cloudbuild-local`工具将执行 cloudbuild 并部署应用程序。

```
cloud-build-local --config=cloudbuild.yaml --dryrun=false .
```

访问`https://<service-url>/secret`查看解密是否成功。您应该会看到类似这样的内容- `{"message":"Secret Correct"}`