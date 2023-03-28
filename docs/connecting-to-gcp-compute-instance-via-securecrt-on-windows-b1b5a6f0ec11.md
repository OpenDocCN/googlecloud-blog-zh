# 通过 Windows 上的 SecureCRT 连接到 GCP 计算实例

> 原文：<https://medium.com/google-cloud/connecting-to-gcp-compute-instance-via-securecrt-on-windows-b1b5a6f0ec11?source=collection_archive---------1----------------------->

如果你曾经想通过 SecureCRT 或你选择的其他工具连接到你的 Google Cloud Provider compute 实例，假设你的机器有一个外部 IP 设置，这是一种方法。

您首先需要自己的 SSH 密钥，它由一个惟一的私有 SSH 密钥文件和一个匹配的公共 SSH 密钥文件组成。我已经使用 Git Bash 生成了一个，因为我已经在本地安装了 Git。

```
**$ ssh-keygen -t rsa -C "**[**your_email@youremail.com**](mailto:your_email@youremail.com)**"**
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/you/.ssh/id_rsa): [Press Enter]
Enter passphrase (empty for no passphrase): [Type a passphrase]
Enter same passphrase again: [Type passphrase again]
```

SSH 公钥应该在`c:\Users\[your username]\.ssh\id_rsa.pub`中可用。公钥将采用以下格式:

```
<protocol> <key-blob> <your_email@youremail.com>
```

现在，转到您的 GCP 控制台，找到您的实例，然后单击 Edit。转到 SSH 密钥，单击添加。如果你粘贴了`id_rsa.pub`的内容，那么 GCP 将尝试解析来自*your_email@yourmail.com*(*your _ email*)的 SSH 用户名，该用户名可能与你在实例中使用的用户名不对应。

在这种情况下，最好显式指定用户名，并将以下内容粘贴到 SSH 框中:

```
<protocol> <key-blob> google-ssh {"userName":"<[username@example.com](mailto:username@example.com)>","expireOn":"<date>"}
```

与`id_rsa.pub`的内容相比，唯一的不同是你将你的电子邮件替换为:

```
google-ssh {"userName":"<[username](mailto:username@example.com)>","expireOn":"<date>"}
```

`<date>`可以是类似`2020-01-01`的东西。`<username>`是您在计算实例中的用户名。