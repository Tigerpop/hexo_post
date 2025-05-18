---
layout: posts
title: huggingface安装安全模型
date: 2025-5-18 20:27:21
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "dify"
- " 网络安全模型"
- "chat-template"
---

# 背景

之前写过 huggingface 安装嵌入模型，其实这个网络安全领域的模型是一样的路数。

这里只写一下通过 huggingface 的使用提示，利用 chatgpt 或 deepseek 来实现yaml的改写。

以 `fdtn-ai/Foundation-Sec-8B` 网络安全模型为例子。

# 读文档



![截屏2025-05-17 20.50.49](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-05-17%2020.50.49.jpg)

我们使用 vllm ，点击 User this model 下的 vllm。

![截屏2025-05-17 20.52.58](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-05-17%2020.52.58.jpg)

```sh
# Deploy with docker on Linux:
docker run --runtime nvidia --gpus all \
	--name my_vllm_container \
	-v ~/.cache/huggingface:/root/.cache/huggingface \
 	--env "HUGGING_FACE_HUB_TOKEN=<secret>" \
	-p 8000:8000 \
	--ipc=host \
	vllm/vllm-openai:latest \
	--model fdtn-ai/Foundation-Sec-8B
```



# 通过大语言模型改造出yaml文件

上面这个 docker 安装的例子中我们可以看见 不是 docker compose 的yaml 来实现的，而且默认了通过一个你的huggingface 的token 来实现下载新的模型的。

而我 习惯 先通过

 `huggingface-cli download fdtn-ai/Foundation-Sec-8B --local-dir ./Foundation-Sec-8B`  

把模型下载到本地

（这样就不用这个`HUGGING_FACE_HUB_TOKEN`了）参考之前的安装嵌入模型的笔记 https://tigerpop.github.io/2025/04/21/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/#huggingface%E4%B8%8B%E8%BD%BD%E6%A8%A1%E5%9E%8B，

然后通过宿主机文件夹映射容器，来实现对模型的部署，在yaml 中感觉看起来要直观很多，而且模块化一些。

## 我的需求告知大模型

```apl
官网给的建议是 这样
# Deploy with docker on Linux:
docker run --runtime nvidia --gpus all \
	--name my_vllm_container \
	-v ~/.cache/huggingface:/root/.cache/huggingface \
 	--env "HUGGING_FACE_HUB_TOKEN=<secret>" \
	-p 8000:8000 \
	--ipc=host \
	vllm/vllm-openai:latest \
	--model fdtn-ai/Foundation-Sec-8B
再运行
# Load and run the model:
docker exec -it my_vllm_container bash -c "vllm serve fdtn-ai/Foundation-Sec-8B"

我的需求：
第一：我想要在宿主机的/home/cys/data/models/Foundation-Sec-8B 来映射 而不是用上面推荐的位置；
第二：我想要改成 yaml 的形式 用 docker compose 来使用，这样使用方便一些。只需要先 docker compose up -d 然后再 进入容器中运行 vllm serve fdtn-ai/Foundation-Sec-8B 是不是就开启了这个模型？
第三：能不能在 yaml 中加上一个 API key 这样用起来安全性高一些。
第四：我不需要一键启动，
第五：模型已经下载到了本地的 /home/cys/data/models/Foundation-Sec-8B，上面的HUGGING_FACE_HUB_TOKEN 是不是就不用了。
请回答我，给我一个能用的yaml文件。
```

把上面的内容发送给大语言模型。

大模型的回答的 `docker-compose.yml`是

```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    container_name: vllm_container

    # 1) 启用 NVIDIA GPU 支持,（Compose v1.28+）
    gpus: all
    #runtime: nvidia
    #deploy:
    #  resources:
    #    reservations:
    #      devices:
    #        - driver: nvidia
    #          count: all
    #          capabilities: [gpu]


    # 2) 挂载本地模型目录（将 models 目录映射到 /models）
    volumes:
      - /home/cys/data/models/Foundation-Sec-8B:/models/Foundation-Sec-8B

    # 3) 暴露端口
    ports:
      - "8000:8000"

    # 4) 不自动启动 serve，保留为进入容器后手动执行
    entrypoint:
      - /bin/bash
    tty: true

    # 5) 注入 API Key（在 .env 文件中定义 VLLM_API_KEY=<your_api_key>）
    env_file:
      - .env
```

## 自定义修改

我再根据我的实际情况（单机双卡3090ti）修改一下 yaml 文件，

从 Transformers v4.44 开始，vLLM 不再自动使用默认模板，所以你必须显式地通过 `--chat-template` 把你想要的对话模板传进来。否则所有 `/v1/chat/completions` 请求都会被拒绝。

所以我们需要先在 项目目录下新建一个名为 `openai_chat.jinja` 的文件

```jinja2
{%- macro system_prompt(system) -%}
<|system|>
{{ system }}
<|endofsystem|>
{%- endmacro -%}

{%- for msg in messages -%}
<|{{ msg.role }}|>
{{ msg.content }}
<|endof{{ msg.role }}|>
{%- endfor -%}
```

yaml文件得出如下：

```yaml
version: '3.8'

services:
  vllm:
    image: vllm/vllm-openai:latest
    container_name: vllm_container
    ipc: host  # 使用主机 IPC 命名空间
    # 1) 启用 NVIDIA GPU 支持（Compose v1.28+）
    gpus: all
    #runtime: nvidia
    #deploy:
    #  resources:
    #    reservations:
    #      devices:
    #        - driver: nvidia
    #          count: all
    #          capabilities: [gpu]

    # 2) 环境变量
    environment:
      - CUDA_VISIBLE_DEVICES=0,1
      - PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
      - NCCL_DEBUG=INFO

    # 3) 挂载本地模型目录（models 目录映射到容器内 /models）
    # openai_chat.jinja 找到这个模版，在之后的命令中使用。
    volumes:
      - /home/cys/data/models/Foundation-Sec-8B:/models/Foundation-Sec-8B
      - ./openai_chat.jinja:/templates/openai_chat.jinja:ro

    # 4) 暴露端口
    ports:
      - "8000:8000"

    # 5) 不自动启动 serve，保留为进入容器后手动执行
    entrypoint:
      - /bin/bash
    tty: true

    # 6) 注入 API Key（在 .env 文件中定义 VLLM_API_KEY=<your_api_key>）
    env_file:
      - .env
```

## 使用步骤

1. 在项目yaml所在目录创建一个 `.env`，写入：

   ```yaml
   VLLM_API_KEY=你的_api_key
   ```

2. 启动容器（此时仅启动一个带有 Bash 的容器，不会自动跑服务）：

   ```sh
   docker-compose up -d
   ```

3. 进入容器并启动模型服务：

   ```sh
   docker exec -it vllm_container bash
   ```

4. 在容器内运行：

   ```sh
   # 普通运行
   nohup vllm serve /models/Foundation-Sec-8B \
        --host 0.0.0.0 \
        --port 8000 \
        --api-key $VLLM_API_KEY \
        > vllm.log 2>&1 &
   # 挂在后台，从vllm.log 中看日志。
   
   
   # 结合我的情况（单机双卡3090ti，并使用我们的chat模版）运行：
   nohup vllm serve /models/Foundation-Sec-8B \
        --host 0.0.0.0 \
        --port 8000 \
        --api-key "$VLLM_API_KEY" \
        --served-model-name Foundation-Sec-8B \
        --chat-template /templates/openai_chat.jinja \
        --tensor-parallel-size 2 \
        --pipeline-parallel-size 1 \
        --gpu-memory-utilization 0.90 \
        --max-model-len 8192 \
        --dtype half \
        --swap-space 8 \
        > vllm.log 2>&1 &
   ```
   
   这样，vLLM 将加载 `/models/Foundation-Sec-8B` 下的本地模型，并以 OpenAI 兼容接口在 `http://localhost:8000` 提供服务。
   
   ![微信图片_20250518161721_699](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250518161721_699.png)
   
   宿主机可见 两3090ti 都要跑满了。

![微信图片_20250518160752_692](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250518160752_692.png)

容器内看到这个说明已经 正确开启了。



# 通过Dify 发布

现在dify的vllm插件中添加这个模型

![微信图片_20250518204609_714](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250518204609_714.png)

![微信图片_20250518204702_704](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250518204702_704.png)

然后测试一下

![微信图片_20250518204726_50](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250518204726_50.png)

以网页形式发布，这里注意一下 ，要点击“发布更新”，因为这个模型是在10.5.9.251机器 dify 在10.5.9.252机器，所以刚刚发布会特别傻这个模型，还可能没有改过来，要反复操作然后等一等。

![微信图片_20250518204804_715](huggingface%E5%AE%89%E8%A3%85%E5%AE%89%E5%85%A8%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250518204804_715.png)

上面是发布好的情况。