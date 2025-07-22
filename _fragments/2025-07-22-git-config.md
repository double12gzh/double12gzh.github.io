---
layout: fragment
title: git 配置临时配置
tags: [git, ]
description: git 配置文件
keywords: git
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 说明

需要临时配置 git 仓库的 ssh 信息，以便可以使用指定的私钥访问私有仓库。 `setup.sh` 仅会在当前 shell 会话中生效。

## 临时配置

配置脚本 `setup.sh` 文件内容如下：

```bash
#!/bin/bash

ssh -T git@git.xxx.com -i /root/xxx/.ssh/id_rsa
eval "$(ssh-agent -s)"

ssh-add ~/.ssh/id_rsa
ssh-add -l
```

ssh 配置文件 `~/.ssh/config` 添加以下内容

```bash
Host git.xxx.com
    User guanzenghui
    Port 443
    HostName git.xxxx.com
    IdentityFile /root/xxx/.ssh/id_rsa
    PreferredAuthentications publickey
```

## 使用

```bash
source setup.sh
```
