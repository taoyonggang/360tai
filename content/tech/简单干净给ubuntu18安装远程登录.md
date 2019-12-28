+++
title = "简单干净给ubuntu18安装远程登录"
date = "2019-12-28T21:18:50+08:00"
tags = ["ubuntu", "xrdp","远程桌面"]
slug = "simple-install-remote-desktop-on-ubuntu19"
dropCap = false
Categories = ["ubuntu", "xrdp","远程桌面"]
+++

## 先安装简单桌面环境 [^1]
```
sudo apt update
sudo apt install xfce4 xfce4-goodies xorg dbus-x11 x11-xserver-utils
```
## 安装xrdp
```
sudo apt install xrdp 
```
安装完成后检查xrdp状态
```
sudo systemctl status xrdp
```
## 将xrdp用户加入ssl证书用户组
```
sudo adduser xrdp ssl-cert  
```
## 配置xrdp
```
sudo nano /etc/xrdp/xrdp.ini
```
在编辑文件末尾增加：
```
exec startxfce4 
```
重启xrdp
```
sudo systemctl restart xrdp
```
## 防火墙处理（如果有的话）
```
sudo ufw allow 3389
```
开放3389端口，更多配置参考防火墙说明文档。

## 打开windows远程桌面客户端mstc

使用xor模式，你就能进入ubuntu桌面环境了，如图：

![xrdp-xfce-desktop.jpg](/images/xrdp-xfce-desktop.jpg "远程登录效果图")

这样安装，软件装的最少，配置最少，最稳定。


[^1]: 来源：https://linuxize.com/post/how-to-install-xrdp-on-ubuntu-18-04/

