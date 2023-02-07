---
title: "linux常用命令"
date: 2023-02-07T16:52:58+08:00
draft: false
description: "linux常用命令"
tags: ["linux"]
categories: ["linux"]
---

1. 常用命令

 (1) 安全类型

 ① sudo 使用root 用户来执行命令

 ② su 使用指定用户来执行命令

 ③ chmod 修改文件权限

 ④ setfacl 修改文件权限，设置文件访问列表

 (2) 进程管理

 ① w 显示已经登陆用户列表

 ② top 	电脑性能分析工具

 ③ ps 显示系统进程

 ④ kill  发送信号杀死进程

 ⑤ pkill	 根据进程名称杀死进程

 ⑥ pstree 	显示进程树木

 ⑦ killall 指定名称杀死所有进程

 (3) 用户管理

 ① id 显示当前用户ID以及分组的ID

 ② usermod 	修改账号信息包括分组权限

 ③ useradd	增加用户

 ④ groupadd	 增加分组

 ⑤ userdel 删除用户

 (4) 文件系统

 ① mount 挂载文件系统

 ② umount 取消挂载

 ③ fsck 	文件修复

 ④ df 查看磁盘信息

 ⑤ du	显示目录占用空间

 (5) 网络应用

 ① curl	 网络请求命令，用于和服务器进行交互

 ② telnet	远程登陆命令

 ③ mail 		邮件命令

 ④ elinks 以文本方式访问网站

 (6) 关机和重启

 ① shutdown 关机

 ② reboot 重启

 (7) 网络测试

 ① ping 检测主机网络传输

 ② netstat 查看端口信息

 ③ host DNS查询

 (8) 网络配置

 ① hostname 显示或者修改主机名称

 ② ifconfig 修改网络配置（开放端口）

 (9) 常用工具

 ① ssh 远程登陆命令

 ② screen 令用于多重视窗管理程序

 ③ clear 清空终端显示信息

 ④ who 当前用户信息

 ⑤ date  时间操作命令

 (10) 软件包管理

 ① yum

 ② rpm

 ③ ap t-get

 (11) 文件查找和比较

 ① locate

 ② find

 (12) 文件内容查看

 ① head 查看头部

 ② tail 查看正在改变的内容

 ③ less 分页

 ④ more 分页查看向后

 (13) 文件处理

 ① touch 创建文件

 ② unlink 删除文件

 ③ rename 重命名

 ④ ln 设置软连接

 ⑤ cat 文件截切连接

 (14) 目录

 ① cd 跳转目录

 ② mv 移动文件

 ③ rm  重命名

 ④ pwd 当前目录

 ⑤ tree 文件树

 ⑥ cp 复制

 ⑦ ls 显示目录文件

 (15) 定时任务

 ① crontab -e ****** 命令（分时日月周）

 ② #at 2:00 tomorrow 明天两点执行
