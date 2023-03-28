# 使用 Codecov 和 kcov & Travis CI 的 Bash 代码覆盖率

> 原文：<https://medium.com/google-cloud/bash-code-coverage-using-codecov-with-kcov-travis-ci-303f3356981c?source=collection_archive---------0----------------------->

我在 Travis CI 中的 VM 构建系统和获取 gcloud 的当前版本方面有一些限制，所以我做了一些小改动，并从 [Codecov bash 示例](https://github.com/codecov/example-bash)中转移出来。

这是我目前正在使用的 travis.yml 的一部分:

```
sudo: falseaddons:
  apt:
    packages:
      - binutils-dev
      - libcurl4-openssl-dev
      - libdw-dev
      - libiberty-devlanguage: bashenv:
- PATH=${PATH}:${HOME}/kcov/binbefore_install:
- wget [https://github.com/SimonKagstrom/kcov/archive/master.tar.gz](https://github.com/SimonKagstrom/kcov/archive/master.tar.gz)install:
- tar xzf master.tar.gz
- cd kcov-master
- mkdir build
- cd build
- cmake -DCMAKE_INSTALL_PREFIX=${HOME}/kcov ..
- make
- make install
- cd ../..
- rm -rf kcov-master
- mkdir -p coveragescript:
- kcov coverage test.shafter_success:
- bash <(curl -s [https://codecov.io/bash](https://codecov.io/bash))
```

这对我来说没有任何问题。它摆脱了 VM Travis CI 构建系统，让我可以灵活地安装 gcloud，而且速度也更快！对[在基于容器的环境](https://docs.travis-ci.com/user/installing-dependencies/#Installing-Packages-on-Container-Based-Infrastructure)上安装包的支持使这成为可能，允许我满足对 [kcov](https://github.com/SimonKagstrom/kcov/blob/master/INSTALL.md) 的依赖。

我还用一种超级简单的秒表方式测试了 Travis CI 中 kcov 的缓存..缓存测试运行大约需要 1.12 分钟，而下载和编译每个构建大约需要 2:40 分钟。

下面是一个更完整的 travis.yml 示例

```
sudo: falseaddons:
  apt:
    packages:
      - binutils-dev
      - libcurl4-openssl-dev
      - libdw-dev
      - libiberty-devlanguage: bashcache:
  directories:
  - "${HOME}/google-cloud-sdk/"
  - "${HOME}/kcov/"env:
- PATH=$PATH:${HOME}/google-cloud-sdk/bin:${HOME}/kcov/bin CLOUDSDK_CORE_DISABLE_PROMPTS=1before_install:
- openssl aes-256-cbc -K $encrypted_c4d8f070a225_key -iv $encrypted_c4d8f070a225_iv
  -in credentials.tar.gz.enc -out credentials.tar.gz -d
- if [ ! -d "${HOME}/google-cloud-sdk/bin" ]; then rm -rf ${HOME}/google-cloud-sdk;
  curl [https://sdk.cloud.google.com](https://sdk.cloud.google.com) | bash; fi
- if [ ! -d "${HOME}/kcov/bin" ]; then wget [https://github.com/SimonKagstrom/kcov/archive/master.tar.gz](https://github.com/SimonKagstrom/kcov/archive/master.tar.gz);
  tar xzf master.tar.gz;
  cd kcov-master;
  mkdir build;
  cd build;
  cmake -DCMAKE_INSTALL_PREFIX=${HOME}/kcov ..;
  make;
  make install;
  cd ../..;
  rm -rf kcov-master;
  mkdir -p coverage; fi
- tar -xzf credentials.tar.gz
- gcloud auth activate-service-account --key-file client-secret.jsoninstall:
- gcloud -q components updatescript:
#- kcov --exclude-path=ops-common coverage test.shafter_success:
- bash <(curl -s [https://codecov.io/bash](https://codecov.io/bash))notifications:
  slack: lzysh:0nheJvFHhsWdYlVVfdX0mRNu
```

我可能会测试一下这个配置，看看效果如何。如果一切正常，编写一些快速检查 kcov 更新的代码，并在需要时进行更新。

与实现代码覆盖的努力相比，即使是简单的 GCP 基础设施 bash 代码，代码覆盖的附加值也是相当高的，所以我要说去做吧！