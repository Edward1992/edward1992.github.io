---
title: Docker 折腾小记
tag: [docker]
---

docker 是个好东西。在本地开发时可以简化开发环境的配置流程。不过 docker 本身也需要一定的学习成本。折腾了好些时日，踩了几个坑。

# 国内镜像
普通网络环境下访问 docker hub 上的镜像时网络会非常地慢。为此安装了 docker 以后首当其冲的就是配置国内的镜像。

国内的 docker 镜像仓库有ustc、daocloud、alicloud 还有官方源。尝试了几个选项后，果然还是[官方源](https://www.docker-cn.com/registry-mirror)用起来最顺手。

在 pull 镜像或者写 dockerfile 的时候，直接在镜像名前加上 docker-cn 的前缀就能从国内镜像源拉取了。


``` bash
docker pull registry.docker-cn.com/library/python:latest # 命令行拉取
```

```
FROM registy.docker-cn.com/library/python:latest
...
...
...
```

也可以修改系统 */etc/docker/daemon.json* 配置 docker 守护进程默认使用国内镜像源加速。

``` json
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

*修改 daemon.json 时注意用户权限*

# Docker image file
Docker 拉取或者本地创建的镜像都会存储单一的一个Docker.qcow2 文件当中。随着镜像的增多这个文件会变得越来越大。而且比较 buggy 的是即使你删除了一些镜像，这个Docker.qcow2也不会随之减小。

![screenshot1](https://github.com/Edward1992/edward1992.github.io/blob/master/image/2017/10/2017-10-30-5.01.08.png?raw=true)

![screenshot2](https://github.com/Edward1992/edward1992.github.io/blob/master/image/2017/10/2017-10-30-5.01.20.png?raw=true)



目前 docker 官方还没有要处理这个问题的迹象。我们用户现在能够做的就是简单删除掉这个.qcow2文件来优化存储空间。一个叫Théo Chamley的法国人为此写了一个[清理 Docker.qcow2的 bash 脚本](https://gist.github.com/MrTrustor/e690ba75cefe844086f5e7da909b35ce#file-clean-docker-for-mac-sh)。
目前我也是定期看着 docker images 里镜像所占用的空间估摸着要不要跑跑这个清理脚本。

``` bash
#!/bin/bash

# Copyright 2017 Théo Chamley
# Permission is hereby granted, free of charge, to any person obtaining a copy of 
# this software and associated documentation files (the "Software"), to deal in the Software
# without restriction, including without limitation the rights to use, copy, modify, merge,
# publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons
# to whom the Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all copies or
# substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING
# BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
# NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
# DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,

IMAGES=$@

echo "This will remove all your current containers and images except for:"
echo ${IMAGES}
read -p "Are you sure? [yes/NO] " -n 1 -r
echo    # (optional) move to a new line
if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    exit 1
fi


TMP_DIR=$(mktemp -d)

pushd $TMP_DIR >/dev/null

open -a Docker
echo "=> Saving the specified images"
for image in ${IMAGES}; do
	echo "==> Saving ${image}"
	tar=$(echo -n ${image} | base64)
	docker save -o ${tar}.tar ${image}
	echo "==> Done."
done

echo "=> Cleaning up"
echo -n "==> Quiting Docker"
osascript -e 'quit app "Docker"'
while docker info >/dev/null 2>&1; do
	echo -n "."
	sleep 1
done;
echo ""

echo "==> Removing Docker.qcow2 file"
rm ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2

echo "==> Launching Docker"
open -a Docker
echo -n "==> Waiting for Docker to start"
until docker info >/dev/null 2>&1; do
	echo -n "."
	sleep 1
done;
echo ""

echo "=> Done."

echo "=> Loading saved images"
for image in ${IMAGES}; do
	echo "==> Loading ${image}"
	tar=$(echo -n ${image} | base64)
	docker load -q -i ${tar}.tar || exit 1
	echo "==> Done."
done

popd >/dev/null
rm -r ${TMP_DIR}
```
