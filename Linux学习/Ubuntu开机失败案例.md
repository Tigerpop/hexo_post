---
layout: posts
title: Ubuntu开机失败案例
date: 2026-03-02 20:20:33
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "Linux"
tags:
- "fsck"
- "Ubuntu"

---



# 案例一：

## 问题：

Ubuntu虚拟机开机一直无法开起来，打开物理机的控制台，在里面看见Ubuntu的提示如下：

```bash
Loading essential drivers ... done.

[    4.802477] raid6: avx512x4   gen() 30553 MB/s
[    4.850476] raid6: avx512x4   xor() 14110 MB/s
[    4.898476] raid6: avx512x2   gen() 28877 MB/s
[    4.946476] raid6: avx512x2   xor() 17117 MB/s
[    4.994477] raid6: avx512x1   gen() 22663 MB/s
[    5.042476] raid6: avx512x1   xor() 12002 MB/s
[    5.090475] raid6: avx2x4     gen() 21117 MB/s
[    5.138476] raid6: avx2x4     xor() 12822 MB/s
...
[    5.618744] raid6: using algorithm avx512x4 gen() 30553 MB/s
[    5.619003] raid6: using avx512x2 recovery algorithm
[    5.621024] xor: automatically using best checksumming function   avx
[    5.623853] async_tx: api initialized (async)

done.
Begin: Running /scripts/init-premount ... done.
Begin: Mounting root file system ...
Begin: Running /scripts/local-top ... done.
Begin: Running /scripts/local-premount ...

[    5.738951] Btrfs loaded, crc32c=crc32c-intel
Scanning for Btrfs filesystems

[    5.858492] blk_update_request: I/O error, dev fd0, sector 0 op 0x0:(READ)
                flags 0x0 phys_seg 1 prio class 0
[    5.858870] floppy: error 10 while reading block 0

done.
Begin: Will now check root file system ...
fsck from util-linux 2.34

[/usr/sbin/fsck.ext4 (1) -- /dev/mapper/ubuntu--vg-ubuntu--lv]
fsck.ext4 -a -C0 /dev/mapper/ubuntu--vg-ubuntu--lv

Inode 3801089 seems to contain garbage.

/dev/mapper/ubuntu--vg-ubuntu--lv:
UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.
(i.e., without -a or -p options)

fsck exited with status code 4
done.

Failure: File system check of the root filesystem failed
The root filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv requires a manual fsck

BusyBox v1.30.1 (Ubuntu 1:1.30.1-4ubuntu6.5) built-in shell (ash)
Enter 'help' for a list of built-in commands.

(initramfs)_
```

**这一般是异常断电导致的**，如果是 raid 或者磁盘损坏导致的话就不太好处理。



## 解决：

关注提示中的关键位置

```bash
UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.
Inode ... seems to contain garbage.
The root filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv requires a manual fsck.
BusyBox (initramfs)
```

这个不是“挂载路径配置错误”，而是：

**根文件系统损坏（ext4 文件系统不一致）**
 需要手动执行 `fsck` 修复。

而且它已经掉进了 **initramfs 紧急模式**。

在紧急模式中使用下面的方法处理：

```bash
fsck -f -y /dev/mapper/ubuntu--vg-ubuntu--lv
# -f 强制检查
# -y 自动修复
# 注意：
# - 是 `ubuntu--vg-ubuntu--lv` 横杆数量别打错了。
# 如果提示： ***** FILE SYSTEM WAS MODIFIED *****
# ✔ 文件系统已经成功修复
# ✔ ext4 结构已恢复
# ✔ 数据基本没丢
# ✔ 不是底层磁盘损坏
# 现在只是 initramfs 里 reboot 不生效 —— 这是正常现象。


# 退出initramfs，正常启动；
exit  # 如果成功，系统会：重新尝试挂载根分区，然后继续启动流程
```

可能遇到的情况以及处理步骤：

> 一、确认`ubuntu--vg-ubuntu--lv`的名字是否正确，
>
> ```
> ls -l /dev/mapper/   # 看看实际生成的名字是什么
> ls /dev/mapper
> ```
>
> 来 修复
>
> ```
> fsck.ext4 -f -y /dev/mapper/实际名字
> # 或者用 LVM 原始路径：例如实际生成的名字`ubuntu--vg-ubuntu--lv`就是如下：
> fsck.ext4 -f -y /dev/ubuntu-vg/ubuntu-lv
> ```



> 二、如果出现了提示  `fsck.ext2: No such file or directory`，而我的根分区实际是 **ext4**，就需要在 `initramfs` 里，直接用：`fsck.ext4 -f -y /dev/mapper/ubuntu--vg-ubuntu--lv`
>
> 注意：
>
> - 是 `fsck.ext4`
>
>   先看看系统里有哪些：
>
>   ```
>   ls /sbin/fsck*
>   ```
>
>   如果看到：
>
>   ```
>   /sbin/fsck.ext4
>   ```
>
>   那就直接运行：
>
>   ```
>   /sbin/fsck.ext4 -f -y /dev/mapper/ubuntu--vg-ubuntu--lv
>   ```

> 如果提示设备不存在
>
> `fsck: error 2 (No such file or directory)
> fsck.ext4: No such file or directory while trying to open /dev/mapper/ubuntu--vg-ubuntu--lv
> Possibly non-existent device?`
>
> 在 initramfs 里执行：
>
> ```
> # 看看 LVM 在不在
> lvm pvscan
> lvm vgscan        # Found volume group "ubuntu-vg"说明物理卷还在。
> 
> # 激活卷组
> lvm vgchange -ay  # 如果成功会看见 logical volume(s) in volume group "ubuntu-vg" now active
> 
> # 确认逻辑卷出现
> ls /dev/mapper/   # 一般能看见 ubuntu--vg-ubuntu--lv
> ```
>
> 然后再执行：
>
> ```
> fsck.ext4 -f -y /dev/mapper/ubuntu--vg-ubuntu--lv
> ```



