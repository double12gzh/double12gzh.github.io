---
layout: snippets
title: logseq 使用
tags: [tool, logseq, github]
description: logseq descripition
keywords: logseq, github, post-commit, pre-commit
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

使用 Github 作为 Logseq 的数据同步

## 1. 日常使用
我在使用 Logseq 作为我的日常信息记录：

有想法就放在 Journal 当中记录，其中支持强大的查询语句，我们可以使用 TAG 来区分不同的内容分类（比如编程、工作等）

除了其提供的基本能力外，其还支持数据同步到 github。它提供了 Git 版本控制的能力，你只需要在设置当中开启 `Git Commit` 的能力，就可以让其自动使用 `Git`` 来添加版本，从而实现将你的变更通过 Git 本身的能力来记录。

## 2. 数据同步
- 打开 Logseq 自带的 Git 功能
- 准备 pre-commit/post-commit
  从代码库 [Logseq-Git-Sync-101](https://github.com/CharlesChiuGit/Logseq-Git-Sync-101) 中，复制上述两个文件到本地代码库的 .git/hooks 中。复制完成后，将会在 `git commit` 前主动执行 `git pull`，可以有效避免冲突，另外，
  在 commit 后会自动执行 `git push` 将数据上传到 github
- 添加可执行权限
  ```bash
  chmod a+x post-commit pre-commit
  ```

## 3. 其它
除了上述的配置可以实现同步外，我们还可以添加插件 `logseq-plugin-git` 实现同样的功能
