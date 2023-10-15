---
layout: fragment
title: 本地容器化方式搭建 gitlab
tags: [tool, gitlab]
description: logseq descripition
keywords: gitlab
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 1. 创建

`bash start_gitlab.sh`

> 目录结构

```bash
root@debian:~/start_gitlab# pwd
/root/start_gitlab
root@debian:~/start_gitlab# ls -alh
total 24K
drwxr-xr-x 2 root root 4.0K Oct 10 22:34 .
drwx------ 6 root root 4.0K Oct 10 03:34 ..
-rw-r--r-- 1 root root 1.2K Oct 10 22:25 gitlab-docker-compose.yaml
-rwxr-xr-x 1 root root  326 Oct 10 22:23 runner_register.sh
-rw-r--r-- 1 root root   62 Oct 10 22:34 start_gitlab_runner.sh
-rwxr-xr-x 1 root root  424 Oct  9 22:48 start_gitlab.sh
```

> start_gitlab.sh

```bash
#!/bin/bash
readonly gitlab_home="/root/GITLAB_HOME"
readonly docker_compose="gitlab-docker-compose.yaml"

export GITLAB_HOME="${gitlab_home}"

SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

if [[ ! -d "${gitlab_home}" ]]; then
    mkdir -p "${gitlab_home}"
fi

if ! command -v docker > /dev/null 2>&1; then
    echo "docker is required"
    exit 1
fi

docker compose -f "${SCRIPT_DIR}/${docker_compose}" up -d
```

> gitlab-docker-compose.yaml

```yaml
ersion: "3.6"
services:
  gitlab:
    image: "gitlab/gitlab-ce:latest"
    container_name: "gitlab"
    restart: always
    hostname: "gitlab.debian.com"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.debian.com:8929'
        # Add any other gitlab.rb configuration here, each on its own line
        pages_external_url 'http://gitlab.debian.com:8929'
        gitlab_pages['enable'] = true
        gitlab_rails['initial_root_password'] = '12345678Abc='
        gitlab_rails['display_initial_root_password'] = true
        gitlab_rails['gitlab_shell_ssh_port'] = 8022
    ports:
      - "8080:80"
      - "8929:8929"
      - "8443:443"
      - "8022:22"
    volumes:
      - "$GITLAB_HOME/gitlab/config:/etc/gitlab"
      - "$GITLAB_HOME/gitlab/logs:/var/log/gitlab"
      - "$GITLAB_HOME/gitlab/data:/var/opt/gitlab"
    shm_size: "256m"

  gitlab-runner:
    image: "gitlab/gitlab-runner:latest"
    container_name: "gitlab-runner"
    restart: always
    # will add to xxx/gitlab-runner/config/config.yaml
    # that is,runners.docker.network_mode="host"
    # ref: https://docs.docker.com/compose/compose-file/compose-file-v3/#network_mode
    # ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html network_mode
    # network_mode 实测不生效，需要在启动成功后手动在 gitlab-runer 的配置文件中手动添加上
    network_mode: host
    volumes:
      - "$GITLAB_HOME/gitlab-runner/config:/etc/gitlab-runner"
      - "/var/run/docker.sock:/var/run/docker.sock"
    depends_on:
      - gitlab
```

> runner_register.sh

```bash
docker exec -it gitlab-runner gitlab-runner register \
  --url http://gitlab.debian.com:8929/ \
  --registration-token GR13489418KqMKLn495Q11bszzHoR\
  --executor docker \
  --description "Docker Runner" \
  --docker-privileged \
  --docker-image "docker:latest" \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

> start_gitlab_runner.sh

```bash
#!/bin/bash

docker exec -it gitlab-runner gitlab-runner start
```

# 2. 访问
- 用户名和密码

| key      | value         |
| -------- | ------------- |
| username | root          |
| password | 123456789Abc= |

> 可从 `cat $GITLAB_HOME/config/initial_root_password` 获取

- 修改本机及运行 gitlab 的机器上的的 /etc/hosts, 添加：`192.168.155.111 gitlab.debian.com`
  > a. 其中 192.168.155.111 是 gitlab 所在机器的 IP
  > 
  > b. 如果开启了 项目的 pages, 同时需要在 /etc/hosts 中添加 `192.168.155.111 gitlab.debian.com {group_name}.gitlab.debian.com`
  > ![image](https://github.com/double12gzh/double12gzh.github.io/assets/2534467/eb5bbcf5-0350-49e3-9c52-bedf2b95e6f3)
- 访问主页 `http://gitlab.debian.com:8929/`
- 访问项目 pages
  示例：
  
  http://personal.gitlab.debian.com:8929/blogger
  
  http://personal.gitlab.debian.com:8929/wiki
  ![image](https://github.com/double12gzh/double12gzh.github.io/assets/2534467/2215d65f-6093-4344-abff-e042eb06a829)
  

# 3. 配置 Runner

在 GitLab 上注册 Runner，执行以下步骤：

- 登录到 GitLab
- 转到的项目
- 在左侧导航中选择 Settings > CI / CD， 找到 Runners 部分，复制注册 token
- 注册 Runner `bash runner_register.sh`
- 修改 gitlab-runner 的配置文件 $GITLAB_HOME/gitlab-runner/config/config.yaml,**添加 network_mode = "host"**
  
  **注意： 每次注册完后都要添加一下 network_mode = "host", 否则将不会使用主机网络，从而导致无法访问外网**
  ![image](https://user-images.githubusercontent.com/2534467/273965926-993c7ba7-fa43-4575-98da-dbde3cbb985d.png)
- 重启 gitlab-runner `docker restart gitlab-runner`
- （可选）启动 gitlab-runner `bash start_gitlab_runner.sh`

> 说明：
>
> - 除了 token 需要修改外，其它的选默认就可
> - 将 YOUR_REGISTRATION_TOKEN 替换为从 GitLab 页面复制的注册 token。示例：
>   ![image](https://user-images.githubusercontent.com/2534467/273678185-bf6bc535-86da-4a9f-bf9c-16a47451b979.png)

# 4. 项目 Pages 配置
## 4.1 Demo1 - 托管 tiddlywiki
### 4.1.1 项目结构
   
![image](https://github.com/double12gzh/double12gzh.github.io/assets/2534467/4ad72a8b-b05a-415b-a3e7-2c40ff171489)


### 4.1.2 `.gitlab-ci.yml`

```yaml
default:
  tags:
    - blog

pages:
  script:
    - mkdir public
    - cp index.html public/index.html
  artifacts:
    paths:
      - public
  only:
    - main
```

### 4.1.3 `index.html`

为 `tiddlywiki` 的 `html` 文件

## 4.2 Demo2 - 托管 jekyll 博客
### 4.2.1 项目结构

![image](https://github.com/double12gzh/double12gzh.github.io/assets/2534467/4df5381f-8826-46c8-a1e6-7ce1ea8396ca)
[参考](https://github.com/double12gzh/double12gzh.github.io)

### 4.2.2 `.gitlab-ci.yml`

```yaml
image: ruby:2.7

default:
  tags:
    - blog

workflow:
  rules:
    - if: $CI_COMMIT_BRANCH

cache:
  paths:
    - vendor/

before_script:
  - gem install bundler
  - bundle install --path vendor

pages:
  stage: deploy
  script:
    - bundle exec jekyll build -d public
  artifacts:
    paths:
      - public
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  environment: production

test:
  stage: test
  script:
    - bundle exec jekyll build -d test
  artifacts:
    paths:
      - test
  rules:
    - if: $CI_COMMIT_BRANCH != "main"
```

# 5. 出处

https://gist.github.com/double12gzh/66b5dc0a53e670da9733c66d2b0ad4f5
https://gitlab.com/double12gzh/blogdemo/-/blob/a4d5b5b949c565a1b8f7a11605585d35ce58c534/how-to-setup-local-hosted-gitlab.md
