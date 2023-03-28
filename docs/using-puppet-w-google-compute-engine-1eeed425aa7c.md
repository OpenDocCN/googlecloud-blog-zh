# 使用谷歌计算引擎木偶

> 原文：<https://medium.com/google-cloud/using-puppet-w-google-compute-engine-1eeed425aa7c?source=collection_archive---------4----------------------->

在本文中，您将了解如何使用 Puppet 在 Google 云平台上的 Google 计算引擎中提供虚拟机实例。

## 先决条件

1.  一个[谷歌云平台](http://cloud.google.com)账号(带项目)
2.  [谷歌云 SDK](https://cloud.google.com/sdk/) 安装完毕
3.  通过“gcloud auth 登录”获得授权的 Google Cloud SDK

## 在本地安装木偶

首先，在本地安装 Puppet。我在我的笔记本电脑(OS X)上遵循了[安装说明](https://docs.puppetlabs.com/guides/install_puppet/pre_install.html)。这很简单。

## 安装 gce_compute 模块

gce_compute 模块允许您将计算引擎实例、磁盘、负载平衡器等作为本机 Puppet 资源进行管理。

```
$ puppet module install puppetlabs-gce_compute
```

接下来，创建一个~/。puppet/device.conf 文件(或者，/etc/puppet/device.conf):

```
[gce_project_id]
  type gce
  url [/dev/null]:gce_project_id
```

接下来，创建一个 puppet 清单文件~/www.pp，内容如下:

```
# Learn more about zones here:
# [https://cloud.google.com/compute/docs/zones](https://cloud.google.com/compute/docs/zones)
$zone = 'us-central1-a'# Configure the firewall rule
gce_firewall { 'puppet-www-http':
 ensure => present,
 network => 'default',
 description => 'allows incoming HTTP connections',
 allowed => 'tcp:80',
}# Configure the boot disk
gce_disk { 'puppet-www-boot':
 ensure => present,
 source_image => 'debian-7',
 size_gb => '10',
 zone => "$zone",
}# Configure one instance
gce_instance { 'puppet-www':
 ensure => present,
 description => 'Basic web server',
 machine_type => n1-standard-1,
 disk => 'puppet-www-boot,boot',
 zone => "$zone",
 network => 'default',
 require => Gce_disk['puppet-www-boot'],
 puppet_master => "",
 manifest => '
  class apache ($version = "latest") {
   package {"apache2":
   ensure => $version,
  }
  file {"/var/www/index.html":
   ensure => present,
   content => "<html>\n<body>\n\t<h2>Hi, this is $hostname ($gce_external_ip).</h2>\n</body>\n</html>\n",
   require => Package["apache2"],
  }
  service {"apache2":
   ensure => running,
   enable => true,
   require => File["/var/www/index.html"],
  }
 }
 include apache',
}
```

应用此清单:

```
$ puppet apply --certname gce_project_id www.pp
```

查找 IP 地址:

```
$ gcloud compute instances list
NAME                   ZONE          MACHINE_TYPE  INTERNAL_IP    EXTERNAL_IP     STATUS
puppet-www             us-central1-a n1-standard-1 ...   xxx.xxx.xxx.xxx RUNNING
```

最后，在您的浏览器中查看一下[http://xxx.xxx.xxx.xxx/](http://the_external_ip/)=)，您应该会看到一个简单的网页，其内容是在 [www.pp](http://www.pp) 文件中指定的。

## 寻找更高级的东西？

在[https://github . com/Google cloud platform/compute-video-demo-Puppet](https://github.com/GoogleCloudPlatform/compute-video-demo-puppet)查看完整的演示，其中包含设置 Puppet 服务器并旋转四个 Puppet 托管节点的源代码！

## 有用的资源

[使用 Puppet、Chef、Salt 和 Ansible 进行计算引擎管理](https://cloud.google.com/developers/articles/google-compute-engine-management-puppet-chef-salt-ansible/) —关于如何使用不同的工具、利弊等保持实例最新的详细信息。[附录](https://cloud.google.com/developers/articles/google-compute-engine-management-puppet-chef-salt-ansible-appendix/)有实际的步骤和我使用的样本清单文件。

[使用 Puppet 自动化 Google 计算引擎](http://googlecloudplatform.blogspot.com/2014/04/using-puppet-to-automate-google-compute-engine.html)——使用 gce_compute 和样本清单文件的好例子。

[gce_compute](https://forge.puppetlabs.com/puppetlabs/gce_compute) —傀儡锻造。关于 gce_compute 你想知道的一切非常详细的信息！

[云 _ 供给者](https://forge.puppetlabs.com/puppetlabs/cloud_provisioner) —傀儡锻造。这里没有太多的信息。请参考[安装文件](https://docs.puppetlabs.com/pe/latest/cloudprovisioner_configuring.html#installing)。