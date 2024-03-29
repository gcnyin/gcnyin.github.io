---
title: "使用haproxy，nginx与aria2搭建下载服务"
date: 2020-12-26T00:00:00+08:00
categories:
- 技术
tags:
- haproxy
---

最近搞了一台vps，用它搭建了一个远程下载服务（网盘），包括aria2做下载工具，nginx做静态文件服务，haproxy根据hostname做代理。

架构大致如下。

![wangpan-arch](/wangpan-arch.jpg)


Aria2配置如下，大家基本可以直接照抄，只需要修改下`rpc-secret`，用`aria2c --conf-path=/root/aria2/aria2.conf -D`命令启动即可。

```conf
# touch /data/aria2.session
# vim /etc/aria2/aria2.conf
## '#'开头为注释内容, 选项都有相应的注释说明, 根据需要修改 ##
## 被注释的选项填写的是默认值, 建议在需要修改时再取消注释  ##

## 文件保存相关 ##

# 文件的保存路径(可使用绝对路径或相对路径), 默认: 当前启动位置
dir=/root/Downloads

# 启用磁盘缓存, 0为禁用缓存, 需1.16以上版本, 默认:16M
disk-cache=32M
# 文件预分配方式, 能有效降低磁盘碎片, 默认:prealloc
# 预分配所需时间: none < falloc ? trunc < prealloc
# falloc和trunc则需要文件系统和内核支持
# NTFS建议使用falloc, EXT3/4建议trunc, MAC 下需要注释此项
file-allocation=trunc
# 断点续传
continue=true

## 下载连接相关 ##

# 最大同时下载任务数, 运行时可修改, 默认:5
max-concurrent-downloads=10
# 同一服务器连接数, 添加时可指定, 默认:1
max-connection-per-server=10
# 最小文件分片大小, 添加时可指定, 取值范围1M -1024M, 默认:20M
# 假定size=10M, 文件为20MiB 则使用两个来源下载; 文件为15MiB 则使用一个来源下载
min-split-size=10M
# 单个任务最大线程数, 添加时可指定, 默认:5
split=5
# 整体下载速度限制, 运行时可修改, 默认:0
#max-overall-download-limit=0
# 单个任务下载速度限制, 默认:0
#max-download-limit=0
# 整体上传速度限制, 运行时可修改, 默认:0
#max-overall-upload-limit=0
# 单个任务上传速度限制, 默认:0
#max-upload-limit=0
# 禁用IPv6, 默认:false
disable-ipv6=true

## 进度保存相关 ##

# 从会话文件中读取下载任务
input-file=/root/aria2/aria2.session
# 在Aria2退出时保存`错误/未完成`的下载任务到会话文件
save-session=/root/aria2/aria2.session
# 定时保存会话, 0为退出时才保存, 需1.16.1以上版本, 默认:0
save-session-interval=60

## RPC相关设置 ##

pause=false
rpc-allow-origin-all=true
rpc-listen-all=true
rpc-save-upload-metadata=true
rpc-secure=false

# 启用RPC, 默认:false
enable-rpc=true
# 允许所有来源, 默认:false
#rpc-allow-origin-all=true
# 允许非外部访问, 默认:false
#rpc-listen-all=true
# 事件轮询方式, 取值:[epoll, kqueue, port, poll, select], 不同系统默认值不同
event-poll=epoll
# RPC监听端口, 端口被占用时可以修改, 默认:6800
rpc-listen-port=6800
# 设置的RPC授权令牌, v1.18.4新增功能, 取代 --rpc-user 和 --rpc-passwd 选项
rpc-secret=XXXXXX
# 设置的RPC访问用户名, 此选项新版已废弃, 建议改用 --rpc-secret 选项
#rpc-user=<USER>
# 设置的RPC访问密码, 此选项新版已废弃, 建议改用 --rpc-secret 选项
#rpc-passwd=<PASSWD>

## BT/PT下载相关 ##

# 当下载的是一个种子(以.torrent结尾)时, 自动开始BT任务, 默认:true
#follow-torrent=true
# BT监听端口, 当端口被屏蔽时使用, 默认:6881-6999
listen-port=51413
# 单个种子最大连接数, 默认:55
#bt-max-peers=55
# 打开DHT功能, PT需要禁用, 默认:true
enable-dht=true
# 打开IPv6 DHT功能, PT需要禁用
#enable-dht6=false
# DHT网络监听端口, 默认:6881-6999
#dht-listen-port=6881-6999
# 本地节点查找, PT需要禁用, 默认:false
bt-enable-lpd=true
# 种子交换, PT需要禁用, 默认:true
enable-peer-exchange=false
# 每个种子限速, 对少种的PT很有用, 默认:50K
#bt-request-peer-speed-limit=50K
# 客户端伪装, PT需要
#peer-id-prefix=-TR2770-
user-agent=Transmission/2.92
#user-agent=netdisk;4.4.0.6;PC;PC-Windows;6.2.9200;WindowsBaiduYunGuanJia
# 当种子的分享率达到这个数时, 自动停止做种, 0为一直做种, 默认:1.0

seed-ratio=1.0
#作种时间大于30分钟，则停止作种
seed-time=30
# 强制保存会话, 话即使任务已经完成, 默认:false
# 较新的版本开启后会在任务完成后依然保留.aria2文件
#force-save=false
# BT校验相关, 默认:true
#bt-hash-check-seed=true
# 继续之前的BT任务时, 无需再次校验, 默认:false
bt-seed-unverified=true
# 保存磁力链接元数据为种子文件(.torrent文件), 默认:false
bt-save-metadata=true
#下载完成后删除.ara2的同名文件
on-download-complete=/root/aria2/delete_aria2
```

Aria2后端搭建好后，还需要一个前端页面去操作，这里选用[AriaNg](https://github.com/mayswind/AriaNg)，下载最新版本，置于`/root/Aria2Ng`目录下，同时配置nginx来host这个页面。Nginx配置如下。注意，使用了8081端口，之后还会提到它。

```
server {
    listen 8081;

    location / {
        autoindex on;
        root /root/AriaNg;
    }
}
```

接着搭建静态文件服务器，在之前的aria2配置里，默认下载目录为`/root/Downloads`，我们只需要建一个文件下载服务器，绑定到这个目录即可。好在nginx提供了开箱即用的文件下载服务器。配置如下，端口为8080。

```
server {
    listen 8080;

    location / {
        autoindex on;
        root /root/Downloads;
    }
}
```

这三个服务全都在同一台机器上，使用端口访问很不方便，所以加一个代理，根据hostname的不同代理到不同服务上。这里我用的是haproxy，haproxy是一一款老牌的proxy应用，易用又强大。核心配置如下。

```
frontend front443
    bind :443 ssl crt-list /etc/haproxy/crt-list.txt alpn http/1.1
    bind :80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }
    acl is_xswl hdr(host) -i xswl.tnb.tw
    acl is_yysy hdr(host) -i yysy.tnb.tw
    acl is_awsl hdr(host) -i awsl.tnb.tw
    use_backend fileserver8080 if is_xswl
    use_backend aria2c6800 if is_yysy
    use_backend AriaNg8081 if is_awsl

backend fileserver8080
    mode http
    server fileserver8080 127.0.0.1:8080

backend AriaNg8081
    mode http
    server AriaNg8081 127.0.0.1:8081

backend aria2c6800
    mode http
    server aria2c6800 127.0.0.1:6800
```

通过acl创建hostname的判断条件，如果符合给定的条件则代理到对应的后端服务上。以下是aria2 frontend与文件下载页面的截图。

![aria2ng](/aria2ng.png)

![nginx-autoindex](/nginx-autoindex.png)
