---
layout: posts
title: clonezilla使用案例
date: 2025-10-12 11:32:10
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "装机"
tags:
- "ventoy"
- "clonezilla"

---



clonezilla 是个好东西，可以克隆硬盘、制作镜像等等。



# 克隆硬盘步骤（简单版）

1、我用一个 64g的u盘制作了一个 ventoy 工具盘，空余空间为12g，clonezilla的ISO文件是放在 这个u盘中的。开机选择 这个u盘启动，注意格式。

![111](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B/111.jpg)



2、选择 语言



3、选择使用“重生龙”



4、选择 “从硬盘/分区克隆到硬盘/分区”



5、初级模式



6、是否保留原分区形式，选是。



7、y...y...y...y y 一路确认。



8、选择 克隆完成以后 reboot 还是 poweroff，我一般选了reboot。



9、我一个nvme 的512g的硬盘 克隆到 另一个nvme的512g硬盘中，ventoy的U盘64g，12g空余，一共耗时4小时。



# 一个失败案例

![aaaebb506391433eaca83d403765733](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B/aaaebb506391433eaca83d403765733.jpg)

如果出现 上面这样的显示，说明不仅仅是 分辨率的问题，很有可能是对u盘中的 clonezilla 读取出现啦一些问题。我在遇到这样的显示之后，克隆的过程中遇到了 下面的报错。

## 报错

克隆的过程中 速度非常非常慢。预计要30小时，我就发现了又问题，然后在克隆5%时候报出如下错误：

```sh
报错 /usr/share/brb1/sbin/ocs-functions: line 15141: 41591 killed.  partclone.dd -z 10485760 -N -L /var/log/partclone.log -s /dev/sdc3 -0 /dev/sdd3  Failed to clone /dev/sdc3 to /dev/sdd3 press "Enter" to continue... 
```

## 解决

这样的报错 可能是 U盘读写性能出现了问题，也可能是 两块硬盘中出现了坏块等等问题，我很难锁定具体问题。

于是我换了一台电脑，在这台电脑上插上 ventoy的U盘和 源、目的两个硬盘，重新操作了一遍，这次没有报错。

![fd0cc1352fd4e95595041bf384d48ce](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B/fd0cc1352fd4e95595041bf384d48ce.jpg)

预计4小时完成，2G/min 克隆速度 属于正常的水平。于是成功克隆。



# 网络批量克隆硬盘案例

我拿一个交换机和 两根网线，连接好  “一台待客隆的源机器” 和 “一个目标空白机器”。

## 服务端

将有Clonezilla的 Ventoy U盘 插到源机器上选择Clonezilla，作为服务端。

![截屏2025-10-12 20.12.16](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.12.16.jpg)



![截屏2025-10-12 20.14.02](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.14.02.jpg)



![截屏2025-10-12 20.14.23](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.14.23.jpg)



![截屏2025-10-12 20.14.43](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.14.43.jpg)



![截屏2025-10-12 20.15.08](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.15.08.jpg)



![截屏2025-10-12 20.15.28](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.15.28.jpg)



![截屏2025-10-12 20.15.48](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.15.48.jpg)

选择新增dhcp服务

![截屏2025-10-12 20.16.55](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.16.55.jpg)



![截屏2025-10-12 20.17.58](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.17.58.jpg)



![截屏2025-10-12 20.18.19](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.18.19.jpg)



![截屏2025-10-12 20.18.52](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.18.52.jpg)



![截屏2025-10-12 20.19.14](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.19.14.jpg)



![截屏2025-10-12 20.20.11](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.20.11.jpg)



![截屏2025-10-12 20.20.59](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.20.59.jpg)



![截屏2025-10-12 20.21.25](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.21.25.jpg)



![截屏2025-10-12 20.22.12](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.22.12.jpg)



![截屏2025-10-12 20.22.57](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.22.57.jpg)



![截屏2025-10-12 20.23.12](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.23.12.jpg)



![截屏2025-10-12 20.23.40](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.23.40.jpg)



![截屏2025-10-12 20.24.21](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.24.21.jpg)

这里注意！ 等客户端那边操作完毕了以后，才能继续！中途打断前功尽弃。

![截屏2025-10-12 20.25.10](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.25.10.jpg)



## 客户端

![截屏2025-10-12 20.28.59](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.28.59.jpg)



![截屏2025-10-12 20.29.54](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.29.54.jpg)



![截屏2025-10-12 20.30.40](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.30.40.jpg)



![截屏2025-10-12 20.31.13](clonezilla%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B.assets/%E6%88%AA%E5%B1%8F2025-10-12%2020.31.13.jpg)



完毕之后，回到服务端 按一下 y继续。

end