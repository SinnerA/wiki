---
title: Linux常用命令
date: 2017-05-12
---

[TOC]

## Linux常用命令

### su(switch user)

- 不加参数，默认切换到root，但此时并没有改变root的登录环境
- su -：表示切换到root并改变到root的环境（可用pwd验证）
- su 参数 - 用户名

### sudo(super user do)

sudo 参数  [-u username│#uid ] command：使用username权限执行命令，默认使用root权限执行

由于su的使用，需要用户知道root密码，普通用户具有root权限是比较危险的。

最好是针对每个管理员的技术特长和管理范围，并且有针对性的下放给权限，并且约定其使用哪些工具来完成与其相关的工作，这时我们就有必要用到 sudo

sudo跟su最大的区别是，普通用户不知道root密码，root授权给某个用户，该用户才能行使root权限，保证了系统安全，因此也被称为授权许可的su

sudo 执行命令的流程是当前用户切换到root（或其它指定切换到的用户），然后以root（或其它指定的切换到的用户）身份执行命令，执行完成后，直接退回到当前用户；而这些的前提是要通过sudo的配置文件/etc/sudoers来进行授权（sudo -su search）

### pwd

当前路径

### id

当前用户信息

### chmod

Read (r)=4  Write(w)=2  Execute(x)=1

chmod 666 file：file对于所有人具有读写权限

sudo chmod (u所属用户 g所属组 o其他用户 a所有用户)  (+增加权限 -减少权限) (r w x) 目录名

例如：有一个文件filename，权限为“-rw-r----x” ,将权限值改为"-rwxrw-r-x"，用数值表示为765

sudo chmod u+x g+w o+r  filename

上面的例子可以用数值表示

sudo chmod 765 filename

### tar(Tap Archive，磁带归档)

- 压缩：tar -czvf    dest/../123.tar.gz    src/../file （c:create，z:zip，v:显示文件，f:使用文档名，之后立即接文档名）
- 解压：tar -xzvf   src/../123.tar.gz    dest/../file（x:解压，z:zip，v:显示文件，f:使用文档名，之后立即接文档名）

### cat(concatenation)

连接两个或者更多文本文件或者以标准输出形式打印文件的内容

### diff

- diff -y files（并排比较显示）


### du

du命令用来查看目录或文件所占用磁盘空间的大小，常用选项组合为：du -sh

### dd

用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。

ps: 备份磁盘开始的512个字节大小的MBR信息到指定文件

dd if=/dev/hda of=/root/image count=1 bs=512

count=1指仅拷贝一个块；bs=512指块大小为512个字节。

