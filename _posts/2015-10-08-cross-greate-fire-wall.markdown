---
layout:     post
title:      "科学上网篇"
subtitle:   "搭建Shadowsocks"
date:       2015-10-08 18:33:00
author:     "Paul Du"
header-img: "img/post-bg-01.jpg"
---

### 优点

比PPTP搭建的VPN易用, 可选择自动代理和全局代理, 自动代理只走浏览器的, 通过白名单方式翻墙
缺点是没有帐号体系, 只能设置一个密码(对于个人用户完全足够了)

### 搭建Shadowsocks

[戳这里看官方文档](https://github.com/shadowsocks/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)

#### 安装

`` apt-get install python-pip ``

`` pip install shadowsocks ``

#### 启动和停止

`` sudo ssserver -p 443 -k password -m rc4-md5 --user nobody -d start ``

`` sudo ssserver -d stop ``

#### 检查日志

`` sudo less /var/log/shadowsocks.log ``

#### 客户端

[下载地址](https://github.com/shadowsocks/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E#%E5%AE%A2%E6%88%B7%E7%AB%AF)

#### 客户端配置

`服务器IP` + 端口: `443`

加密方式: `rc4-md5`

密码: `password`


