---
title: 日常开发工具
date: 2017-05-12
---

[TOC]

### 免密码登录

1、本地生产key

2、远程机器创建目录：mkdir ~/.ssh，并设置chmod **0700 ** ~/.ssh，目录内创建文件touch ~/.ssh/authorized_keys，并设置chmod **0600** ~/.ssh/authorized_keys

3、将本地生产的id_rsa.pub添加到远程机器的authorized_keys中，cat id_rsa.pub >> authorized_keys
