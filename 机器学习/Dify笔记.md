---
layout: posts
title: Dify笔记
date: 2025-8-13 16:27:21
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



# 动态更新

对于 docker compose 的 yaml 部署的 dify ，仅仅需要更新这三个容器就可以实现版本更新，但是注意好备份。注意步骤：

1、备份 PostgreSQL

2、备份你的挂载目录

3、备份配置文件.env 和yaml

4、修改docker-compose.yaml文件

5、拉取镜像，并重新启动

```bash
# 根据你实际的容器名和用户名修改命令。在 .env 和 docker ps 中可以看见。
# postgres 表示使用用户名；dify 是要备份的数据库名；docker-db-1 是容器名； 
docker exec -t docker-db-1 pg_dump -U postgres dify > ./backup/dify_backup_$(date +%F).sql


# 至少打包 Dify 主体部分 + PostgreSQL + Redis：
tar czf backup_volumes_$(date +%F).tar.gz ./volumes/app ./volumes/db ./volumes/redis

cp .env backup/.env.bak_$(date +%F)
cp docker-compose.yml backup/docker-compose.yml.bak_$(date +%F)

# yaml 文件中对以下三个容器修改。
services:
  api:
    image: langgenius/dify-api:latest
  web:
    image: langgenius/dify-web:latest
  worker:
    image: langgenius/dify-api:latest
    
docker-compose pull
docker-compose up -d
```

在docker ps 中可以看见 dify 版本。



# 遇到的问题以及解决方法

### **1、发布的时候，会出现404分不到，在“探索”中打开却能正常访问。**

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



### 2、Failed to parse response from plugin daemon to PluginDaemonBasicResponse

遇到报错：

> Failed to parse response from plugin daemon to PluginDaemonBasicResponse [PluginListResponse], url: plugin/a7060300-e4d0-426c-8690-5036fef955e7/management/list

分析问题，是一个守护进程不停的访问，然后返回的格式又不匹配导致的。以下是分析过程：
我们先通过 日志确定问题 `docker logs -f docker-plugin_daemon-1`,发现 

> 2025/08/12 13:02:09 stdio.go:160: [INFO]plugin langgenius/huggingface_tei:0.0.3: Installed model: huggingface_tei 2025/08/12 13:02:09 stdio.go:160: [INFO]plugin langgenius/azure_openai:0.0.17: Installed model: azure_openai [GIN] 2025/08/12 - 13:02:23 | 200 |   19.930974ms |      172.20.0.8 | GET      "/plugin/a7060300-e4d0-426c-8690-5036fef955e7/management/list?page=1&page_size=100"

而我通过 `docker network ls` 找到 `3efdfbe91ed7 docker_default`；

` docker network inspect dify_default` 发现 前面日志中的 172.20.0.8 就是 docker-api-1。我们进一步看它的日志。

`docker logs -f docker-api-1` 提示说 返回的格式不匹配:

>  File "/app/api/.venv/lib/python3.12/site-packages/pydantic/main.py", line 253, in __init__    validated_self = self.__pydantic_validator__.validate_python(data, self_instance=self)                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ pydantic_core._pydantic_core.ValidationError: 1 validation error for PluginDaemonBasicResponse[PluginListResponse] data  Input should be a valid dictionary or instance of PluginListResponse [type=model_type, input_value=[{'id': 'f45acc46-9e4d-4f...5c71c2231', 'meta': {}}], input_type=list]    For further information visit https://errors.pydantic.dev/2.11/v/model_type


我确定了是之前的 插件或者插件容器相关的内容导致的问题，

解决思路如下：

1、清理 PostgreSQL + Redis 中的内容；

2、清理 本地残留插件文件；

3、进入 docker-compose.yaml 文件查看是否版本有明显偏差；

步骤：

1、清理 PostgreSQL + Redis 中的内容：

```bash
# 查看有哪些表
docker exec -it docker-db-1 psql -U postgres -l

docker exec -it docker-db-1 psql -U postgres -d dify_plugin
\dt
# 以 dify_plugin 为例，逐个检查，发现 a7060300-e4d0-426c-8690-5036fef955e7 相关的就删除。
select * from tool_installations;
delete from tool_installations where tenant_id='a7060300-e4d0-426c-8690-5036fef955e7' ;
exit

# 清理 redis
docker exec -it docker-redis-1 redis-cli
FLUSHALL
exit
```

![截屏2025-08-12 20.37.51](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-08-12%2020.37.51.png)

![截屏2025-08-12 23.00.55](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-08-12%2023.00.55.png)

但是一个一个删除很慢，用下面的脚本 模糊搜索 删除也行；

`vim  clean_plugin.sh`

```bash
#!/bin/bash
set -e

PLUGIN_ID="$1"

if [ -z "$PLUGIN_ID" ]; then
    echo "❌ 用法: $0 <插件ID或关键字>"
    exit 1
fi

# 容器名（改成你的）
PG_CONTAINER="docker-db-1"
REDIS_CONTAINER="docker-redis-1"

echo "🔍 开始在 PostgreSQL 中搜索包含 [$PLUGIN_ID] 的记录..."
PG_MATCHES=$(docker exec -i "$PG_CONTAINER" psql -U postgres -d dify -t -A -c "
SELECT table_name, column_name
FROM information_schema.columns
WHERE table_schema='public';
" | while IFS="|" read -r table column; do
    docker exec -i "$PG_CONTAINER" psql -U postgres -d dify -t -A -c \
    "SELECT '$table', '$column', $column FROM \"$table\" WHERE CAST($column AS TEXT) LIKE '%$PLUGIN_ID%'" \
    2>/dev/null
done)

if [ -n "$PG_MATCHES" ]; then
    echo "找到以下匹配记录："
    echo "$PG_MATCHES"
else
    echo "✅ PostgreSQL 未找到匹配记录。"
fi

read -p "⚠️ 是否删除这些 PostgreSQL 记录？(y/n) " confirm
if [[ "$confirm" == "y" ]]; then
    docker exec -i "$PG_CONTAINER" psql -U postgres -d dify -t -A -c "
    SELECT table_name, column_name
    FROM information_schema.columns
    WHERE table_schema='public';
    " | while IFS="|" read -r table column; do
        docker exec -i "$PG_CONTAINER" psql -U postgres -d dify -c \
        "DELETE FROM \"$table\" WHERE CAST($column AS TEXT) LIKE '%$PLUGIN_ID%'" \
        2>/dev/null || true
    done
    echo "🗑 PostgreSQL 相关记录已删除。"
fi

echo "🔍 开始在 Redis 中搜索包含 [$PLUGIN_ID] 的键和值..."
REDIS_KEYS=$(docker exec -i "$REDIS_CONTAINER" redis-cli --scan | grep "$PLUGIN_ID" || true)
REDIS_VAL_KEYS=$(docker exec -i "$REDIS_CONTAINER" redis-cli --scan | while read -r key; do
    if docker exec -i "$REDIS_CONTAINER" redis-cli GET "$key" 2>/dev/null | grep -q "$PLUGIN_ID"; then
        echo "$key"
    fi
done)

if [ -n "$REDIS_KEYS$REDIS_VAL_KEYS" ]; then
    echo "找到以下 Redis 相关键："
    echo "$REDIS_KEYS"
    echo "$REDIS_VAL_KEYS"
else
    echo "✅ Redis 未找到匹配记录。"
fi

read -p "⚠️ 是否删除这些 Redis 键？(y/n) " confirm
if [[ "$confirm" == "y" ]]; then
    echo "$REDIS_KEYS" "$REDIS_VAL_KEYS" | tr ' ' '\n' | sort -u | while read -r key; do
        docker exec -i "$REDIS_CONTAINER" redis-cli DEL "$key" >/dev/null
    done
    echo "🗑 Redis 相关键已删除。"
fi

echo "🎯 清理完成。"
```

运行上面脚本

```bash
chmod +x clean_plugin.sh
./clean_plugin.sh a7060300-e4d0-426c-8690-5036fef955e7 
# a7060300-e4d0-426c-8690-5036fef955e7 是我的插件报错的提示内容。
```

2、清理 本地残留插件文件：

`/home/cys/data/docker-data/dify/docker/volumes` 是我的本地位置，如果你的目录结构不同，请替换路径。

```bash 
# 分别进入下面两个文件夹中，然后删除和报错相关的内容。
cd /home/cys/data/docker-data/dify/docker/volumes/plugin_daemon/plugin
# 逐个检查，发现就 a7060300-e4d0-426c-8690-5036fef955e7 相关的就删除 rm 。
cd /home/cys/data/docker-data/dify/docker/volumes/plugin_daemon/plugin_packages
# 逐个检查，发现就 a7060300-e4d0-426c-8690-5036fef955e7 相关的就删除 rm 。
```

3、进入 docker-compose.yaml 文件修改；

```bash
docker-compose stop plugin_daemon
docker-compose rm -f plugin_daemon

docker rmi <plugin_daemon_image_name>
```

查看 GitHub中 项目的 `docker-compose.yaml `中的 `plugin_daemon`:

![截屏2025-08-13 13.55.24](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-08-13%2013.55.24.png)

进入我们的  `docker-compose.yaml `中修改  `plugin_daemon`:，然后重新拉取，我这里是把  `langgenius/dify-plugin-daemon:latest:0.0.9-local` 改成 `langgenius/dify-plugin-daemon:0.2.0-local `。

![截屏2025-08-13 13.57.24](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-08-13%2013.57.24.png)

```bash
# 把yaml修改与Dify官方GitHub一致
vim docker-compose.yaml 

# 拉取
docker-compose up -d plugin_daemon

# 检查
docker images | grep langgenius/dify-plugin-daemon
```



![截屏2025-08-13 13.59.54](Dify%E7%AC%94%E8%AE%B0/%E6%88%AA%E5%B1%8F2025-08-13%2013.59.54.png)问题解决了。



# 参考：

https://docs.dify.ai/zh-hans/guides/workflow/node/doc-extractor
