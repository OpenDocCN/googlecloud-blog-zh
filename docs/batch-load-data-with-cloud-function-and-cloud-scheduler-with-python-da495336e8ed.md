# 使用谷歌云功能和云调度程序批量加载数据

> 原文：<https://medium.com/google-cloud/batch-load-data-with-cloud-function-and-cloud-scheduler-with-python-da495336e8ed?source=collection_archive---------0----------------------->

有没有尝试过从公共数据源向您的云存储重复获取数据？我相信你有。在 GCP 有一个简单的方法:

1.  编写一个云函数来获取数据并上传到 GCS bucket(我们将使用 Python 来完成)
2.  配置云调度程序作业来触发此云功能
3.  将调度程序设置为在您希望的时间运行，瞧，您的云桶中有数据了！