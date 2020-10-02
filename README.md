## 个人博客


## 如何本地调试

[参考](https://github.com/Starefossen/docker-github-pages.git)

```bash
# 1. 进入到项目的根目录
cd {项目的根目录}

# 2. 运行容器本地测试
docker run -it --rm -v "$PWD":/usr/src/app -p "4000:4000" starefossen/github-pages
```