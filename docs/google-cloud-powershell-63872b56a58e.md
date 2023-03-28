# 谷歌云 PowerShell

> 原文：<https://medium.com/google-cloud/google-cloud-powershell-63872b56a58e?source=collection_archive---------1----------------------->

## 快速命中

在谷歌[云壳](https://cloud.google.com/shell/)运行 PowerShell 很容易。

打开云壳:

```
**sudo apt-get install -y powershell**
```

然后:

```
**pwsh**
PowerShell 6.1.3
Copyright (c) Microsoft Corporation. All rights reserved.[https://aka.ms/pscore6-docs](https://aka.ms/pscore6-docs)
Type 'help' to get help.PS /home/[USER]>
```

然后:

`[https://cloud.google.com/tools/powershell/docs/quickstart](https://cloud.google.com/tools/powershell/docs/quickstart)`

并且:

```
**Install-Module GoogleCloud -Scope CurrentUser**
```

> **注意**因为 Cloud Shell 是针对每个用户、每个会话提供的计算引擎实例，所以您需要在会话之间重新运行安装。

但是你会得到:

```
**Get-GceDisk**CreationTimestamp           : 2019-03-22T14:09:37.848-07:00
Description                 :
DiskEncryptionKey           :
Id                          : 8082606374212137470
Kind                        : compute#disk
LabelFingerprint            : 42WmSpB8rSM=
Labels                      :
LastAttachTimestamp         : 2019-03-22T14:09:37.848-07:00
LastDetachTimestamp         :
Licenses                    :
Name                        : instance-1
Options                     :
SelfLink                    : [.../disks/instance-1](https://www.googleapis.com/compute/v1/projects/dazwilkin-190319-docker/zones/us-east1-b/disks/instance-1)
SizeGb                      : 10
SourceImage                 : 
SourceImageEncryptionKey    :
SourceImageId               :
SourceSnapshot              :
SourceSnapshotEncryptionKey :
SourceSnapshotId            :
Status                      : READY
Type                        :
Users                       :
Zone                        :...[/us-east1-b](https://www.googleapis.com/compute/v1/projects/dazwilkin-190319-docker/zones/us-east1-b)
ETag                        :
```

并且:

```
**Get-GceInstance
Get-GceInstance instance-1 -Zone us-east1-b**Name       CpuPlatform   MachineType   Zone       TimeCreated
----       -----------   -----------   ----       -----------
instance-1 Intel Haswell n1-standard-1 us-east1-b 2019-03-22T14:09:37.824-07:00
```

更多信息，请参见:

`[https://googlecloudplatform.github.io/google-cloud-powershell/#/](https://googlecloudplatform.github.io/google-cloud-powershell/#/)`

仅此而已！