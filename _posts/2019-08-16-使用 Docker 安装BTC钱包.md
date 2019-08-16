---
layout:     post
title:      使用 Docker 安装BTC钱包
subtitle:   使用 Docker 部署BTC钱包，另有 Dokcerfile
date:       2019-08-16
author:     Trojan
header-img: img/post-bg-9.jpg
catalog: true
tags:
    - Docket
---

## 使用 Docker 部署BTC钱包

### **环境**
>  ubuntu 18.04  
>  docker 19.03.1  
>  *bitcoin-core-0.18.1*

安装docker步骤略过

### **基础环境**
1. Docker 拉取最新的 ubuntu image  

```
docker pull ubuntu 
```
 
2. 然后查看镜像
```
➜  ~ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              a2a15febcdf3        20 hours ago        64.2MB
```
### **安装钱包**
1. 启动镜像 (-i: 以交互模式运行容器，通常与 -t 同时使用; -t: 为容器重新分配一个伪输入终端;)
```
➜  ~ docker run -it --name btcwallet ubuntu
root@391f2aa473eb:/# 
```   
2. 安装wget(为了下载钱包文件)   
```
root@391f2aa473eb:/#  apt-get update && apt-get install -y wget
```
3. 下载钱包并且解压安装   
```
root@391f2aa473eb:/# wget https://bitcoin.org/bin/bitcoin-core-0.18.1/bitcoin-0.18.1-x86_64-linux-gnu.tar.gz -O - | tar -xz
root@391f2aa473eb:/# install -m 0755 -o root -g root -t /usr/local/bin ./bitcoin-0.18.1/bin/*
```
### **启动钱包**
1. 先在前台运行方式启动钱包，让它产生基础配置文件(配置文件位置在  ***~/.bitcoin***  目录下)
```
root@391f2aa473eb:/# bitcoind
root@391f2aa473eb:/# cd ~/.bitcoin && ls
banlist.dat  blocks  chainstate  debug.log  fee_estimates.dat  mempool.dat  peers.dat  wallets
```
2. 钱包部分命令
```
bitcoind -daemon // 钱包后台运行
bitcoin-cli stop // 停止钱包服务
bitcoin-cli getblockcount // 获取区块数量
```  
到此我们在 ***Docker*** 中安装钱包的步骤就结束了   

### **将容器打包成镜像**
> 这一步主要是为了方便以后使用   
1. 卸载多余软件并且退出容器回到宿主机
```
root@391f2aa473eb:/# apt-get --purge -y remove wget && exit
```
2.  查看容器
```
➜  ~ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
391f2aa473eb        ubuntu              "/bin/bash"         3 hours ago         Exited (0) 34 seconds ago                       btcwallet
```   
3. 打包   
   > docker commit -m  "提交信息"   -a  "作者"   [CONTAINER ID]  [给新的镜像命名]
```
➜  ~ docker commit -a "Trojan" -m "Create BTC wallet" 391f2aa473eb trojan/btcwallet
sha256:3b8d452b9cb2f6513fab9aaa8b26771e8cf91d66cce901e79d13f532e610a191
```
4. 查看镜像 (trojan/btcwallet 就是刚才容器打包成的镜像)
```
➜  ~ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
trojan/btcwallet    latest              3b8d452b9cb2        17 seconds ago      272MB
ubuntu              latest              a2a15febcdf3        23 hours ago        64.2MB
```   
<span id="jump"></span>
### **配置钱包并且启动**

1. 钱包配置文件  
   > 钱包文件放在宿主机的位置（我这里放在 ***~/wallet/btc*** 文件夹下）
```
➜  ~ mkdir -p ~/wallet/btc
➜  ~ vim ~/wallet/btc/bitcoin.conf

#监听模式，默认启动
listen=1
#事务索引
txindex=1
#允许bitcoin接收JSON-RPC
server=1
#RPC用户名
rpcuser=******
#RPC密码
rpcpassword=******
#RPC端口
rpcport=8332
#允许RPC访问ip
rpcallowip=127.0.0.1
rpcallowip=0.0.0.0/24

```   
> esc :wq 保存退出

1. 启动容器 
   > (数据卷挂载) 并且将配置文件进容器；将钱包文件也映射出来   
   >  -v:绑定一个卷; -p: 指定端口映射，格式为：主机(宿主)端口:容器端口
```
➜  ~ docker run -itd -p 18332:8332 -v ~/wallet/btc:/root/.bitcoin --name wallet_btc trojan/btcwallet bitcoind
```
---
Docker 安装BTC钱包就结束了 
---   

### 使用 Dockerfile 安装BTC钱包
1. Dockerfile 文件
```
FROM ubuntu:latest
RUN apt-get update && apt-get install wget \
    && wget https://bitcoin.org/bin/bitcoin-core-0.18.1/bitcoin-0.18.1-x86_64-linux-gnu.tar.gz -O - | tar -xz \
    && install -m 0755 -o root -g root -t /usr/local/bin ./bitcoin-0.18.1/bin/* \
    && apt-get --purge -y remove wget
ENTRYPOINT ["bitcoind"]
```

2. 使用Dockerfile 创建钱包镜像   
```
➜  ~ docker build -t [标签] .
```
3. 启动   
[点击跳转](#jump)

   

