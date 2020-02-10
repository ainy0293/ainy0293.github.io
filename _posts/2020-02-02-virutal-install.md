---
layout: post
title: "虚拟机安装"
subtitle: 'windows虚拟机在proxmox kvm虚拟化平台的安装和驱动加载'
author: "Ainy"
header-style: text
tags:
  - kvm
  - proxmox
  - vm
  - cloud
---

在proxmox平台使用虚拟机，虚拟机的硬盘，建议使用virtio类型，因为proxmox的虚拟机，是基于kvm，virtio是kvm支持的最好，也是性能最好的。网卡也是一样，使用virtio，性能最好

virtio ，windows 系统，不带个驱动，Linux 自带，所以在安装windows虚拟机的guest os时，会出现找不到磁盘驱动器的情况，如下图

![no virtual disk](https://ifmx.cc/images/nohd.png)

### 安装流程

创建虚拟机的时候，磁盘驱动器，选择 <code>virtio</code>

![](https://ifmx.cc/images/virtio_vd.png)

网卡同样也选择 <code>virtio</code>

![](https://ifmx.cc/images/virtio_eth.png)



---



### guest os 驱动安装

![](https://ifmx.cc/images/nohd.png)



如上图所示，找不到驱动，这时候，到虚拟机的硬件设置里，把镜像切换一下，如下图：

![](https://ifmx.cc/images/cdiso1.png)

之后再返回到虚拟机安装界面，添加驱动，点击 <code>加载驱动程序</code>

![](https://ifmx.cc/images/adddrive.png)

再点击 <code>浏览</code>

![](https://ifmx.cc/images/adddrive2.png)

之后展开 <code>CD驱动器 virtio-win-0.1.1</code> ，并往下拉

![](https://ifmx.cc/images/drive1.png)

展开 <code>viostor</code> 文件夹，一般最最后一个文件夹，点前面的 <code>+</code>号

![](https://ifmx.cc/images/drive2.png)

之后可以看到已经加载出来的驱动，点击 <code>下一步</code>

![](https://ifmx.cc/images/drive3.png)

稍后几秒，就会返回安装界面了，这时候可以看到提示，无法安装windows。

这是因为我们之前切换的 iso 镜像，没有切换回来，再把再到虚拟机的硬件设置，把iso镜像切换回来

![](https://ifmx.cc/images/drive4.png)

切换镜像

![](https://ifmx.cc/images/cdiso2.png)


换完之后，再返回安装界面，并点 <code>刷新</code>

![](https://ifmx.cc/images/refresh1.png)

刷新完之后，可以看到已经正常，点击 <code>下一步</code> 就可以安装系统了

![](https://ifmx.cc/images/refresh2.png)


以上实例是 windows server 2008 R2，其它的windows 系统，也是一样（xp和2003除外）


