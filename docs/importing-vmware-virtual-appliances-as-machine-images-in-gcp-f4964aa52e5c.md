# 在 GCP 将 VMWare 虚拟设备作为机器映像导入

> 原文：<https://medium.com/google-cloud/importing-vmware-virtual-appliances-as-machine-images-in-gcp-f4964aa52e5c?source=collection_archive---------1----------------------->

![](img/af87c659bace17eef9e34b5cfd7fb0f0.png)

*本* ***博客*** *介绍了如何将虚拟机镜像从 VMWare 虚拟设备导入****【GCP】****。在这篇博客中，我们将带您了解使用 cli 从 GCP 的 OVA 和 VMDK 文件导入机器映像的不同步骤。除此之外，我们还将以****VMWare Unified Access Gateway Appliance****为例，向 GCP 展示一个* ***自定义操作系统*** *映像的导入。*

# **先决条件**

以下是继续导入图像所需的先决条件

1.  安装或更新至最新版本的[谷歌云 CLI](https://cloud.google.com/compute/docs/gcloud-compute) 。
2.  [在 *gcloud* CLI 上配置](https://cloud.google.com/compute/docs/gcloud-compute#set_default_zone_and_region_in_your_local_client)一个默认的项目、区域和 zone。
3.  确保 *gcloud* CLI 能够访问 GCS 和计算引擎。

# 从 OVA/OVF 文件导入机器映像

下面给出了从 OVA 文件导入机器映像的步骤。

**第一步:导出环境变量**

导出下面给出的变量，开始使用——

```
$ export network_name="<NETWORK_NAME>"
$ export subnet_name="<SUBNET_NAME>"
$ export machine_image_zone="<ZONE>"
$ export image_name="<MACHINE_IMAGE_NAME>"
$ export gcs_artifact_path="<GCS_PATH_TO_STORE_ARTIFACTS>"
$ export os_type="<OS>"
$ export ova_file_path="<OVA_FILE_PATH>"
```

替换以下内容:

*   `MACHINE_IMAGE_NAME`:您要导入的机器图像的名称。
*   `OS`:OVA 文件的操作系统。指定正在导入的机器映像的操作系统。必须是:`centos-7`、`debian-10`、`debian-11`、`debian-8`、`debian-9`、`opensuse-15`、`rhel-6`、`rhel-6-byol`、`rhel-7`、`rhel-7-byol`、`rhel-8`、`rhel-8-byol`、`rocky-8`、`sles-12`、`sles-12-byol`、`sles-15`、`sles-15-byol`、`sles-sap-12`、`sles-sap-12-byol`、`sles-sap-15`、`sles-sap-15-byol`、`ubuntu-1404`、`ubuntu-1604`、`ubuntu-1804`、`ubuntu-2004`、`ubuntu-2204`
*   `NETWORK_NAME`:要导入图像的网络名称。(可选)
*   `SUBNET_NAME`:要导入图像的子网名称。(可选)
*   `ZONE`:导入机器图像的[区](https://cloud.google.com/compute/docs/regions-zones#locations)。如果留空，则使用项目的默认区域(可选)。
*   `OVA_FILE_PATH`:本地系统中 OVA 的文件路径。
*   `GCS_PATH_TO_STORE_ARTIFACTS`:存储 OVA 文件的云存储路径。

**第二步:上传虚拟磁盘文件到云存储**

使用`gsutil cp`将 OVA 文件推送到云存储。

```
$ gsutil cp $ova_file_path $gcs_artifact_path
$ gcs_file_name=$(basename $ova_file_path)
$ export gs_ova_file_url="$gcs_artifact_path/$gcs_file_name"
```

**第三步:将图像导入 GCP**

使用`cloud compute machine-images import`命令从虚拟设备导入 OVA 格式的机器映像。

```
$ gcloud compute machine-images import $image_name \
    --os=$os_type \
    --source-uri=$gs_ova_file_url \
    --network=$network_name \
    --subnet=$subnet_name \
    --zone=$machine_image_zone
```

> **注意**:如果命令不支持操作系统类型，则`cloud compute machine-images import`不能用于创建机器映像。ex-光子图像
> 
> 有关**g cloud compute machine-images import**命令的更多详细信息，请参考[官方文档](https://cloud.google.com/sdk/gcloud/reference/compute/machine-images/import)。

# 从 VMDK 文件导入机器映像

下面给出了从 VMDK 文件导入机器映像的步骤

**第一步:导出环境变量**

导出以下变量以开始—

```
$ export subnet_name="<SUBNET_NAME>"
$ export machine_image_zone="<ZONE>"
$ export image_name="<MACHINE_IMAGE_NAME>"
$ export gcs_artifact_path="<GCS_PATH_TO_STORE_ARTIFACTS>"
$ export vmdk_file_path="<VMDK_FILE_PATH>"
```

替换以下内容:

*   `MACHINE_IMAGE_NAME`:您要导入的机器图像的名称。
*   `SOURCE_URI`:云存储中 VMDK 文件的路径。
*   `NETWORK_NAME`:要导入图像的网络名称。(可选)
*   `SUBNET_NAME`:要导入图像的子网名称。(可选)
*   `ZONE`:导入机器图像的[区](https://cloud.google.com/compute/docs/regions-zones#locations)。如果留空，则使用项目的默认区域(可选)。
*   `VMDK_FILE_PATH`:VMDK 在您本地系统中的文件路径。
*   `GCS_PATH_TO_STORE_ARTIFACTS`:存放 VMDK 文件的云存储路径。

**第二步:上传虚拟磁盘文件到云存储**

使用`gsutil cp`将 VMDK 文件推送到云存储。

```
$ gsutil cp $vmdk_file_path $gcs_artifact_path
$ export gs_vmdk_file_url="$gcs_artifact_path/$(basename $vmdk_file_path)"
```

**第三步:将图像导入 GCP**

使用`cloud compute images import`命令导入机器图像。

```
$ gcloud compute images import $image_name \
  --source-file $gs_vmdk_file_url \
  --subnet=$subnet_name \
  --zone=$machine_image_zone \
  --no-address
```

> **注** : `cloud compute images import`也可用于将虚拟磁盘映像，如 VMware VHD 文件导入计算引擎。
> 
> 要覆盖检测到的操作系统，请指定`--os`标志。您可以使用`--data-disk`标志省略翻译步骤。有关可选标志的更多信息，请参考[官方文档](https://cloud.google.com/sdk/gcloud/reference/compute/images/import)。

有关操作系统支持，请参见[操作系统详情。](https://cloud.google.com/compute/docs/images/os-details#import)

# 将统一接入网关镜像上传至谷歌云平台

下面给出了在 GCP 从统一访问网关 OVA 文件创建机器映像的步骤。

**步骤 1:下载统一接入网关镜像(OVA 文件)**

要将统一接入网关实例部署到计算引擎，您必须将统一接入网关设备磁盘映像上传到 Google 云平台。

从 [VMware 下载页面](https://my.vmware.com/web/vmware/downloads/#all_products)下载 2103 版或更高版本的 Unified Access Gateway.ova 镜像文件。

**第二步:从 OVA 文件中提取 VMDK 文件**

```
$ mkdir /tmp/images
$ cd /tmp/images
$ tar -xvf $ova_file_name .
```

`ova_file_name` : ova_file_name 是。ova 映像文件，在前面的步骤中从 VMware 下载页面下载。

***例如***:euc-unified-access-gateway-21 . 03 . 0 . 0–42741891 _ ov F10 . ova 从 VMware 下载页面下载，其中`21-03`为版本号，`42741891`为内部版本号。

**第三步:导出环境变量**

导出下面给出的所需变量—

```
$ export network_name="<NETWORK_NAME>"
$ export subnet_name="<SUBNET_NAME>"
$ export machine_image_zone="<ZONE>"
$ export gcs_artifact_path="<GCS_PATH_TO_STORE_ARTIFACTS>"
$ export uag_image_folder="/tmp/images"
$ export vmdk_file_name="euc-unified-access-gateway-21.03.0.0-42741891-system.vmdk"
```

替换以下内容:

*   `MACHINE_IMAGE_NAME`:您要导入的机器图像的名称。
*   `SOURCE_URI`:云存储上你的 OVA 或者 OVF 文件的路径。
*   `NETWORK_NAME`:要导入图像的网络名称。(可选)
*   `SUBNET_NAME`:要导入图像的子网名称。(可选)
*   `ZONE`:导入机器图像的[区](https://cloud.google.com/compute/docs/regions-zones#locations)。如果留空，则使用项目的默认区域(可选)。
*   `OVA_FILE_PATH`:本地系统中 OVA 的文件路径。
*   `GCS_PATH_TO_STORE_ARTIFACTS`:存储 OVA 文件的云存储路径。
*   `vmdk_file_name`:用/tmp/images 文件夹中的 VMDK 文件名替换此变量。

**第四步:上传 VMDK 文件到云存储**

使用`gsutil cp`将 VMDK 文件推送到云存储。

```
$ gsutil cp $uag_image_folder/$vmdk_image_file $gcs_artifact_path
```

**第五步:设置变量**

设置下面给出的变量以创建机器图像—

```
$ export gs_vmkd_file_url="<SOURCE_URI>"
$ export image_name=$(echo $vmdk_file_name | sed 's/-system.vmdk//' | sed 's/\./-/g')
```

`SOURCE_URI`:云存储中 VMDK 文件的路径。

**步骤 6:在计算引擎中创建设备映像**

使用`gcloud compute images import`命令导入机器图像。

```
$ gcloud compute images import $image_name \
  --data-disk --source-file $gs_vmdk_file_url \
  --subnet=$subnet_name \
  --zone=$machine_image_zone \
  --no-address 
```

> **注意**:我们不能使用`gcloud compute machine-images import`命令创建机器镜像，因为统一访问网关是 Photon OS，该命令不支持。
> 
> 因此，我们使用带有`--data-disk`标志的`gcloud compute images import`命令。此标志指定磁盘上没有安装可引导的操作系统。导入磁盘，但不使其可引导或在其上安装 Google tools。

有关可选标志的更多信息，请参考[官方文件](https://cloud.google.com/sdk/gcloud/reference/compute/images/import)。

# 结论

*总之，用户可以参考本博客，从 OVA 和 VMDK 文件创建机器映像。该博客还介绍了如何导入定制操作系统映像，例如 GCP 的 VMWare 统一访问网关映像。*

感谢您的阅读。:)