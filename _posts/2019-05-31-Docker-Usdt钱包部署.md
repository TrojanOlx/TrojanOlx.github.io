---
layout:     post
title:      docker 部署Usdt钱包
subtitle:   虚拟币钱包部署
date:       2019-05-31
author:     Trojan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Docker
---


> 环境 centos 7

### 安装 docker

[docke安装](https://docs.docker.com/install/linux/docker-ce/centos/){:target="_blank"}


```
FROM ubuntu
WORKDIR /app
EXPOSE 8332

RUN apt-get update
RUN apt-get -y install wget

RUN cd /app
RUN wget https://bintray.com/artifact/download/omni/OmniBinaries/omnicore-0.5.0-x86_64-linux-gnu.tar.gz

RUN tar -xzvf omnicore-0.5.0-x86_64-linux-gnu.tar.gz

RUN cd omnicore-0.5.0-x86_64-linux-gnu/bin

ENTRYPOINT ["/app/omnicore-0.5.0-x86_64-linux-gnu/bin/omnicore", "--daemon"]
```

修改配置文件 ~/.bitcoin/bitcoin.conf

```
#监听模式，默认启动
listen=1  
#允许bitcoin接收JSON-RPC
server=1  
#RPC用户名
rpcuser=bitcoin 
#RPC密码
rpcpassword=********
#RPC端口
rpcport=8332
#允许RPC访问ip
rpcallowip=127.0.0.1
```

> 生成docker image
```
docker build --tag usdt/ubuntu:v1 .
```
>启动容器
```
docker run -p 8332:8332 -v /opt/bitcoin/:/root/ --name="usdt1" -d usdt/ubuntu:v1
```


