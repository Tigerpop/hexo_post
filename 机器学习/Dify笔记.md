---
layout: posts
title: Dify笔记
date: 2025-5-16 16:27:21
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "dify"
- "workflow"
- "chatflow"
---



# Dify 可以做什么

- 聊天助手
- 文本生成应用
- Agent
- Chatflow 对话流：复杂流程的多轮对话场景，支持记忆，动态应用编排；
- workflow 工作流：自动化、批处理等单轮生成类型任务的应用编排方式，单向生成结果；



# 下载安装

```bash
cd /home/cys/data/docker-data

git https://github.com/langgenius/dify.git

cd /home/cys/data/docker-data/dify/docker

docker compose up -d 
```

![截屏2025-05-01 23.07.40](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-01%2023.07.40.png)

# 修改配置

```sh
cd /home/cys/data/docker-data/dify/docker

# 这里的 docker-compose.yaml 文件我就不全部展开了，服务很多，写了灵活的配置，会从 .env 文件中 调用，我们对 几个关键的位置 就行修改。

# 我们 需要 先明白几个点，在 yaml 文件中的那个服务 如果没有明确宿主机和容器的映射关系，那就是只在这个容器中内部使用，然后通过yaml 中写的 docker network 连接 其他几个容器，没有和宿主机扯上关系；一个宿主机下有几个这样不端口映射的容器，那么几个（如radis） 的容器运行是不会相互影响的。

```

## 存储位置

从你的 `docker-compose.yml` 文件可以看出，当前数据存储主要通过以下服务挂载到容器中：

1. **PostgreSQL（db服务）**：`./volumes/db/data`
2. **Redis**：`./volumes/redis/data`
3. **Weaviate/Qdrant/Chroma等向量库**：`./volumes/weaviate`、`./volumes/qdrant`
4. **插件和存储目录**：`./volumes/app/storage`
5. **其他辅助服务**：如 `sandbox`、`plugin_daemon` 等

而我的dify 下载的位置本身就在 `/home/cys/data/docker-data/dify`，`/home/cys/data` 下挂了52T的raid 磁盘空间是足够的。所以就不需要再修改了。

![微信图片_20250515100036_463](Dify%E7%AC%94%E8%AE%B0/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250515100036_463.png)

可以看见 数据其实都在 `/home/cys/data/docker-data/dify/docker/volumes` 中。

## **核心服务列表**

|     服务名称      |                  功能说明                  |     端口映射/内部端口      |
| :---------------: | :----------------------------------------: | :------------------------: |
|     **nginx**     |     反向代理，对外暴露 Web 和 API 服务     |     `80:80`、`443:443`     |
|      **web**      |               Dify 前端界面                | 通过 nginx 代理（默认 80） |
|      **api**      |               Dify 后端 API                | 通过 nginx 代理（默认 80） |
|    **worker**     | 异步任务处理服务（如数据集处理、模型调用） |         无直接暴露         |
|      **db**       |             PostgreSQL 数据库              |         内部 5432          |
|     **redis**     |                 Redis 缓存                 |         内部 6379          |
|    **sandbox**    |      代码沙箱环境（用于插件安全执行）      |         内部 8194          |
|  **ssrf_proxy**   |       安全代理服务（防止 SSRF 攻击）       |         内部 3128          |
| **plugin_daemon** |        插件守护进程（用于插件管理）        |      外部 `5003:5003`      |
|   **weaviate**    |          向量数据库（需手动启用）          |         内部 8080          |
|    **qdrant**     |          向量数据库（需手动启用）          |         内部 6333          |
| **elasticsearch** |         全文搜索服务（需手动启用）         |      外部 `9200:9200`      |

## 修改端口

```sh
# 我想把 默认的Dify前端后端的port改成 8003 和 8004；

# 确保宿主机的 8003 和 8004 端口未被占用：# 若无输出，则表示端口可用
sudo netstat -tuln | grep -E '8003|8004'

# 由于Dify的访问是通过 nginx 代理（默认 80），所以我们实际上是对yaml中 nginx 相关内容修改。
```

Docker-compose.yaml 的services:中，修改nginx服务的映射端口：

`${EXPOSE_NGINX_PORT:-8003}:${NGINX_PORT:-8003}`

`'${EXPOSE_NGINX_SSL_PORT:-8004}:${NGINX_SSL_PORT:-8004}'`

```yaml 
------------------------------------------------------------------------
 nginx:
    image: nginx:latest
    restart: always
    volumes:
      - ./nginx/nginx.conf.template:/etc/nginx/nginx.conf.template
      - ./nginx/proxy.conf.template:/etc/nginx/proxy.conf.template
      - ./nginx/https.conf.template:/etc/nginx/https.conf.template
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/docker-entrypoint.sh:/docker-entrypoint-mount.sh
      - ./nginx/ssl:/etc/ssl # cert dir (legacy)
      - ./volumes/certbot/conf/live:/etc/letsencrypt/live # cert dir (with certbot container)
      - ./volumes/certbot/conf:/etc/letsencrypt
      - ./volumes/certbot/www:/var/www/html
    entrypoint: [ 'sh', '-c', "cp /docker-entrypoint-mount.sh /docker-entrypoint.sh && sed -i 's/\r$$//' /docker-entrypoint.sh && chmod +x /docker-entrypoint.sh && /docker-entrypoint.sh" ]
    environment:
      NGINX_SERVER_NAME: ${NGINX_SERVER_NAME:-_}
      NGINX_HTTPS_ENABLED: ${NGINX_HTTPS_ENABLED:-false}
      NGINX_SSL_PORT: ${NGINX_SSL_PORT:-8004}  #${NGINX_SSL_PORT:-443}
      NGINX_PORT: ${NGINX_PORT:-8003} #${NGINX_PORT:-80}
      # You're required to add your own SSL certificates/keys to the `./nginx/ssl` directory
      # and modify the env vars below in .env if HTTPS_ENABLED is true.
      NGINX_SSL_CERT_FILENAME: ${NGINX_SSL_CERT_FILENAME:-dify.crt}
      NGINX_SSL_CERT_KEY_FILENAME: ${NGINX_SSL_CERT_KEY_FILENAME:-dify.key}
      NGINX_SSL_PROTOCOLS: ${NGINX_SSL_PROTOCOLS:-TLSv1.1 TLSv1.2 TLSv1.3}
      NGINX_WORKER_PROCESSES: ${NGINX_WORKER_PROCESSES:-auto}
      NGINX_CLIENT_MAX_BODY_SIZE: ${NGINX_CLIENT_MAX_BODY_SIZE:-15M}
      NGINX_KEEPALIVE_TIMEOUT: ${NGINX_KEEPALIVE_TIMEOUT:-65}
      NGINX_PROXY_READ_TIMEOUT: ${NGINX_PROXY_READ_TIMEOUT:-3600s}
      NGINX_PROXY_SEND_TIMEOUT: ${NGINX_PROXY_SEND_TIMEOUT:-3600s}
      NGINX_ENABLE_CERTBOT_CHALLENGE: ${NGINX_ENABLE_CERTBOT_CHALLENGE:-false}
      CERTBOT_DOMAIN: ${CERTBOT_DOMAIN:-}
    depends_on:
      - api
      - web
    ports:
      - '${EXPOSE_NGINX_PORT:-8003}:${NGINX_PORT:-8003}'  #'${EXPOSE_NGINX_PORT:-80}:${NGINX_PORT:-80}'
      - '${EXPOSE_NGINX_SSL_PORT:-8004}:${NGINX_SSL_PORT:-8004}'  #'${EXPOSE_NGINX_SSL_PORT:-443}:${NGINX_SSL_PORT:-443}'
------------------------------------------------------------------------
```

.env

```yaml
# 找到 nginx 的外部端口 和容器内端口 一共四个地方，修改即可，这里用 https的 走8004，http走8003 ；
EXPOSE_NGINX_PORT=8003
EXPOSE_NGINX_SSL_PORT=8004
NGINX_PORT=8003
NGINX_SSL_PORT=8004
```

防火墙给端口开一下；

```sh
sudo ufw allow 8003/tcp
sudo ufw allow 8004/tcp
```

重启开启一下 

```sh
docker compose down
docker compose up -d
```

如果没有 成功开启 可以 `docker compose logs nginx` 检查一下 容器日志。

`./nginx/docker-entrypoint.sh `这个脚本会决定最终生成的  `default.conf`

```sh
cys@cysserver:~/data/docker-data/dify/docker/nginx$ ls
conf.d  docker-entrypoint.sh  https.conf.template  nginx.conf.template  proxy.conf.template  ssl

cys@cysserver:~/data/docker-data/dify/docker/nginx/conf.d$ ls
default.conf  default.conf.template  default.conf~
```

我们可以看到 这个 `default.conf` 文件 中 监听的端口已经是8003了。

```sh
server {
    listen 8003;
    server_name _;

    location /console/api {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /api {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /v1 {
      proxy_pass http://api:5001;
      include proxy.conf;
    }
    ...
    }
```

## 进入web页面

访问：   http://10.5.9.252:8003/

注册一下Dify。

![截屏2025-05-07 21.50.46](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-07%2021.50.46.png)

在设置中安装模型供应商的插件，我们之前在本地部署了 vllm集群的 Deepseek，也可以换大模型，但是换了模型就要在这里换一下对应的插件。

![截屏2025-05-08 09.43.46](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-08%2009.43.46.jpg)

![截屏2025-05-08 09.46.50](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-08%2009.46.50.jpg)

![截屏2025-05-14 10.39.53](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-14%2010.39.53.jpg)

在这里可以 把我们之前本地部署的大模型和嵌入模型等等 都配置进去；

![截屏2025-05-14 10.38.13](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-14%2010.38.13.jpg)

同样的道理，我之前在本地部署的 嵌入模型 也可以用同样的方法添加进来，在模型供应商中找啊到 我用的Text Embedding Inference。

![截屏2025-05-15 10.17.21](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2010.17.21.jpg)

![截屏2025-05-15 10.22.42](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2010.22.42.jpg)

我这里的 URL 是 `http://10.5.9.252:8002` 没有 带 /v1 。

![截屏2025-05-15 10.31.24](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2010.31.24.jpg)

插件中可见 已经有了 嵌入模型了。

![截屏2025-05-15 10.35.20](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2010.35.20.jpg)

# AI 应用的创建

点击“工作室” 创建空白应用。

## 创建聊天助手

![截屏2025-05-14 10.48.53](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-14%2010.48.53.jpg)

填写提示词

![截屏2025-05-14 10.52.30](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-14%2010.52.30.jpg)

推荐用 大语言模型生成提示词，等个几十秒吧。![截屏2025-05-14 11.01.53](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-14%2011.01.53.jpg)

生成以后再应用一下就好。

在调试和预览、对话、知识库什么的都可以用，右上角可以换大模型。

![截屏2025-05-15 10.33.00](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2010.33.00.jpg)

## 创建Agent

打个比方 是 生成 图片的 Agent ，可以先本地运行一个 多模态的模型，然后 在插件中 下载一个 stability，在 工作室 Agent 界面下方 工具中添加 stability 插件。

![截屏2025-05-15 14.50.36](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2014.50.36.png)

## 创建文本生成应用

![截屏2025-05-15 14.55.28](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2014.55.28.jpg)

打一个 `{` 就会让你选择是否新建变量。

![截屏2025-05-15 14.57.37](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2014.57.37.jpg)

![截屏2025-05-15 14.58.14](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2014.58.14.jpg)

![截屏2025-05-15 15.01.29](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2015.01.29.jpg)

![截屏2025-05-15 15.02.29](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2015.02.29.jpg)

可见 右边 “调试与预览” 中 出现了左边 提示词的 内容；

点击运行

![截屏2025-05-15 16.52.43](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2016.52.43.jpg)



# AI应用工具箱

![截屏2025-05-15 17.20.55](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2017.20.55.png)

以聊天助手为例

![截屏2025-05-15 17.25.30](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2017.25.30.jpg)

![截屏2025-05-15 17.25.16](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2017.25.16.jpg)

对话开场白 可以手动给用户选择，下一步问题的建议、 实现内容审查等等。![截屏2025-05-15 17.28.42](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2017.28.42.png)![截屏2025-05-15 17.29.05](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2017.29.05.png)

![截屏2025-05-15 17.30.26](../../%E6%88%AA%E5%B1%8F2025-05-15%2017.29.39.png)

![截屏2025-05-15 17.30.26](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2017.30.26-7301625.png)



# 工作流（关键）

![截屏2025-05-15 21.16.09](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.16.09.png)

## 变量

![截屏2025-05-15 21.17.23](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.17.23.png)

系统变量（sys.xxx）、环境变量、会话变量 其实都是和编程语言类似的逻辑。

![截屏2025-05-15 21.33.37](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.33.37.png)

![截屏2025-05-15 21.34.52](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.34.52.png)

## 节点

![截屏2025-05-15 21.38.21](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.38.21.png)

## chatflow工作流

我新建了一个 chatflow工作流（和workflow区别就是chatflow是对话形式）

![截屏2025-05-15 21.46.07](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.46.07.jpg)

![截屏2025-05-15 21.47.14](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.47.14.jpg)

![截屏2025-05-15 21.55.06](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.55.06.jpg)

选择直接回复大模型的text。

![截屏2025-05-15 21.56.45](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2021.56.45.jpg)

一个节点的输出喂下一个节点。

![截屏2025-05-15 22.18.22](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2022.18.22.jpg)

![截屏2025-05-15 22.20.24](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2022.20.24.jpg)

![截屏2025-05-15 22.30.19](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-05-15%2022.30.19.png)

## workflow工作流

workflow类似我原来开发的自动化处理流程，而chatflow类似于对话的形式；

一个是工作软件一个对对话助手。



# 知识库

RAG 可视化界面操作。

一个文件大小上限15M，![123](Dify%E7%AC%94%E8%AE%B0/123.png)

![微信图片_20250516101549_501](Dify%E7%AC%94%E8%AE%B0/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250516101549_501.png)这里来设置如何切文档或者表格、文件的嵌入模型和检索设置；向量、全文还是混合搜索，之前的文档中我有过介绍。这样直接可视化的切分确实比open webUI看起来直观很多。

![微信图片_20250516101727_502](Dify%E7%AC%94%E8%AE%B0/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250516101727_502.png)![微信图片_20250516101720_503](Dify%E7%AC%94%E8%AE%B0/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250516101720_503.png)

这样就添加好了一个 文件。

关联上知识库以后，就可以用了。

![微信图片_20250516103408_504](Dify%E7%AC%94%E8%AE%B0/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250516103408_504.png)



# 发布AI应用

由于我们是内网使用，所以直接运行在服务器上就可以了。

![微信图片_20250516110401_5](Dify%E7%AC%94%E8%AE%B0/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250516110401_5.png)

- 点击运行，会直接出现一个网址，别的机器访问这个url就可以实现访问dify。

- 或者点击下方的嵌入到网页，把一段它给出的 代码 粘贴进 html 的前端页面中即可。

- 或者 直接用 APIs ，不关注后端情况，直接用接口。

  ![564](Dify%E7%AC%94%E8%AE%B0/564.png)

  

这样的做法可以让应用更加定制化一些，相比于open webUI。





# 遇到的问题以及解决方法

**1、发布的时候，会出现404分不到，在“探索”中打开却能正常访问。**

这是因为我前面yaml配置中env配置中修改过端口号。手动在生成的URL中的IP后添加端口即可。

![333](Dify%E7%AC%94%E8%AE%B0/333.png)

同样的道理，在 **嵌入到前端页面 **中的时候也要加上端口号。

例如：

```html
<script>
 window.difyChatbotConfig = {
  token: 'RtixHB3eI1lQvUsR',
  baseUrl: 'http://10.5.9.252',
  systemVariables: {
    // user_id: 'YOU CAN DEFINE USER ID HERE',
    // conversation_id: 'YOU CAN DEFINE CONVERSATION ID HERE, IT MUST BE A VALID UUID',
  },
 }
</script>
<script
 src="http://10.5.9.252/embed.min.js"
 id="RtixHB3eI1lQvUsR"
 defer>
</script>
<style>
  #dify-chatbot-bubble-button {
    background-color: #1C64F2 !important;
  }
  #dify-chatbot-bubble-window {
    width: 24rem !important;
    height: 40rem !important;
  }
</style>
```

上方统一需要在252后加 8003 我前面改的端口号。





# 参考：

https://docs.dify.ai/zh-hans/guides/workflow/node/doc-extractor
