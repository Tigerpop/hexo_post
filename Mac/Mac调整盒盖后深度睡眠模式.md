---
layout: posts
title: Mac调整盒盖后深度睡眠模式
date: 2025-9-23 12:08:32
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "Mac"
tags:
- "Mac"
---





`sleepmode.sh`

```bash
#!/bin/bash

if [ "$1" == "fast" ]; then
    echo "切换到 快唤醒模式（hibernatemode=3，8小时后才进入深度睡眠，保留唤醒动画）..."
    sudo pmset -a hibernatemode 3           # 普通睡眠+RAM供电，快速唤醒
    sudo pmset -a standby 1                 # 允许深度睡眠
    sudo pmset -a standbydelayhigh 28800    # 高电量延迟 8 小时
    sudo pmset -a standbydelaylow 14400     # 低电量延迟 4 小时
    sudo pmset -a highstandbythreshold 50   # 电量 >50% 用高电量延迟
    sudo pmset -a womp 0                    # 关闭网络唤醒
    sudo pmset -a powernap 0                # 关闭 Power Nap，减少后台任务
    sudo pmset -a proximitywake 1           # 开启接近唤醒，保留唤醒动画
    echo "✅ 已切换为快唤醒模式（8小时后才进入深度睡眠，低电量4小时才进入深度睡眠）"
elif [ "$1" == "save" ]; then
    echo "切换到 省电深度睡眠模式（hibernatemode=25）..."
    sudo pmset -a hibernatemode 25
    sudo pmset -a standby 0
    sudo pmset -a autopoweroff 0
    sudo pmset -a womp 0
    sudo pmset -a powernap 0
    sudo pmset -a proximitywake 0
    echo "✅ 已切换为省电深度睡眠模式"

elif [ "$1" == "restore" ]; then
    echo "恢复到 macOS 出厂默认盒盖睡眠设置..."
    sudo pmset -a hibernatemode 3
    sudo pmset -a standby 1
    sudo pmset -a standbydelayhigh 86400
    sudo pmset -a standbydelaylow 10800
    sudo pmset -a highstandbythreshold 50
    sudo pmset -a autopoweroff 1
    sudo pmset -a autopoweroffdelay 28800
    sudo pmset -a womp 1
    sudo pmset -a powernap 1
    sudo pmset -a proximitywake 1
    echo "✅ 已恢复为 macOS 默认设置"

else
    echo "用法: "
    echo "  ./sleepmode.sh fast     # 快唤醒模式"
    echo "  ./sleepmode.sh save     # 省电深度睡眠模式"
    echo "  ./sleepmode.sh restore  # 恢复 macOS 默认合盖睡眠设置"
fi
```

