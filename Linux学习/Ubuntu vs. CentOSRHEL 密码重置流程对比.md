---
layout: posts
title: Ubuntu vs. CentOS\RHEL 密码重置流程对比
date: 2025-9-13 23:28:18
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "Linux"
tags:
- "centos"
- "ubuntu"
- "密码"
---







### Ubuntu vs. CentOS/RHEL 密码重置

流程对比

| 步骤              | CentOS / RHEL                           | Ubuntu（推荐方法）                                           | 作用说明                                                     |
| ----------------- | --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **1. 进入 GRUB**  | Esc 按 **e** 编辑启动参数               | Esc 按 **e** 编辑启动参数（如果是多系统，按esc可能要经历过一次bios退出，然后在显示ubuntu、advanced options for ubuntu 这个界面的那几秒中时间，把光标上下移动到ubuntu上，不要按回车，直接按e，才能进入 GRUB 界面。） | 打开启动菜单，进入内核参数编辑模式                           |
| **2. 修改参数**   | 在 `linux16` 行尾加 `rw init=/bin/bash` | 在 `linux` 行尾替换 `quiet splash` 为 `rw init=/bin/bash`                             （注意：这里的显示不一定是完整的，可能只能看见 ro ，其实就是 ro quiet splash ，一样删除按照上面的改就好。改完按F10保存退出自己就会进入一个有编辑权限的命令界面。） | 让系统以 **单用户模式 + 可写根目录** 启动，进入 bash         |
| **3. 挂载根目录** | `mount -o remount,rw /`                 | `mount -o remount,rw /`（通常已可写，但最好执行）            | 确保根目录有写权限，否则不能改密码                           |
| **4. 核心操作**   | `passwd`（直接改 root 密码）            | `passwd 用户名`（修改普通用户密码）                          | 修改目标账户的密码（CentOS 默认 root 可用；Ubuntu 默认 root 禁用） |
| **5. SELinux**    | `touch /.autorelabel`（必须）           | 不需要（Ubuntu 默认无 SELinux）                              | 触发系统重新标记文件安全上下文，避免登录失败                 |
| **6. 重启**       | `exec /sbin/init` 或 `reboot -f`        | `exec /sbin/init` 或 `reboot -f`                             | 退出恢复模式，重新正常启动系统                               |