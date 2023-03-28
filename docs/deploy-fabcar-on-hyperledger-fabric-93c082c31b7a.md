# 在 Hyperledger 结构上部署“fabcar”

> 原文：<https://medium.com/google-cloud/deploy-fabcar-on-hyperledger-fabric-93c082c31b7a?source=collection_archive---------1----------------------->

## 区块链历险记

我将在 11 月花时间提高我对区块链技术的了解(并为 11 月的[日](https://mobro.co/dazwilkin)留胡子)。我从舒适区开始，将 Hyperledger Fabric 部署到谷歌云平台(GCP)虚拟机，并通过“fabcar”[示例](http://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html)应用程序进行旋转。

部署 fabcar 和运行 fabcar 非常简单，所以这篇文章主要是作为一篇基础文章，如果你想在 GCP 上运行 fabcar，你可能会感兴趣。

## 超分类帐结构

我在 GCP 的 Ubuntu 16.04 虚拟机上运行一切。如果你没有 GCP 帐户，你可以在这里免费开始使用。我假设您使用安装了 Google 的 Cloud SDK 的 Linux 机器来创建 VM:

```
BILLING=[[YOUR-BILLING-ID]]
PROJECT=[[YOUR-PROJECT-ID]]
ZONE=[[YOUR-PREFERRED-ZONE]] // us-west1-cINSTANCE=hyperledger-fabric-01gcloud alpha projects create $PROJECTgcloud alpha billing projects link $PROJECT --account-id=$BILLINGgcloud services enable compute.googleapis.com --project=$PROJECTgcloud compute instances create ${INSTANCE} \
--project=${PROJECT} \
--zone=${ZONE} \
--machine-type=custom-2-8192 \
--image=ubuntu-1604-xenial-v20171011 \
--image-project=ubuntu-os-cloud \
--boot-disk-size=50 \
--boot-disk-type=pd-standard \
--boot-disk-device-name=${INSTANCE}gcloud compute ssh ${INSTANCE} \
--zone=${ZONE} \
--project=${PROJECT}
```

从“hyperledger-fabric-01”上的 ssh 会话中，我们需要安装多个先决条件组件。

```
[https://hyperledger-fabric.readthedocs.io/en/release/prereqs.html](https://hyperledger-fabric.readthedocs.io/en/release/prereqs.html)
```

开始了。

**Docker-CE**

```
sudo apt-get updatesudo apt-get install \
    linux-image-extra-$(uname -r) \
    linux-image-extra-virtualsudo apt-get updatesudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-commoncurl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo apt-key add -sudo add-apt-repository \
   "deb [arch=amd64] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
   $(lsb_release -cs) \
   stable"sudo apt-get update
sudo apt-get install docker-ce
sudo usermod -aG docker $USER
sudo systemctl enable dockerdocker --version
```

**docker-compose**

```
sudo curl -L [https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname](https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname) -s`-`uname -m` -o /usr/local/bin/docker-composesudo chmod +x /usr/local/bin/docker-composedocker-compose --version
```

**戈朗**

```
VERSION=1.9.1
OS=linux
ARCH=amd64
sudo curl \
--location [https://golang.org/dl/go$VERSION.$OS-$ARCH.tar.gz](https://golang.org/dl/go$VERSION.$OS-$ARCH.tar.gz) \
--output go$VERSION.$OS-$ARCH.tar.gzsudo tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gzexport PATH=$PATH:/usr/local/go/bingo version
```

**Node.js**

```
curl -sL [https://deb.nodesource.com/setup_6.x](https://deb.nodesource.com/setup_6.x) | sudo -E bash -
sudo apt-get install -y nodejsnodejs --version
v6.11.5npm --version
3.10.10
```

**降级 Python(在 Ubuntu 上)**

```
sudo apt-get install pythonpython --version
Python 2.7.12
```

## 面料样品:fabcar

```
git clone https://github.com/hyperledger/fabric-samples.gitcd fabric-samples/fabcar
```

技巧:Hyperledger 在 dockerhub 上托管结构容器。因为这些是为各种运行时架构构建的，所以没有单一的“最新”容器。诀窍是识别和下载容器的最新版本，它们在方便的锁步中被版本化，然后用“最新”标记它们。以下是 Linux X86_64 的脚本:

```
TAG=x86_64-1.0.3
for IMAGE in orderer couchdb peer ca tools
do
  docker pull hyperledger/fabric-${IMAGE}:${TAG} && \
  docker tag hyperledger/fabric-${IMAGE}:${TAG} hyperledger/fabric-${IMAGE}:latest
done
```

完成后:

```
docker images \
--format="{{ .Tag }}\t{{ .Repository }}" \
| grep hyperledger \
| sortlatest  hyperledger/fabric-ca
latest  hyperledger/fabric-couchdb
latest  hyperledger/fabric-orderer
latest  hyperledger/fabric-peer
latest  hyperledger/fabric-toolsx86_64-1.0.3    hyperledger/fabric-ca
x86_64-1.0.3    hyperledger/fabric-ccenv
x86_64-1.0.3    hyperledger/fabric-couchdb
x86_64-1.0.3    hyperledger/fabric-orderer
x86_64-1.0.3    hyperledger/fabric-peer
x86_64-1.0.3    hyperledger/fabric-tools
```

然后运行以下命令来运行结构测试集群:

```
./startFabric.sh
```

您应该会看到一串日志结束于:

```
2017-11-04 00:03:42.880 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 00a Chaincode invoke successful. result: status:200 
2017-11-04 00:03:42.880 UTC [main] main -> INFO 00b Exiting.....Total execution time : 33 secs ...
```

而且，您应该能够:

```
docker ps --format="{{ .ID }}\t{{ .Names }}"bdcad9115060    dev-peer0.org1.example.com-fabcar-1.0
e3a219fd5c50    cli
160318483edd    peer0.org1.example.com
5caa3f261c05    couchdb
f1b93255fa5d    ca.example.com
385d784e0581    orderer.example.com
```

好吧！还有一个先决条件步骤:

```
sudo apt-get install build-essentialnpm install
```

现在，您应该能够:

```
node query.jsCreate a client and set the wallet location
Set wallet path, and associate user  PeerAdmin  with application
Check user is enrolled, and set a query URL in the network
Make query
Assigning transaction_id:  2b7d262eca16c68523fe61331f08114110f6c6b4d6f1d4725063fcb00d8da4d7
returned from query
Query result count =  1
Response is  [
{"Key":"CAR0", "Record":{"colour":"blue","make":"Toyota","model":"Prius","owner":"Tomoko"}},{"Key":"CAR1", "Record":{"colour":"red","make":"Ford","model":"Mustang","owner":"Brad"}},{"Key":"CAR2", "Record":{"colour":"green","make":"Hyundai","model":"Tucson","owner":"Jin Soo"}},
{"Key":"CAR3", "Record":{"colour":"yellow","make":"Volkswagen","model":"Passat","owner":"Max"}},{"Key":"CAR4", "Record":{"colour":"black","make":"Tesla","model":"S","owner":"Adriana"}},{"Key":"CAR5", "Record":{"colour":"purple","make":"Peugeot","model":"205","owner":"Michel"}},{"Key":"CAR6", "Record":{"colour":"white","make":"Chery","model":"S22L","owner":"Aarav"}},{"Key":"CAR7", "Record":{"colour":"violet","make":"Fiat","model":"Punto","owner":"Pari"}},{"Key":"CAR8", "Record":{"colour":"indigo","make":"Tata","model":"Nano","owner":"Valeria"}},{"Key":"CAR9", "Record":{"colour":"brown","make":"Holden","model":"Barina","owner":"Shotaro"}}
]
```

我不会在这里重复织物教程，而是:

```
[https://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html](https://hyperledger-fabric.readthedocs.io/en/latest/write_first_app.html)
```

## 结论

先不说先决条件，让 Fabric 测试集群运行并使用“fabcar”样本是很简单的。Fabric 是用 Golang 编写的，支持 Golang 和 Java for contracts，在本例中，还支持 Node.js 作为客户端。

## 拆毁

要关闭结构群集，请执行以下操作:

```
cd ../basic-network
docker-compose --file ./docker-compose.yml downStopping cli                    ... done
Stopping peer0.org1.example.com ... done
Stopping orderer.example.com    ... done
Stopping couchdb                ... done
Stopping ca.example.com         ... done
Removing cli                    ... done
Removing peer0.org1.example.com ... done
Removing orderer.example.com    ... done
Removing couchdb                ... done
Removing ca.example.com         ... done
Removing network net_basic
```

您可以从所有 ssh 会话中“退出”。

您应该会返回到您的主机会话。就是你运行 gcloud 命令的那个。您可以停止 hyperledger-fabric-01 虚拟机:

```
gcloud compute instances stop ${INSTANCE} \
--zone=${ZONE} \
--project=${PROJECT}
```

或者(永久)删除它:

```
gcloud compute instances delete $(INSTANCE} \
--zone=${ZONE} \
--project=${PROJECT}
--quiet
```

或者干脆删除整个 GCP 项目:

```
gcloud alpha projects delete ${PROJECT} --quiet
```

就是这样！