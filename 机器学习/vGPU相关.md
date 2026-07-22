---
layout: posts
title: vGPU相关
date: 2025-11-19 15:53:22
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "vGPU"
- "华三"
- "Nvidia授权"
- "DLS"

---



# vGPU基本逻辑介绍

## 核心角色

- **License（许可证）：** 就是您**购买的咖啡券**。这张券规定了您可以制作多少杯（对应多少个vGPU）、什么口味的咖啡（对应什么等级的vGPU型号），以及有效期。
- **NVIDIA 官方授权中心：** 就像**咖啡券的总发行商**（比如星巴克总部）。它不直接对顾客，而是向各个分店提供授权。
- **DLS (Demonstrated License Server) / CLS (Container License Server)：** 这是**两种不同类型的分店**。**DLS（大型中心店）：** 是一个功能完整、可以组建集群（高可用HA）的独立门店。它规模大、稳定，可以为整个区域（多个客户/租户）提供服务。文中的“配置DLS服务实例”就是在搭建这样一家大型中心店。**CLS（快捷迷你店）：** 是一个轻量级的、嵌入在商场里的咖啡柜台。它更简单、部署快，但通常只为当前这个商场（单个客户/租户）服务。文中的“配置CLS服务实例”就是设立这样一个迷你柜台。
- **License Server（许可证服务器）：** 就是这家**分店里的收银机和兑券系统**。它是具体执行“验证咖啡券并出杯”的机器。您需要在DLS或CLS这家“店”里面，安装这台“收银机”。
- **服务实例令牌（DLS/CLS Token）：** 这是**分店的营业执照和总部授权书**。当您想开一家分店（DLS或CLS）时，需要向总部（NVIDIA授权中心）注册，总部会颁发给您这个“令牌”，证明这家店是合法的，可以连接总部并兑换咖啡券。
- **客户端配置令牌（Client Config Token）：** 这是**告诉顾客“去哪家分店兑券”的小纸条**。当您的虚拟桌面（顾客）启动时，它需要知道应该去哪个License Server（哪家分店）验证许可证。您需要把这个“小纸条”（客户端配置令牌）提前放在虚拟桌面镜像里。



# vGPU 找密码

Nvidia 的 vGPU授权的虚拟机，管理web网页的密码不记得了，怎么办？

## 问题：

在http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】先找出来 【VGPU_LICENSE_SERVER】是哪一个IP地址。

然后在网页中打开 `172.26.0.204` 进入Nvidia的vGPU授权的管理页面。

发现密码 记不住了，怎么办？

## 解决方法：

DLS管理员用户名默认为dls_admin；

**借助** **NVIDIA** **企业门户下载的密钥重置**

1. 1. 登录[NVIDIA 企业门户](https://nvid.nvidia.com/)并进入授权门户，点击左侧的**SERVICE INSTANCES**，找到该 DLS 对应的实例，点击实例右侧的**Action**按钮，选择**Download Reset Secret**下载重置密钥文件。
   2. 在浏览器中输入 DLS 服务器的 IP 地址进入后台登录页面，点击页面中的**Forgot password****？** 选项。
   3. 点击页面中的**SELECT DLS LOCAL RESET SECRET**，选中上一步下载的密钥文件，随后输入两次新密码，点击**RESET PASSWORD**就能完成密码重置。

注意： Nvidia的这个密码重置建议设置一个全角输入法的密码，这样在重置密码以后才能方便用全角间距的密码登录；（不支持半角密码输入，全角类似这样`ａｂｃｄｅｆｇ１２３４５６７８`）


# License Pools许可证池未成功分配

## 问题：

`172.26.0.204` 进入Nvidia的vGPU授权的DLS管理页面。

进入 【dashboard 仪表盘】 的 【License Pools 许可证池】，打开下拉框，看见 【in use / allocated 使用/已经分配】下8个已分配 下面只有4个是in use。

同时，我们进入每一个使用GPU的 虚拟机中，用 命令查询 `nvidia-smi -q | grep License` 可见没有成功授权

```bash
(base) jlsf@admin:~$ nvidia-smi -q | grep -i 'License Status'
        License Status                    : Unlicensed
```

## 解决办法：  

### 先找出问题的原因：

我们从`172.26.0.204` 进入Nvidia的vGPU授权的管理页面，进入 【dashboard 仪表盘】 的 【server features 服务器功能】，打开下拉框，看见 `NVIDIA RTX Virtual Workstation-5.0`,这说明，我们是买了一个 Nvidia的 `VWS 授权`，而 `VWS 授权` 只能对应 `Q/B` 授权；

我们再从 http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【任意一台物理机】的 【GPU设备】可见

```
GPU设备
...

vGPU
... GRID V100S-16C ... 
```

我们看见了 GRID V100S-16C 而不是 GRID V100S-16Q或者 GRID V100S-16B，不符合 `VWS 授权` 只能对应 `Q/B` 授权的 匹配性；

### 措施：重新切分Q类授权

前期准备：

看【H3C Workspace云桌面 vGPU配置指导(办公场景)-5W116-整本手册(1)】把配套的生成token和授权的文件这些 先准备好。

然后（顺序可能有一些变化）：

1. 在每个使用 vGPU 的虚拟机ssh登录进去，检查有无 文件 `/etc/nvidia/ClientConfigToken/XXX.tok` 和 `/etc/nvidia/gridd.conf` 两个文件，改一下`/etc/nvidia/ClientConfigToken`权限，最好是按照【H3C Workspace云桌面 vGPU配置指导(办公场景)-5W116-整本手册(1)】文档搞一个新的 tok 下来；

2.  http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【虚拟机】关机删除GPU；

3.  http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【概览】【智能资源调度】删除vGPU；

4.  http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【物理机】【GPU设备】回收vGPU；

5. 重新启动  http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【两个授权的虚拟机】；

6.  http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【概览】【智能资源调度】增加智能资源调度；

7. http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【物理机】【GPU设备】按照Q类切分GPU，【vGPU】可见切出来的vGPU资源，以Q结尾命名方便区分，类似`GRID V100S-16Q`；

8. 切换平台到 http://172.26.0.202:8803 的云桌面配置界面，给每个虚拟机加上 vGPU 资源；

9.  http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【虚拟机】开机，有的可能需要进入虚拟机看看Nvidia企业版驱动是不是要被重新安装；

10. 进入 https://172.26.0.204/ 进入Nvidia的DLS的管理页面，![微信图片_20251119152815_6415_17](vGPU%E7%9B%B8%E5%85%B3/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20251119152815_6415_17.png)

    如果需要 手动释放(针对异常占用或批量释放场景)：

    当自动释放失效，或需主动回收特定 leases 时，可通过 DLS 服务器管理页面与 NVIDIA 许可门户操作，步骤如下:

    ```markdown
    1.登录本地 DLS 的“License Sever Details"页面，找到右上角“ACTIONS"下拉菜单，点击“Disable”禁用服务器，这是修改许可配置的前置操作。
    2.再次点击"ACTIONS"，选择"Manage Features"，在该页面中找到需要释放的许可类型，调整对应 leases的数量(比如将占用的许可数量下调至目标值)
    3.完成设置后，进入 DLS 页面的“MAINTENANCE”菜单，选择“Export feature returns”，下载用于还原许可的 BIN 文件。
    4.打开NVIDlA Licensing Portal (https://nvid.nvidia.com/dashboard/#/dashboard )找到对应的许可服务器，在其“ACTIONS"选项中选择“Return Features”，上传刚才下载的 BIN 文件。
    5.最后返回本地 DLS 页面启用服务器，此时释放的 leases 会回归许可池，可在“Server Features” 中查看更新后的许可数量。
    ```

    **但是我的实际操作是：**

    ```markdown
    1.登录本地 DLS 的“License Sever Details"页面，找到右上角“ACTIONS"下拉菜单，点击“Disable”禁用服务器，这是修改许可配置的前置操作。
    2.登录本地 DLS 的“License Sever Details"页面，找到 "Lease" 点击任意ID的Release即可；
    ```

    **需要注意的是：**

    在虚拟机重启以后，要在这个 DLS 界面中 让它能够识别到新的授权虚拟机，按如下操作：

    ```markdown
    0.http://172.26.0.202:8083 的云桌面管理界面，【数据中心】【虚拟化】【概览】【智能资源调度】修改智能资源调度，可以在虚拟机都关机的情况下，删了重新建vGPU的智能资源调度；
    1.登录本地 DLS 的“License Sever Details"页面，找到右上角“ACTIONS"下拉菜单，点击“Enable”启用服务器，这是识别到新的授权虚拟机的前置操作。
    2.然后反复操作：
    （ 关闭要被授权的虚拟机；
       在 http://172.26.0.202:8803 卸载这台虚拟机的vGPU；
       开机要被授权的虚拟机；
       关闭要被授权的虚拟机；
       在 http://172.26.0.202:8803 添加这台虚拟机的vGPU；
       开机要被授权的虚拟机；
     ）
    ```

    直到我们看见 Leases 下的虚拟机都关联上了，同时 License Pools 中 in use / allocated 都成功了，就ok了。

11. 在每个使用 vGPU 的虚拟机ssh登录进去，重启授权服务 `sudo service nvidia-gridd restart`；

12. 检查好了没有 `nvidia-smi -q | grep License`；

    ```bash
    (base) jlsf@admin:/etc/nvidia$ nvidia-smi -q | grep License
        vGPU Software Licensed Product
            License Status                    : Licensed (Expiry: 2025-11-19 6:36:54 GMT)
    ```

    这样显示就是 授权恢复了。

