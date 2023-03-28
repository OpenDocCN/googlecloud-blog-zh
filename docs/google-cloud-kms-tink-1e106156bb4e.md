# 谷歌云 KMS &叮当

> 原文：<https://medium.com/google-cloud/google-cloud-kms-tink-1e106156bb4e?source=collection_archive---------0----------------------->

## 谷歌 OOS 聚宝盆历险记

披露:我是一名谷歌员工。

对我来说，偶然发现谷歌 OSS 珠宝并不罕见。通常，我会使用一个 Google OSS 项目，而它会使用另一个我不熟悉的 Google OSS 项目。我想知道人们是如何发现这些有用的项目的。

给使用其他[Google] OSS 项目的[Google] OSS 开发者的建议是:宣传里面的东西，以帮助宣传你的依赖性。

所以，Tink…是 SDK。我们的目标是通过一个简单的、经过安全审查的 SDK，让开发人员的生活更轻松，从而更好地开发使用加密的代码。它还集成了谷歌云 KMS(用于加密|解密检查签名是否也在诉讼表上)。

## 准备

设置您的环境:

```
WORKDIR=[[YOUR-WORKDIR]]GOPATH=${WORKDIR}/go
mkdir -p ${GOPATH} && cd ${WORKDIR}PATH=${GOPATH}/bin:${PATH}
```

定义变量:

```
PROJECT=[[YOUR-PROJECT]]
LOCATION=[[YOUR-LOCATION]] # Perhaps "us-west1"
KEYRING=[[YOUR-KEYRING]]   # Perhaps "keyring-01"
```

启用云 KMS

```
gcloud services enable cloudkms.googleapis.com \
--project=${PROJECT}gcloud kms keyrings create ${KEYRING} \
--location=${LOCATION} \
--project=${PROJECT}
```

## 叮当云 KMS

下面是一个使用 Google 的 Golang " [信封加密](https://github.com/google/tink/blob/master/docs/GOLANG-HOWTO.md#envelope-encryption)"示例的简单演示。该脚本创建一个对称加密密钥(`${KEY}`)，并将其添加到您之前创建的密匙环(`${KEYRING}`)。然后，它创建一个服务帐户(`${ROBOT}`)和一个服务帐户密钥(存储在`${FILE}`)。一个`cloudkms.cryptoKeyEncrypterDecrypter`的角色被添加到云 KMS 键(不是添加到项目，只是云 KMS 键！):

```
KEY=[[YOUR-KEY]]              # Perhaps "tink-cloudkms"
ROBOT=[[YOUR-SERVICE-ACCOUNT] # Perhaps "tink-cloudkms"
FILE=${WORKDIR}/${ROBOT}.jsongcloud kms keys create ${KEY} \
--purpose=encryption \
--keyring=${KEYRING} \
--location=${LOCATION} \
--project=${PROJECT}gcloud iam service-accounts create $ROBOT \
--display-name=$ROBOT \
--project=$PROJECTgcloud iam service-accounts keys create ${FILE} \
--iam-account=${ROBOT}@${PROJECT}.iam.gserviceaccount.com \
--project=$PROJECT**gcloud kms keys add-iam-policy-binding** ${KEY} \
--location=${LOCATION} \
--keyring=${KEYRING} \
--member=serviceAccount:${ROBOT}@${PROJECT}.iam.gserviceaccount.com \
--role=roles/**cloudkms.cryptoKeyEncrypterDecrypter** \
--project=$PROJECT
```

这个 IAM 角色足以让我们的代码获得密钥，加密消息，然后使用相同的密钥(对称)解码消息。强有力的是，即使是这个有限的角色也只允许在这个键上使用(`${KEY}`)。以下是来自谷歌 Tink 文档的 Golang 大部分内容:

您可以运行代码:

```
GOOGLE_APPLICATION_CREDENTIALS=${FILE} \
go run main.go \
--project=${PROJECT} \
--location=${LOCATION} \
--keyring=${KEYRING} \
--key=${KEY}
```

> N **B** 我们利用应用程序默认凭证通过环境将服务帐户密钥传递给代码(`GOOGLE_APPLICATION_CREDENTIALS`)。

您应该得到类似于以下内容的内容:

```
Cipher text: 
ASb80OCm...uGGT09qA==
Plain text: manifest
```

## 纯云 KMS

我的意图是使用 Tink 进行签名，但是——IIUC——SDK 目前不支持使用云 KMS 进行签名。因此，相反，我使用谷歌的云 KMS 样本来创建并验证签名。

这一次，我们需要在密钥创建过程中提供更多的细节，为了简化工作，我们将使用单个服务帐户对消息进行签名和验证。实际上，签名者和验证者可能是不同的实体。

```
KEY=[[YOUR-KEY]]              # Perhaps "pure-cloudkms"
ROBOT=[[YOUR-SERVICE-ACCOUNT] # Perhaps "pure-cloudkms"
FILE=${WORKDIR}/${ROBOT}.json**PURPOSE=asymmetric-signing
ALGORITHM=EC_SIGN_P256_SHA256
PROTECTION=software**gcloud alpha kms keys create ${KEY} \
**--purpose=${PURPOSE}** \
**--default-algorithm=${ALGORITHM}** \
**--protection-level=${PROTECTION}** \
--keyring=${KEYRING} \
--location=${LOCATION} \
--project=${PROJECT}gcloud iam service-accounts create $ROBOT \
--display-name=$ROBOT \
--project=$PROJECTgcloud iam service-accounts keys create ${FILE} \
--iam-account=${ROBOT}@${PROJECT}.iam.gserviceaccount.com \
--project=$PROJECTgcloud kms keys add-iam-policy-binding ${KEY} \
--location=${LOCATION} \
--keyring=${KEYRING} \
--member=serviceAccount:${ROBOT}@${PROJECT}.iam.gserviceaccount.com \
--role=**roles/cloudkms.signer** \
--project=${PROJECT}gcloud kms keys add-iam-policy-binding ${KEY} \
--location=${LOCATION} \
--keyring=${KEYRING} \
--member=serviceAccount:${ROBOT}@${PROJECT}.iam.gserviceaccount.com \
--role=**roles/cloudkms.signerVerifier** \
--project=${PROJECT}
```

下面是 Golang，大部分原样来自谷歌的云 KMS 文档:

该代码通过首先对消息进行哈希运算，然后使用签名者的私有(！idspnonenote)来创建签名。)密钥进行加密(！)它。验证者访问签名者的公共(！)密钥并使用它来解密(！idspninfopath _ NV)。)的签名。它将结果与哈希(共享)消息进行比较，以确认只有能够访问私钥的人才能创建该消息。

## 结论

如果 Tink 团队为我提供了如何使用云 KMS 签名的指导，我会更新这篇帖子。在我写纯云 KMS 客户端的时候，我意识到我真的应该把签名者和验证者分成两个服务帐户，这样更现实。

希望这篇短文能鼓励你在将来的工作中考虑 Tink，包括加密库。Tink 也支持 AWS 的密钥管理解决方案，一个开发者已经提出为 Azure 的服务编写集成。

仅此而已！