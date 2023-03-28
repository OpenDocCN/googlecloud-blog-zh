# Google Cloud 上的 Gitlab 和工作负载身份联盟

> 原文：<https://medium.com/google-cloud/gitlab-and-workload-identity-federation-on-google-cloud-a0795091e404?source=collection_archive---------1----------------------->

您可以轻松地使用 Workload Identity Federation 从 Gitlab CI 管道中安全地使用 Google Cloud APIs，例如将 Docker 容器映像推送到工件注册中心。

![](img/a256eb5e3ef515b8f4b9fd4c94dd9c90.png)

照片由 [Denise Jans](https://unsplash.com/@dmjdenise?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 在 [Unsplash](https://unsplash.com/s/photos/swiss-army-knife?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 上拍摄

您需要启用 IAM APIs。首先为 Gitlab 创建一个工作负载身份池。

```
gcloud iam workload-identity-pools create gitlab \
    --location="global" \
    --display-name="Gitlab"
```

接下来，我们需要为池配置 Gitlab 提供程序。

```
gcloud iam workload-identity-pools providers create-oidc gitlab \
    --location="global" \
    --workload-identity-pool="gitlab" \
    --issuer-uri="https://gitlab.com" \
    --allowed-audiences="[AUDIENCE](https://gitlab.com)" \
    --attribute-mapping="google.subject=assertion.sub"
```

然后，我们创建一个服务帐户，并授予 Gitlab repo 对该服务帐户的访问权限。您需要替换`PROJECT_NUMBER`和 Gitlab 项目路径，其形式为`project_path:GROUP/REPO:ref_type:branch:ref:main`

```
gcloud iam service-accounts create gitlab-pipelinegcloud iam service-accounts add-iam-policy-binding SERVICE_ACCOUNT_EMAIL \
    --role=ROLE_ID  --member=principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/gitlab/subject/GITLAB_PROJECT_PATH
```

在您的管道中，您可以使用凭证文件针对 API 进行身份验证，例如许多客户端库、gcloud CLI 或 Terraform 都支持这一点。确保用正确的值替换`PROJECT_NUMBER`、`POOL_ID`、`PROVIDER_ID`和`SERVICE_ACCOUNT_EMAIL`。

```
stages: 
  - verify
example: 
  image: "google/cloud-sdk:slim"
  script: 
    - |
        echo "$CI_JOB_JWT_V2" > token.txt
        gcloud iam workload-identity-pools create-cred-config \
        projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID \
        --service-account=SERVICE_ACCOUNT_EMAIL \
        --service-account-token-lifetime-seconds=600 \
        --output-file=$CI_PROJECT_DIR/credentials.json \
        --credential-source-file=token.txt
    - "export GOOGLE_APPLICATION_CREDENTIALS=$CI_PROJECT_DIR/credentials.json"
    - "gcloud auth login --cred-file=$CI_PROJECT_DIR/credentials.json"
    - "gcloud auth print-access-token"
  stage: verify
```

仅此而已。不再需要导出服务帐户密钥，不再需要摆弄 cURL 来手动调用 IAM APIs。你所需要的只是 gcloud SDK 与谷歌云 API 交互的瑞士军刀。

感谢 [Ludovico](/@ludomagno) 简化了令牌加载！