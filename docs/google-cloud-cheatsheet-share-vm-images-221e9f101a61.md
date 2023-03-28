# 谷歌云备忘单:共享虚拟机映像

> 原文：<https://medium.com/google-cloud/google-cloud-cheatsheet-share-vm-images-221e9f101a61?source=collection_archive---------1----------------------->

共享虚拟机映像是在同一环境中产生大量实例的一种简单方法。更有趣的是，这些虚拟机映像可以跨项目共享。假设该图像是在项目 SRC 中创建的，并将被项目 DEST 使用。以下是你应该做的事情的清单:

*   从实例创建映像或从另一个映像克隆它

```
gcloud compute images create [IMAGE_NAME] \
  --source-image [SOURCE_IMAGE] \
  --source-image-project [IMAGE_PROJECT_SRC] \
  --family [IMAGE_FAMILY] \
  --project [IMAGE_PROJECT_DEST]
```

*   列出项目中可用的图像，并确保您选择的图像确实存在

```
gcloud compute images list --project [IMAGE_PROJECT_SRC]
```

*   在 DEST 项目中创建图像

```
gcloud compute instances create [INSTANCE_NAME]
    --image-family [IMAGE_FAMILY]
    --image-project [IMAGE_PROJECT_SRC]
    --project [IMAGE_PROJECT_DEST]
```