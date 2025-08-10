---
layout: posts
title: Cloudflare免费代理节点（可用chatgpt、X等）教程
date: 2025-8-8 08:08:18
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "科学那个上网"
tags:
- "cloudflare"
- "域名"
- "代理"
- "科学上网"


---



# 原理

利用 cloudflare 中的 workers ，我们自行修改 脚本实现代码 完成科学上网。

![截屏2025-08-08 15.48.17](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/%E6%88%AA%E5%B1%8F2025-08-08%2015.48.17.png)

实测效果相当优秀，4k视频毫无压力。好评，感谢  甬哥 提供的代码和 [Seven科技生活](https://7techlife.blogspot.com/)的讲解；

## 解决问题：

1、之前通过 cloudflare 代理有时候 会失效是 因为proxyip 经常久了以后就失效了；

2、同时通过 cloudflare代理 不能访问 chatgpt、X等 网站。这是因为 这些网站也是cloudflare来解析的域名，cloudflare 为了避免内部相互访问加了措施，所以不能访问。

这里我们用了 甬哥代码方案解决上述两个问题，实现科学上网，这种方法是把我们的cloudflare伪装成了外部的，从而实现对 chatgpt 、X等 网站，同时用了 甬哥 代码避开了proxyip，不再当心proxyip失效。

## 存在问题：

IP不停跳转，有一些网站或者一些网站的文件 写有对IP的限制，就会被限，但是这样的概率是很小的，我们最好留一个便宜的备用 科学上网代理 如proton 。





# 一、在cloudflare创建worker

worker命名不要写vpn、vps、代理、翻墙、科学上网 之类的词，避免被cloudflare查出来然后限掉。

![6a2292e308ec6a45c8d9161e2b3328b1](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/6a2292e308ec6a45c8d9161e2b3328b1.png)

![8b6f801636850c85e77a99d0ba4c2a91](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/8b6f801636850c85e77a99d0ba4c2a91.png)



# 二、复制代码到worker

我们选择了[Vless_workers_pages](https://github.com/yonggekkk/Cloudflare-vless-trojan/tree/main/Vless_workers_pages)，[vless无需proxyip的nat64套壳版 (推荐使用).js](https://github.com/yonggekkk/Cloudflare-vless-trojan/commit/ccf3b0b7a160cac7f1ca5c4fe3d0efcfab85ce14)

![c2e6be635ddea0c6fb5edcfdc2f07484](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/c2e6be635ddea0c6fb5edcfdc2f07484-4641484.png)

![08d7365ac07f541098274c7b35479aea](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/08d7365ac07f541098274c7b35479aea-4641565.png)



# 三、通过环境变量修改uuid

![f22fbaba26feb01cb38803a6d29ebb28](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/f22fbaba26feb01cb38803a6d29ebb28-4641379.png)



# 四、使用自定义域名

![b5bc13c194f2f6817ae1b108a49a4401](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/b5bc13c194f2f6817ae1b108a49a4401.png)



# 五、查看节点连接并使用

我们用 cloudflare中worker下的  “自定义域”+斜杆/+“自定义的文本变量uuid”来访问节点指导页面；

![截屏2025-08-08 16.30.08](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/%E6%88%AA%E5%B1%8F2025-08-08%2016.30.08.png)

![截屏2025-08-08 16.31.49](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/%E6%88%AA%E5%B1%8F2025-08-08%2016.31.49.png)

## 以macOS下v2rayN为例：

我们用上图中的 “聚合通用订阅链接”，用GitHub 中的v2rayN 项目操作；

导入链接：

![2f85fe107c9d501af2056cbc7e3c59ab](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/2f85fe107c9d501af2056cbc7e3c59ab-4642195.png)

更新订阅（不通过代理）：

![cba492cf556fbd28a6ef300c393d5a52](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/cba492cf556fbd28a6ef300c393d5a52.png)

![f99915de4db2db677ed9d55e0623a7ff](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/f99915de4db2db677ed9d55e0623a7ff.png)

![def170f7f1b0e8eb02289a9a9ccfb425](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/def170f7f1b0e8eb02289a9a9ccfb425.png)

注意 v2rayN 左下角处，代理走的都是 10808 端口，如果chromo浏览器中有 ZeroOmega 代理插件的话，也要设置一个新的 10808端口的情景模式；

复制终端代理命令以后，在终端运行后，可以用下面命令来检查是否已经定位到了外部：

```bash
curl ipinfo.io       # 看你的所在位置。
curl -I google.com   # 返回200 2XX 就是 成功了。
```

推荐用 curl 命令来经验而不用ping，是因为

`ping` 命令使用的协议是 **ICMP（Internet Control Message Protocol）**，它工作在网络层（OSI 模型的第三层）。`ping` 命令会直接向目标服务器发送 ICMP Echo Request 数据包，并期待收到 Echo Reply 数据包。

而设置的代理 (`http_proxy`, `https_proxy`, `all_proxy` 等) 都是应用层代理，它们工作在 TCP/IP 协议栈的上层。



## 移动端的v2rayNG为例：

移动端的v2rayNG应用中需要启用本地dns、虚拟dns、分片Fragment； 

![b87269748c92e254eca78e4bc22a4a46](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/b87269748c92e254eca78e4bc22a4a46.jpg)

![28262364b250a64fc38a56cfb426ae5a](Cloudflare%E5%85%8D%E8%B4%B9%E4%BB%A3%E7%90%86%E8%8A%82%E7%82%B9%EF%BC%88%E5%8F%AF%E7%94%A8chatgpt%E3%80%81X%E7%AD%89%EF%BC%89%E6%95%99%E7%A8%8B/28262364b250a64fc38a56cfb426ae5a.jpg)

### 追加HTTP代理至VPN

​	按照上面的操作，手机浏览器是可以科学上网的，但是ChatGPT等相关应用是不行的。为了解决这个应用都能科学上网，我们需要追加HTTP代理至VPN。

​	需要关闭 “分片Fragment”，而开启 “追加HTTP代理至VPN”，才能访问Chatgpt，而且需要重启一下 v2rayNG 一次，才能生效；

原理解释是：

就是把原本的“间接”转发方式改成了“直接”转发。

- **传统方式 (未开启此选项)：** 

  浏览器 -> 虚拟网卡设备 -> v2rayNG -> 代理服务器

- **开启此选项后 (针对支持的应用)：** 

  浏览器 -> v2rayNG (直接HTTP代理) -> 代理服务器

无论你的流量是经过虚拟网卡还是直接通过HTTP代理转发，v2rayNG都会对其进行**加密和代理**。最终，你的数据还是会以加密的形式传输到v2ray服务器，再由服务器转发到目标网站。



# 参考：

https://www.youtube.com/watch?v=HcD4xYKXuRY&t=178s

## 相关连接：

Cloudflare：[https://dash.cloudflare.com](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbDB6dmw3NDBxMXhVVzJLVHAwdGVyZmNQTEE5d3xBQ3Jtc0tsQk1Tb1hBS2NLN0ZEZTRZNE02RXY0Rnc2THZUY1ZLSS1jRGNzSG1KeTIzYWZMZzVhNFotcDE0UXpoMVZyRjZTb0ZhNng0SnhhNXk3RmoxQVNlZjF1QU02a2xqODE0aXlGZzhJTXRDSTVLLThSZE9iRQ&q=https%3A%2F%2Fdash.cloudflare.com%2F&v=HcD4xYKXuRY) 

甬哥代码地址：[https://github.com/yonggekkk/Cloudfla...](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqa3BSNnNiN3JnVHNPSnI0RWs2S2IyOE9mbk9fd3xBQ3Jtc0ttTFhEbF9RZk9YREtoQ1BTb3A3T2FjMXMtMXNMYXUwZTRUa0l1NXZlT095VF9falI0VHVLRUE1NWNRWkdud0dGVFVRenZLZk9HQTNGbjdkSFVHUFdFMURjdGFNYk5VUFEzRW5uLVRaemgyaTNLMUJjVQ&q=https%3A%2F%2Fgithub.com%2Fyonggekkk%2FCloudflare-vless-trojan&v=HcD4xYKXuRY) 

UUID生成器：[https://www.uuidgenerator.net/](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbkx6TmQtczJudWhicFg3Uk9oWHMyVFppMmFnUXxBQ3Jtc0tuNUdBcGxYcjBlYU8xRnR1Y1ZfSjYzcUJEeE5oLVQ5eGQ0MmJGWjVqQWdEbFA4Y3F1T0dBYlZ1cXhRTExKNUM2S3E1WXBQYmVwUVNEbFRFajM3Z2hwTEVvTjFCTTdFdUxITXgxQl9rNEttdWliV3J2OA&q=https%3A%2F%2Fwww.uuidgenerator.net%2F&v=HcD4xYKXuRY)

