---
layout: post
title: "树莓派4B Docker环境安装OpenWrt及配置代理服务"
date:   2024-5-2
tags: [Openwrt,rpi,guide]
comments: true
author: YanLuo洛(zaz203)
---

温馨提示：本文共1877词，大约需要十分钟阅读。

##### -Intro:为什么要安装⌈OpenWrt⌋和写本篇文章?

本段引用自 Copilot 智能总结：

> 刷OpenWrt可以为你的路由器带来许多好处，使其功能更加丰富和强大。OpenWrt是一个开源的嵌入式Linux系统，专为路由器和其他嵌入式设备设计。以下是刷OpenWrt的一些主要好处：**自定义功能和扩展性**、**提高网络性能和安全性**、**支持多种硬件和协议**、**学习和开发平台**。

在查阅网上资料时，发现网上很多资料都已过期或是在操作过程中出现了错误，因此写下本篇文章来记录步骤。

##### -Preparation:准备阶段

本篇博客更新于 2025年5月2日，部分信息可能已过期，请注意甄别。

基础环境：**未经任何修改**的Ubuntu 24.04 Server环境。

##### -Step1:基础准备

###### 若你已经换源并安装 Docker，可跳过该步骤。

通过SSH登入你的树莓派。

在 命令行输入 来登入root用户，需要身份验证。

```bash
sudo -i
```

输入 并跟随屏幕操作指示来进行换源操作。

```bash
bash <(curl -sSL https://linuxmirrors.cn/main.sh)
```

随后，输入 并全部按照默认指示来实现 更换Docker源 与 安装Docker的操作。

```bash
bash <(curl -sSL https://linuxmirrors.cn/docker.sh)
```

##### -Step2:网络配置

本步骤将帮助你进行基本网络的配置

###### ·开启网卡混杂模式

> 混杂模式就是接收所有经过网卡的数据包，包括不是发给本机的包，即不验证MAC地址。普通模式下网卡只接收发给本机的包（包括广播包）传递给上层程序，其它的包一律丢弃。

树莓派需要在 eth0 接口开启。

```bash
sudo ip link set eth0 promisc on
```

执行下方命令，若返回的信息含`PROMISC`即表示成功。

```bash
ifconfig eth0
```

###### ·配置容器网络

1.创建 `macvlan`

> `macvlan` 是一种网卡虚拟化技术，允许在同一个物理网卡上配置多个 MAC 地址，即多个 `interface`，每个 `interface` 可以配置自己的 IP；`macvlan`直接通过以太网的 `interface` 连接到物理网络，因此性能极好
>
> 因此，软路由需要使用 `macvlan` 配合混杂模式在容器中实现路由功能
>
> Docker 创建 `macvlan` 时要确定所在的网段，可以在路由器后台进行确认；如小米路由器常用的是 `192.168.31.0/24`网段；在创建网络时需要保证子网网段`subnet`和网关地址`gateway`参数与当前网络一致

```bash
docker network create -d macvlan --subnet=192.168.31.0/24 --gateway=192.168.31.1 -o parent=eth0 macnet
```

2.创建容器

```bash
docker run --restart always --name openwrt -d --network macnet --privileged   sulinggg/openwrt:rpi4 /sbin/init
```

> [!IMPORTANT]
>
> 此命令仅限 RPI 4B 使用，3B未经过测试，请使用对应型号的镜像。

3.修改容器网络配置

创建好容器后通常无法上网，需要进入bash修改网络配置。

```bash
docker exec -it openwrt bash
```

修改网络配置

```bash
vi /etc/config/network
```

你应该找到：`config interface 'lan'`这一项目

将其修改为

```bash
config interface 'lan'
option type 'bridge'
option ifname 'eth0'
option proto 'static'
option netmask '255.255.255.0'
option ip6assign '60'
option ipaddr '192.168.31.41'
option gateway '192.168.31.1'
option dns '114.114.114.114'
```

> [!NOTE]
>
> 上表中的`ipaddr`为OpenWrt的IP地址，用于访问路由器。
>
> `gateway`及网关，填写你主路由器的地址，仅接受纯IP。
>
> `dns`通常填写国内DNS或是主路由地址。

修改完成后重启容器网络

```bash
/etc/init.d/network restart
```

重启完成后在浏览器访问  http://192.168.31.41 输入用户名`root`和密码`root`来登入后台。

> [!IMPORTANT]
>
> 请在登入后立即更改访问密码，以免非法登录。

##### -Step3:配置客户端

对于 Android/IOS 设备，在`WLAN`界面设置。

OpenClash端口为 7890

![Screenshot_20250502-141251](https://raw.githubusercontent.com/zaz203/zaz203.github.io/main/images/Screenshot_20250502-141251.png)

对于 Windows 设备，在`网络与Internet连接-代理-手动设置`页面设置。

![屏幕截图 2025-05-02 141631](https://raw.githubusercontent.com/zaz203/zaz203.github.io/main/images/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-05-02%20141631.png)

> [!NOTE]
>
> 对于不想配置代理的用户，该教程到此结束。

##### -Step4:配置代理

使用的镜像已经自带`PassWall``SS+``Openclash`这三个插件。

本教程以`Openclash`为基础配置

###### ·导入配置

1.点击`服务-OpenClash`进入界面。点击上方`配置文件订阅`，按照页面提示导入配置。

> [!NOTE]
>
> 通常无需配置`订阅转换模板`，请勿勾选相关选项。

###### ·配置模式

2.点击`全局设置`，点击下方`切换页面到 Fake-IP 模式`，选择增强，按如图所示配置。

![屏幕截图 2025-05-02 142508](https://raw.githubusercontent.com/zaz203/zaz203.github.io/main/images/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-05-02%20142508.png)

> [!IMPORTANT]
>
> - OpenClash 运行模式：
>
> Fake-IP（增强）模式：
>
> ```
> 客户端进行通讯时会先进行DNS查询目标IP地址，拿到查询结果后再尝试进行连接。
> 
> Fake-IP 模式在客户端发起DNS请求时会立即返回一个保留地址（198.18.0.1/16），同时向上游DNS服务器查询结果，如果判定返回结果为污染或者命中代理规则，则直接发送域名至代理服务器进行远端解析。
> 
> 此时客户端立即向Fake-IP发起的请求会被快速响应，节约了一次本地向DNS服务器查询的时间。
> 
> 实际效果：客户端响应速度加快，浏览体验更加顺畅，减轻网页加载时间过长的情况。
> ```
>
> Redir-Host（兼容）模式：
>
> ```
> 客户端进行通讯时DNS由Clash先进行并发查询，等待返回结果后再尝试进行规则判定和连接。
> 
> 当判定需要代理时，使用fallback组DNS的查询结果进行通讯
> 
> 实际效果：客户端响应速度一般，可能出现网页加载时间过长的情况。
> ```
>
> 
>
> Redir-Host（TUN）模式
>
> ```
> 此模式与Redir-Host（兼容）模式类似，不同在于能够代理所有UDP链接，提升nat等级，改善游戏联机体验。
> ```
>
> 
>
> Fake-IP（TUN）模式：
>
> ```
> 此模式与Fake-IP（增强）模式类似，不同在于能够代理使用域名的UDP链接。
> ```
>
> 
>
> Redir-Host（游戏）模式
>
> ```
> 此模式与Redir-Host（兼容）模式类似，不同在于能够代理所有UDP链接，提升nat等级，改善游戏联机体验。
> ```
>
> 
>
> Fake-IP（游戏）模式：
>
> ```
> 此模式与Fake-IP（增强）模式类似，不同在于能够代理所有UDP链接，提升nat等级，改善游戏联机体验。
> ```

3.在下方的`设置 SOCKS5/HTTP(S) 认证信息`中删除认证信息，如果你不想要设备输入密码才能访问代理的话。

4.点击上方`Meta`设置，启用`Meta内核`、`启用 TCP 并发`、`启用流量（域名）探测`这三个选项。

5.点击上方`大陆白名单订阅`，设置自动更新。

###### 随后点击下方的 保存配置，在主界面开启 OpenClash 即可。

#### 教程完

##### 作者

YanLuo洛，

http://zaz203.github.io

Coolapk@Luoyanxyy

X(Formerly Twitter)@kekeyanluo

如有问题，欢迎联系 [邮件](mailto:zeromostia@gmail.com)，我将会尽力回答。

##### 致谢：

引用了下列内容：

https://github.com/vernesong/OpenClash

[树莓派 4B 容器方式安装 OpenWrt 作为软路由](https://blog.hellowood.dev/posts/树莓派-4b-容器方式安装-openwrt-作为软路由/)






