---
layout: posts
title: 断电掉ISCS共享存储解决办法
date: 2025-02-15 22:25:10
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "装机"
tags:
- "共享存储"
- "iscs"

---



# 背景：

​	有两台物理机，每台物理机器内 有4个虚拟机，一共4张GPU卡，华三的 云服务部署在 共享存储中，可以通过  云管理平台来 实现对这8台虚拟机的控制，共享存储用的是 iscs协议连接。

老的 虚拟机的共享存储，在断电情况下，突发了物理机器连不上 共享存储的情况。

```sh
# 脱机的是 
iscsiadm -m session
iscsiadm: No active sessions.

# 正常的是 
iscsiadm -m session
tcp: [1] 172.26.1.207:3260,6 iqn.2004-12.com.inspur:mcs.inspurstore.node2 (non-flash)
tcp: [2] 172.26.1.206:3260,6 iqn.2004-12.com.inspur:mcs.inspurstore.node1 (non-flash)
```

在云管理平台上可见，其中一台物理机器的iscs连接脱机了。我并不清楚发生了什么改变。

![image-20250129165912465](%E6%96%AD%E7%94%B5%E6%8E%89ISCS%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/image-20250129165912465.png)



# 我的处理回顾：

## 1、iSCSIconfig文件换了。

```sh
systemctl status iscsid

# 列出已发现的目标：
iscsiadm -m discovery -t sendtargets -p <iSCSI 目标 IP>

# 查看回话：
iscsiadm -m session

#  iSCSI 配置文件
cat /etc/iscsi/iscsid.conf

# 检查iscs报错
journalctl -xe | grep iscsi

# 确认防火墙没有阻止 iSCSI 端口（默认 3260） 
自己搞吧

# 删除现有的 iSCSI 配置
iscsiadm -m node -o delete -T iqn.2004-12.com.inspur:mcs.inspurstore.node1 -p 172.26.1.206
iscsiadm -m node -o delete -T iqn.2004-12.com.inspur:mcs.inspurstore.node2 -p 172.26.1.207

# 删除后再执行目标发现命令
iscsiadm -m discovery -t sendtargets -p 172.26.1.206
iscsiadm -m discovery -t sendtargets -p 172.26.1.207

# sudo iscsiadm -m discovery -t st -p <iSCSI 存储 IP 地址>
sudo iscsiadm -m discovery -t st -p 172.26.1.206
sudo iscsiadm -m discovery -t st -p 172.26.1.207
sudo iscsiadm -m node --login
```

换 iscsconfig 文件 

```sh
# Open-iSCSI default configuration.
# Could be located at /etc/iscsi/iscsid.conf or ~/.iscsid.conf
#
# Note: To set any of these values for a specific node/session run
# the iscsiadm --mode node --op command for the value. See the README
# and man page for iscsiadm for details on the --op command.
#

######################
# iscsid daemon config
######################
# If you want iscsid to start the first time a iscsi tool
# needs to access it, instead of starting it when the init
# scripts run, set the iscsid startup command here. This
# should normally only need to be done by distro package
# maintainers.
#
# Default for Fedora and RHEL. (uncomment to activate).
# iscsid.startup = /etc/rc.d/init.d/iscsid force-start
# 
# Default for upstream open-iscsi scripts (uncomment to activate).
iscsid.startup = /usr/sbin/iscsid


#############################
# NIC/HBA and driver settings
#############################
# open-iscsi can create a session and bind it to a NIC/HBA.
# To set this up see the example iface config file.

#*****************
# Startup settings
#*****************

# To request that the iscsi initd scripts startup a session set to "automatic".
# node.startup = automatic
#
# To manually startup the session set to "manual". The default is manual.
node.startup = manual

# For "automatic" startup nodes, setting this to "Yes" will try logins on each
# available iface until one succeeds, and then stop.  The default "No" will try
# logins on all availble ifaces simultaneously.
node.leading_login = No

# *************
# CHAP Settings
# *************

# To enable CHAP authentication set node.session.auth.authmethod
# to CHAP. The default is None.
#node.session.auth.authmethod = CHAP

# To set a CHAP username and password for initiator
# authentication by the target(s), uncomment the following lines:
#node.session.auth.username = username
#node.session.auth.password = password

# To set a CHAP username and password for target(s)
# authentication by the initiator, uncomment the following lines:
#node.session.auth.username_in = username_in
#node.session.auth.password_in = password_in

# To enable CHAP authentication for a discovery session to the target
# set discovery.sendtargets.auth.authmethod to CHAP. The default is None.
#discovery.sendtargets.auth.authmethod = CHAP

# To set a discovery session CHAP username and password for the initiator
# authentication by the target(s), uncomment the following lines:
#discovery.sendtargets.auth.username = username
#discovery.sendtargets.auth.password = password

# To set a discovery session CHAP username and password for target(s)
# authentication by the initiator, uncomment the following lines:
#discovery.sendtargets.auth.username_in = username_in
#discovery.sendtargets.auth.password_in = password_in

# ********
# Timeouts
# ********
#
# See the iSCSI REAME's Advanced Configuration section for tips
# on setting timeouts when using multipath or doing root over iSCSI.
#
# To specify the length of time to wait for session re-establishment
# before failing SCSI commands back to the application when running
# the Linux SCSI Layer error handler, edit the line.
# The value is in seconds and the default is 120 seconds.
# Special values:
# - If the value is 0, IO will be failed immediately.
# - If the value is less than 0, IO will remain queued until the session
# is logged back in, or until the user runs the logout command.
node.session.timeo.replacement_timeout = 17

# To specify the time to wait for login to complete, edit the line.
# The value is in seconds and the default is 15 seconds.
node.conn[0].timeo.login_timeout = 15

# To specify the time to wait for logout to complete, edit the line.
# The value is in seconds and the default is 15 seconds.
node.conn[0].timeo.logout_timeout = 15

# Time interval to wait for on connection before sending a ping.
node.conn[0].timeo.noop_out_interval = 3

# To specify the time to wait for a Nop-out response before failing
# the connection, edit this line. Failing the connection will
# cause IO to be failed back to the SCSI layer. If using dm-multipath
# this will cause the IO to be failed to the multipath layer.
node.conn[0].timeo.noop_out_timeout = 5

# To specify the time to wait for abort response before
# failing the operation and trying a logical unit reset edit the line.
# The value is in seconds and the default is 15 seconds.
node.session.err_timeo.abort_timeout = 15

# To specify the time to wait for a logical unit response
# before failing the operation and trying session re-establishment
# edit the line.
# The value is in seconds and the default is 30 seconds.
node.session.err_timeo.lu_reset_timeout = 30

# To specify the time to wait for a target response
# before failing the operation and trying session re-establishment
# edit the line.
# The value is in seconds and the default is 30 seconds.
node.session.err_timeo.tgt_reset_timeout = 30


#******
# Retry
#******

# To specify the number of times iscsid should retry a login
# if the login attempt fails due to the node.conn[0].timeo.login_timeout
# expiring modify the following line. Note that if the login fails
# quickly (before node.conn[0].timeo.login_timeout fires) because the network
# layer or the target returns an error, iscsid may retry the login more than
# node.session.initial_login_retry_max times.
#
# This retry count along with node.conn[0].timeo.login_timeout
# determines the maximum amount of time iscsid will try to
# establish the initial login. node.session.initial_login_retry_max is
# multiplied by the node.conn[0].timeo.login_timeout to determine the
# maximum amount.
#
# The default node.session.initial_login_retry_max is 8 and
# node.conn[0].timeo.login_timeout is 15 so we have:
#
# node.conn[0].timeo.login_timeout * node.session.initial_login_retry_max =
#								120 seconds
#
# Valid values are any integer value. This only
# affects the initial login. Setting it to a high value can slow
# down the iscsi service startup. Setting it to a low value can
# cause a session to not get logged into, if there are distuptions
# during startup or if the network is not ready at that time.
node.session.initial_login_retry_max = 8

################################
# session and device queue depth
################################

# To control how many commands the session will queue set
# node.session.cmds_max to an integer between 2 and 2048 that is also
# a power of 2. The default is 128.
node.session.cmds_max = 128

# To control the device's queue depth set node.session.queue_depth
# to a value between 1 and 1024. The default is 32.
node.session.queue_depth = 32

##################################
# MISC SYSTEM PERFORMANCE SETTINGS
##################################

# For software iscsi (iscsi_tcp) and iser (ib_iser) each session
# has a thread used to transmit or queue data to the hardware. For
# cxgb3i you will get a thread per host.
#
# Setting the thread's priority to a lower value can lead to higher throughput
# and lower latencies. The lowest value is -20. Setting the priority to
# a higher value, can lead to reduced IO performance, but if you are seeing
# the iscsi or scsi threads dominate the use of the CPU then you may want
# to set this value higher.
#
# Note: For cxgb3i you must set all sessions to the same value, or the
# behavior is not defined.
#
# The default value is -20. The setting must be between -20 and 20.
node.session.xmit_thread_priority = -20


#***************
# iSCSI settings
#***************

# To enable R2T flow control (i.e., the initiator must wait for an R2T
# command before sending any data), uncomment the following line:
#
#node.session.iscsi.InitialR2T = Yes
#
# To disable R2T flow control (i.e., the initiator has an implied
# initial R2T of "FirstBurstLength" at offset 0), uncomment the following line:
#
# The defaults is No.
node.session.iscsi.InitialR2T = No

#
# To disable immediate data (i.e., the initiator does not send
# unsolicited data with the iSCSI command PDU), uncomment the following line:
#
#node.session.iscsi.ImmediateData = No
#
# To enable immediate data (i.e., the initiator sends unsolicited data
# with the iSCSI command packet), uncomment the following line:
#
# The default is Yes
node.session.iscsi.ImmediateData = Yes

# To specify the maximum number of unsolicited data bytes the initiator
# can send in an iSCSI PDU to a target, edit the following line.
#
# The value is the number of bytes in the range of 512 to (2^24-1) and
# the default is 262144
node.session.iscsi.FirstBurstLength = 262144

# To specify the maximum SCSI payload that the initiator will negotiate
# with the target for, edit the following line.
#
# The value is the number of bytes in the range of 512 to (2^24-1) and
# the defauls it 16776192
node.session.iscsi.MaxBurstLength = 16776192

# To specify the maximum number of data bytes the initiator can receive
# in an iSCSI PDU from a target, edit the following line.
#
# The value is the number of bytes in the range of 512 to (2^24-1) and
# the default is 262144
node.conn[0].iscsi.MaxRecvDataSegmentLength = 262144

# To specify the maximum number of data bytes the initiator will send
# in an iSCSI PDU to the target, edit the following line.
#
# The value is the number of bytes in the range of 512 to (2^24-1).
# Zero is a special case. If set to zero, the initiator will use
# the target's MaxRecvDataSegmentLength for the MaxXmitDataSegmentLength.
# The default is 0.
node.conn[0].iscsi.MaxXmitDataSegmentLength = 0

# To specify the maximum number of data bytes the initiator can receive
# in an iSCSI PDU from a target during a discovery session, edit the
# following line.
#
# The value is the number of bytes in the range of 512 to (2^24-1) and
# the default is 32768
# 
discovery.sendtargets.iscsi.MaxRecvDataSegmentLength = 32768

# To allow the targets to control the setting of the digest checking,
# with the initiator requesting a preference of enabling the checking, uncomment# one or both of the following lines:
#node.conn[0].iscsi.HeaderDigest = CRC32C,None
#node.conn[0].iscsi.DataDigest = CRC32C,None
#
# To allow the targets to control the setting of the digest checking,
# with the initiator requesting a preference of disabling the checking,
# uncomment one or both of the following lines:
#node.conn[0].iscsi.HeaderDigest = None,CRC32C
#node.conn[0].iscsi.DataDigest = None,CRC32C
#
# To enable CRC32C digest checking for the header and/or data part of
# iSCSI PDUs, uncomment one or both of the following lines:
#node.conn[0].iscsi.HeaderDigest = CRC32C
#node.conn[0].iscsi.DataDigest = CRC32C
#
# To disable digest checking for the header and/or data part of
# iSCSI PDUs, uncomment one or both of the following lines:
#node.conn[0].iscsi.HeaderDigest = None
#node.conn[0].iscsi.DataDigest = None
#
# The default is to never use DataDigests or HeaderDigests.
#

# For multipath configurations, you may want more than one session to be
# created on each iface record.  If node.session.nr_sessions is greater
# than 1, performing a 'login' for that node will ensure that the
# appropriate number of sessions is created.
node.session.nr_sessions = 1

#************
# Workarounds
#************

# Some targets like IET prefer after an initiator has sent a task
# management function like an ABORT TASK or LOGICAL UNIT RESET, that
# it does not respond to PDUs like R2Ts. To enable this behavior uncomment
# the following line (The default behavior is Yes):
node.session.iscsi.FastAbort = Yes

# Some targets like Equalogic prefer that after an initiator has sent
# a task management function like an ABORT TASK or LOGICAL UNIT RESET, that
# it continue to respond to R2Ts. To enable this uncomment this line
# node.session.iscsi.FastAbort = No
```

重启一下服务

```sh
systemctl status iscsid
systemctl restart iscsid
```



## 2、改动 hosts文件。

这一步保证了，云管理平台的正常访问。

第一台机器

```sh
127.0.0.1 localhost
172.26.0.203 cvknode2
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
172.26.0.202 cvknode1
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
172.26.0.202 center.authorization.h3c.com
```

第二台机器

```sh
127.0.0.1 localhost
172.26.0.203 cvknode2
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
172.26.0.202 cvknode1
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
```

由此可见，第一台机器似乎决定了 云管理平台的访问 DNS。



## 3、在共享存储服务器 连了 主机下的映射端口。

进入共享存储的web 管理页面，进入主机；

![Screenshot 2025-01-29 at 10.13.52 PM](%E6%96%AD%E7%94%B5%E6%8E%89ISCS%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/Screenshot%202025-01-29%20at%2010.13.52%E2%80%AFPM.png)

查看主机属性，重新添加一遍。

![Screenshot 2025-01-29 at 10.15.54 PM](%E6%96%AD%E7%94%B5%E6%8E%89ISCS%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/Screenshot%202025-01-29%20at%2010.15.54%E2%80%AFPM.png)



## 4、在管理平台添加

 先集群添加，

![Screenshot 2025-02-16 at 11.29.03 AM](%E6%96%AD%E7%94%B5%E6%8E%89ISCS%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/Screenshot%202025-02-16%20at%2011.29.03%E2%80%AFAM.png)

再每台机器添加。

![Screenshot 2025-02-16 at 11.32.42 AM](%E6%96%AD%E7%94%B5%E6%8E%89ISCS%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/Screenshot%202025-02-16%20at%2011.32.42%E2%80%AFAM.png)

![Screenshot 2025-02-16 at 11.34.09 AM](%E6%96%AD%E7%94%B5%E6%8E%89ISCS%E5%85%B1%E4%BA%AB%E5%AD%98%E5%82%A8%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/Screenshot%202025-02-16%20at%2011.34.09%E2%80%AFAM.png)

IP地址可以物理机器 用 命令看见 

```sh
sudo iscsiadm -m session
```

