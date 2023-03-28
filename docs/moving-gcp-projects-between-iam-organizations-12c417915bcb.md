# 在 IAM 组织之间移动 GCP 项目

> 原文：<https://medium.com/google-cloud/moving-gcp-projects-between-iam-organizations-12c417915bcb?source=collection_archive---------0----------------------->

一旦一个 GCP 项目已经[与一个 IAM 组织](https://cloud.google.com/resource-manager/docs/migrating-projects-billing)关联，就没有办法将项目移出组织*(自己)*。然而，这可以在[谷歌企业支持](https://enterprise.google.com/supportcenter)的帮助下完成。

**流程**

1.  创建要移动的项目列表。
2.  将所有项目移出当前组织中的任何文件夹，并移至顶层。
3.  [联系支持人员](https://enterprise.google.com/supportcenter)并提供您希望从当前组织转移到另一个组织的项目列表。
4.  支持人员会将项目移出当前组织，因此它们没有父级(没有组织)。
5.  将所有项目移入新组织。

**示例**

创建项目列表:

```
bash$ **cat > projects.txt << EOF
project-1
project-2
project-3
EOF** 
```

获取当前组织的组织 ID(例如 mydomain.com):

```
bash$ **gcloud organizations list | grep '^mydomain.com'**
mydomain.com        123456789012  C00123456
```

将项目移动到组织的顶层:

```
bash$ **for project in `cat projects.txt`; do
gcloud alpha projects move $project --organization 123456789012
done** 
```

[联系谷歌支持](https://enterprise.google.com/supportcenter)关于将你的项目移出组织。

一旦 Google support 确认您的项目已经被移出组织，那么您就可以[将它们移入新的组织](https://cloud.google.com/resource-manager/docs/migrating-projects-billing)。

获取新组织的组织 ID(例如 mynewdomain.com):

```
bash$ **gcloud organizations list | grep '^mynewdomain.com'**
mynewdomain.com        987654321098  C00987654
```

将项目移入新组织:

```
bash$ **for project in `cat projects.txt`; do
gcloud alpha projects move $project --organization 987654321098
done**
```

现在，`projects.txt`中的所有项目都将在新的组织中！