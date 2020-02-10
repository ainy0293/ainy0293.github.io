---
layout: post
title: "Linux添加开机自动启动服务"
subtitle: 'Linux开机自动启动服务，systemctl 和 rc.local方法'
author: "Ainy"
header-style: text
tags:
  - linux
  - service
  - auto
---

#### 使用 systemctl 方式

在 <code> /lib/systemd/system/ </code> 目录创建文件，并写以下内容

```shell
[Unit]
Description=liquid api server Compatibility
After=network.target

[Service]
User=go
ExecStart=/data/subfile/liquid-api-server/liquid-api
#ExecStartPre=rm -rf /data/subfile/tmp/cache/*
#ExecStartPost=
Restart=on-failure
RestartSec=5s
SysVStartPriority=99

[Install]
WantedBy=multi-user.target
```
#### 注解
<code>[Unit]</code> 区块，启动顺序与依赖关系
<code>Description</code> 服务注释，可随意写
<code>After</code> 指定在某个服务启动以后，再启动此服务，比如这个是依赖网络的，那就在网络服务启动之再启动这个

<code>[Service]</code> 区块是核心，主要在这里
<code>User</code>  指定运行程序的用户，或不指定，则是root用户
<code>ExecStart</code> 指定运行的程序路径和参数，注意：这里必须是绝对路径
<code>ExecStartPre</code> 指定运行程序前的指令，比如每次启动前清除缓存
<code>ExecStartPost</code> 启动后的执行后的指令
<code>Restart</code> 指定重启行为，<code>on-failure</code> 代表非正常退出时重启（退出代码非0）
<code>RestartSec</code> 退出后重启的间隔
<code>SysVStartPriority</code> 优先级，自定服务一般设置为99即可
<code>[install]</code> 区块保持默认即可

#### 使用 启动脚本方式

```shell
 /data/subfile/liquid-api-server/liquid-api
2019/12/07 10:11:07 configFile.Get err
#open config/app-local.yaml: no such file or directory
```
如上日志，目前咱们这个服务，没办法使用绝对路径来启动，必须先cd到目录，才可以启动，所以只能用这种方法

ubuntu 18.04 以后，默认没有开户 <code>/etc/rc.local</code> 开机启动脚本，首先开启它

修改 <code>/lib/systemd/system/rc-local.service</code> 文件，内容保持和下方一样即可

```shell
[Unit]
Description=/etc/rc.local Compatibility
Documentation=man:systemd-rc-local-generator(8)
#ConditionFileIsExecutable=/etc/rc.local
After=network.target

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
RemainAfterExit=yes
GuessMainPID=no
SysVStartPriority=99

[Install]
WantedBy=multi-user.target
```

创建 <code>/etc/rc.local</code> 文件
```shell
sudo touch /etc/rc.local

sudo chmod +x /etc/rc.local
```
启用 <code> rc.local</code> 脚本
```shell
systemctl enable rc-local.service
```

### 创建启动脚本，并添加自起服务

```shell
sudo vim /usr/share/subtitle.sh
```
脚本位置无所谓，记住就好，后面会用到

文件添加以下内容，咱们这个因为必须 cd到目录才可以启动，并且指定 go 用户运行，所以脚本如下，其它的参考示例了。

```shell
#!/bin/bash
sleep 10
cd /data/subfile/liquid-api-server
sudo -u go nohup /data/subfile/liquid-api-server/liquid-api > /dev/null 2>&1 &
sleep 1
cd /data/subfile/subtitle-server
sudo -u go nohup /data/subfile/subtitle-server/subtitle > /dev/null 2>&1 &
```
这个启用了 <code>nohup</code>来保证不会因为shell的断开而停止后台运行， <code> > /dev/null 2>&1 & </code> 表示将程序所产生的所有输出（不论标准或错误），全部重定向到 <code>/dev/null</code> ，并且后台运行， 默认情况下，<code>nohup</code> 会在执行的目录创建<code>.nohup</code>来记录程序的标准输出和错误输出，而有些程序的标准输出量特别多，时间长了，会把磁盘空间占满，所以重写向到垃圾桶，可保证不会因为程序的输出信息而占满磁盘空间带来其它的问题。

添加执行权限
```shell
sudo chmod +x /usr/share/subtitle.sh
```

添加到 <code>/etc/rc.local</code>
```shell
sudo vim /etc/rc.local
```
添加e脚本的问题，注意脚本需要有执行位权限：
```shell
#!/bin/sh -e

/usr/share/subtitle.sh

eixt 0 
```

以后所有添加的其它服务，都必须添加在 <code>exit 0</code> 这一行的前面


