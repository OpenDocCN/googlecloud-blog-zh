# 拉&推给别人的上游 GitHub 公关

> 原文：<https://medium.com/google-cloud/pull-push-to-someone-elses-upstream-github-pr-6073ae5005e7?source=collection_archive---------3----------------------->

另一个简单的“如何”给那些需要拉和推别人的上游 GitHub 拉请求(PR)的人。

我正在处理一个队友对海洋项目的 GitHub pull 请求(这是我目前关注的焦点)，以帮助完成它。因为她的拉取请求来自她的个人账户，所以我需要一些关键的命令来直接拉取和推送。

# 拉现有的 PR

从上游项目中提取现有 PR。

```
git fetch [*NAME REMOTE PROJECT*] pull/[*PR NUMBER*]/head:[*PR BRANCH NAME*]Example:
git fetch upstream pull/10/head:pr_change
```

该命令包括上例中的拉请求编号 *10* 和上例中的拉请求名称 *pr_change。*

该命令假设项目的远程 URL 名为 *upstream* 。如果远程 URL 分配了不同的名称，则使用该名称。

这将在本地创建一个名为 *pr_change* 的分支，一旦拉操作完成，您将被置于该分支中。当你想把它推回去的时候，对这个树枝做些改变，使它保持干净，对你自己来说更容易。

# 推送到现有 PR

在上游项目中将变更/提交推回给其他人的 PR。

```
git push git@github.com:[*THE SOMEONE ELSE USER ID*]/[*PROJECT NAME*].git [*PR BRANCH NAME*]:[*LOCAL BRANCH NAME*]Example:
git push git@github.com:someoneelse/main_project_name.git pr_change:pr_change
```

在命令中， *someoneelse* 是您放置 PR 来源的用户帐户的名称的地方。在我正在做的项目中，我队友的 GitHub 句柄是杏仁核，我用杏仁核替换了其他人的*。*

另外，将 GitHub 项目名 *main_project_name.git* 替换为您正在处理的项目。在我的项目中，名称是 project-OCEAN.git，这就是我使用的名称。

请确保包括分支机构的名称，这是在公关以及在您的电脑上。在本地检查分支名称并输入。理想情况下，您使用的是与远程服务器相同的分支名称，只需要重复一遍并在中间加上一个:。

## 准备中的其他步骤

在推送之前，我通过提交我的更改来确保我的内容与主分支同步，然后…

正在获取上游主服务器上的最新…

```
git fetch upstream
```

和我当地的分公司合并。

```
git merge upstream/pr_change
```

然后，就可以使用本节开头的命令了。

# 总结性的新闻报导

这里有一个关于如何在 GitHub 项目上拉/推别人 PR 的小故事。就是这样。向前迈进，在你的项目中扰乱其他人的公关。