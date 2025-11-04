---
layout: posts
title: Dify利用Cloudflare远程调用MinerU
date: 2025-11-2 13:18:18
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "Dify"
- "docker"
- "MinerU"
- "OCR"
---







# MinerU 部署（docker版本）失败了！

我的 【使用 Dockerfile 构建 Docker 镜像 】这一步总是失败，最后放弃了docker版本的MinerU的安装。

参考MinerU官网：

https://opendatalab.github.io/MinerU/quick_start/docker_deployment/#start-docker-container

```bash
# Download compose.yaml file
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/docker/compose.yaml
```

MinerU 的结构本质上是「两部分组成」

1. OCR / 文档内容识别本体（MinerU Engine）
2. 语言模型（vLLM Server）

也就是说：

- **MinerU 可以只做 OCR 纯识别，不依赖大模型**
- 如果你 **希望得到结构化、可读性很强的段落/表格/语义文档** → 那就需要 **vLLM Server**（也就是将大模型挂载给 MinerU 做后处理）

**注意：**

由于 我用 vllm ray 集群部署大模型和dify的是两台机器甲、乙，而 两台机器中的每一张卡的显存 都已经22879MiB /  24564MiB，快要满了，同时 MinerU 推荐 使用单卡的 gpu 来做OCR 识别，所以我上面的 MinerU 是部署到了第三台机器中，以下涉及到MinerU的都是在第三台机器中运行的。

然后我在 dify 中掉用这集群外的机器的 MinerU。



## 修改yaml文件

看看 yaml 内容,稍微修改，这里不建议 用多卡 做 OCR ，还是推荐它默认的 gpu 0 单卡。我换了一个17777端口。` compose.yaml`

```yaml
services:
  mineru-vllm-server:
    image: mineru-vllm:latest
    container_name: mineru-vllm-server
    restart: always
    profiles: ["vllm-server"]
    ports:
      - 30000:30000
    environment:
      MINERU_MODEL_SOURCE: local
    entrypoint: mineru-vllm-server
    command:
      --host 0.0.0.0
      --port 30000
      # --data-parallel-size 2  # If using multiple GPUs, increase throughput using vllm's multi-GPU parallel mode
      # --gpu-memory-utilization 0.5  # If running on a single GPU and encountering VRAM shortage, reduce the KV cache size by this parameter, if VRAM issues persist, try lowering it further to `0.4` or below.
    ulimits:
      memlock: -1
      stack: 67108864
    ipc: host
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:30000/health || exit 1"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ["0"]
              capabilities: [gpu]

  mineru-api:
    image: mineru-vllm:latest
    container_name: mineru-api
    restart: always
    profiles: ["api"]
    ports:
      - 17777:8000
    environment:
      MINERU_MODEL_SOURCE: local
    entrypoint: mineru-api
    command:
      --host 0.0.0.0
      --port 8000
      # parameters for vllm-engine
      # --data-parallel-size 2  # If using multiple GPUs, increase throughput using vllm's multi-GPU parallel mode
      # --gpu-memory-utilization 0.5  # If running on a single GPU and encountering VRAM shortage, reduce the KV cache size by this parameter, if VRAM issues persist, try lowering it further to `0.4` or below.
    ulimits:
      memlock: -1
      stack: 67108864
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: [ "0" ]
              capabilities: [ gpu ]

  mineru-gradio:
    image: mineru-vllm:latest
    container_name: mineru-gradio
    restart: always
    profiles: ["gradio"]
    ports:
      - 7860:7860
    environment:
      MINERU_MODEL_SOURCE: local
    entrypoint: mineru-gradio
    command:
      --server-name 0.0.0.0
      --server-port 7860
      --enable-vllm-engine true  # Enable the vllm engine for Gradio
      # --enable-api false  # If you want to disable the API, set this to false
      # --max-convert-pages 20  # If you want to limit the number of pages for conversion, set this to a specific number
      # parameters for vllm-engine
      # --data-parallel-size 2  # If using multiple GPUs, increase throughput using vllm's multi-GPU parallel mode
      # --gpu-memory-utilization 0.5  # If running on a single GPU and encountering VRAM shortage, reduce the KV cache size by this parameter, if VRAM issues persist, try lowering it further to `0.4` or below.
    ulimits:
      memlock: -1
      stack: 67108864
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: [ "0" ]
              capabilities: [ gpu ]
```



## 运行

这里需要注意几个点：

一、

MinerU 的GitHub仓库中写的是 需要 

```bash
# Build Docker Image using Dockerfile
# 使用 Dockerfile 本地构建 Docker 镜像
# 先运行 
docker build -t mineru-vllm:latest -f Dockerfile .
# 如果容器内部网络配置问题，无法解析和连接到 huggingface.co，在构建时使用宿主机的网络栈，运行：
docker build --network=host -t mineru-vllm:latest -f Dockerfile .
```

而不能直接去 docker compose 从指定的源去拉取，那样会找不到文件。
二、

```markdown
Due to the pre-allocation of GPU memory by the vllm inference acceleration framework, you may not be able to run multiple vllm services simultaneously on the same machine. Therefore, ensure that other services that might use GPU memory have been stopped before starting the vlm-vllm-server service or using the vlm-vllm-engine backend.
由于 vllm 推理加速框架预先分配了 GPU 内存，您可能无法在同一台机器上同时运行多个 vllm 服务。因此，请确保在启动 vlm-vllm-server 服务或使用 vlm-vllm-engine 后端之前，已停止其他可能使用 GPU 内存的服务。
```

**既然我已经有自己的 vLLM + Ray 推理集群，就不要启 MinerU 的 vLLM-Server。**
 **直接把 MinerU 当 OCR 引擎用，再把结果交给你自己的大模型做处理，是最佳方案。**

```bash
# 仅仅执行 OCR。
docker compose -f compose.yaml --profile api up -d

# 而不要执行：MinerU 下 yaml 中的 vllm-server 
```

> | 服务名             | Profile 名（启动用的）在yaml文件中有写明 |
> | ------------------ | ---------------------------------------- |
> | mineru-vllm-server | vllm-server                              |
> | mineru-api         | api                                      |
> | mineru-gradio      | gradio                                   |
>
> ```bash
> # 启动的是：profiles: ["api"] → mineru-api
> docker compose -f compose.yaml --profile api up -d
> 
> # 启动的是：mineru-gradio
> # docker compose -f compose.yaml --profile gradio up -d
> 
> # 启动的是：mineru-vllm-server
> # docker compose -f compose.yaml --profile vllm-server up -d
> ```



## 处理docker墙内问题

` sudo cat /etc/docker/daemon.json`

```json

{
    "registry-mirrors": [
        "https://docker.m.daocloud.io",
        "https://mirror.ccs.tencentyun.com",
        "https://func.ink",
        "https://proxy.1panel.live",
        "https://docker.zhai.cm"
    ],
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
```

如果遇到 报错
` [+] Running 1/1 ✘ mineru-api Error Head "https://ghcr.io/v2/open-mmlab/mineru/manifests/latest": denied                                 1.5sError response from daemon: Head "https://ghcr.io/v2/open-mmlab/mineru/manifests/latest": denied`

MinerU 镜像在 **ghcr.io**，它默认需要 Token，否则会出现你看到的 `denied`。

> > https://github.com/settings/tokens → 选择 **Classic Tokens** → 打开 `read:packages` 权限,然后执行
>
> ```
> echo YOUR_GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
> ```
>
> ```
> echo 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XXXX  | docker login ghcr.io -u  Tigerpop --password-stdin
> ```

如果还是不行，就直接使用我之前的笔记，Ubuntu上用v2rayA代理的笔记。


# MinerU部署（from source code） 亲测可用

## 安装

### 一、仅仅调用CPU模式

conda 新建虚拟环境，在 此环境>3.10中运行。

```bash
pip install uv
git clone https://github.com/opendatalab/MinerU.git
cd MinerU
python -m venv .venv # 创建虚拟运行空间，运行成功后可见一个.venv虚拟运行文件夹
source .venv/bin/activate # 激活此新建虚拟空间
uv pip install -e .[core] -i https://mirrors.aliyun.com/pypi/simple # 启动安装命令


```

### 二、GPU加速模式（补充）

安装nvidia-cuda-toolkit ，`nvcc --version` 检查是否安装好了。

torch 官网 https://pytorch.org/get-started/locally/  ，.venv虚拟环境中安装cuda版本的三件套 `torch torchvision torchaudio`。可以在 前面 .venv 环境的pip 中先检查是否已经装好。

## 使用

命令使用`mineru`，

```bash
# 仅使用cpu
mineru -p <input_path> -o <output_path>
mineru -p <input_path> -o <output_path> --source modelscope # 国内魔塔源

# 调用GPU
mineru -p <input_path> -o <output_path> -d cuda
mineru -p <input_path> -o <output_path> -d cuda --source modelscope # 国内魔塔源

# 第一次运行的时候会自行下载一些配套工具，稍微等待即可，我GPU做一次A4纸的文件识别大约18秒。
```

## 集成进Dify（重点）

### 构建镜像

在MinerU文件夹内进入docker文件夹，

进入 china 文件夹，目的是找到 Dockerfile 文件，选取 Dockerfile 中指定服务，来本地构建 Docker 镜像，具体哪些服务可以从 compose.yaml 文件中看。

```bash 
cd docker/china
docker build -t mineru-api . # 仅仅为这个服务构建镜像。15G 左右等等，可能花3小时时间。
```

效果

```bash

(.venv) (py3.10) cys@cysserver:/data/MinerU/docker/china$ docker build -t mineru-api .
[+] Building 10267.1s (8/8) FINISHED                                                                                                       docker:default
 => [internal] load build definition from Dockerfile                                                                                                 0.0s
 => => transferring dockerfile: 1.34kB                                                                                                               0.0s
 => [internal] load metadata for docker.m.daocloud.io/vllm/vllm-openai:v0.10.1.1                                                                     0.9s
 => [internal] load .dockerignore                                                                                                                    0.0s
 => => transferring context: 2B                                                                                                                      0.0s
 => [1/4] FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.1.1@sha256:d731ee65c044ae0977421eed3d93f931d4b7d79614394184c939db35b8f28fc2               0.0s
 => CACHED [2/4] RUN apt-get update &&     apt-get install -y         fonts-noto-core         fonts-noto-cjk         fontconfig         libgl1 &&    0.0s
 => [3/4] RUN python3 -m pip install -U 'mineru[core]' -i https://mirrors.aliyun.com/pypi/simple --break-system-packages &&     python3 -m pip c  9464.8s
 => [4/4] RUN /bin/bash -c "mineru-models-download -s modelscope -m all"                                                                           791.6s
 => exporting to image                                                                                                                               9.6s
 => => exporting layers                                                                                                                              9.6s
 => => writing image sha256:c808ac45897484d9771d4b1b36e902955fa9ba6867df2f454bac431a3091b18f                                                         0.0s
 => => naming to docker.io/library/mineru-api     
 
# 可见花了我 10267.1s = 2小时51分7.1秒
```

### 启动 mineru-api 服务

启动 MinerU-api (内容有哪些可以去官网看，也能从compose.yaml看到)

```bash
# 开起伪终端，且退出终端自动 rm 销毁容器。
docker run --rm --gpus all \
  -p 17777:8000 \
  --ipc=host \
  -it mineru-api \
  /bin/bash

# 容器内开启 web API 服务
mineru-api --host 0.0.0.0 --port 8000
# python -m：以模块方式运行 Python 代码
# mineru.api：运行 MinerU 包中的 API 模块
# 0.0.0.0表示监听所有可用的网络接口
# 8000是容器内部的服务端口
# -------------------- 成功时展现 --------------------
2025-11-01 05:14:43.703531494 [W:onnxruntime:Default, device_discovery.cc:164 DiscoverDevicesForPlatform] GPU device discovery failed: device_discovery.cc:89 ReadFileContents Failed to open file: "/sys/class/drm/card0/device/vendor"
Start MinerU FastAPI Service: http://0.0.0.0:8000
The API documentation can be accessed at the following address:
- Swagger UI: http://0.0.0.0:8000/docs
- ReDoc: http://0.0.0.0:8000/redoc
INFO:     Started server process [13]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
# -------------------- 成功时展现 --------------------
```

我们在伪终端和宿主机中检查是否成功，注意伪终端不能exit退出，退出就rm镜像了。

```bash
# 在容器内执行
nvidia-smi
# 检查 MinerU 是否安装
which mineru
mineru --help

# 在宿主机执行
docker ps
docker images
```

我们读 doc 文档 `http://10.5.9.249:17777/docs`

```bash
看见 post 写着 /file_parse  # parse pdf ,这个意思是 post 需要访问的URL是 http://10.5.9.249:17777/file_parse

看见 Request body 后面写着 multipart/form-data # 
# http协议里
# application/json → 发送 JSON 数据（不能直接传文件）
# multipart/form-data → 发送文件和普通字段混合数据
# Python 的 requests 库专门对 multipart/form-data 做了封装：
# resp = requests.post(url, files=files) 就是固定写法。

下面一堆内容，例如 
files *
array<string>
# files → 参数名； * → 必填；files 是一个数组，数组内的元素是字符串，在 HTTP 里，上传文件要用 multipart/form-data。比如你用浏览器上传文件：
# 一段解释数据说明它后面是a.jpg
# 一段二进制数据a.jpg
# 一段解释数据说明它后面是b.jpg
# 一段二进制数据b.jpg
# ...
# 所以这就成了固定的写法：
#  files = [
#     ('files', open('file1.pdf', 'rb')),
#     ('files', open('file2.jpg', 'rb'))
# ] 
# 对照 HTTP：
# 第一个元素 'files' → 对应 name="files"，所以都写成 files。
# 第二个元素 open(..., 'rb') → 文件二进制流，对应 HTTP body
# 数组就直接用 Python list [ ... ] 包裹多个 (name, file) 元组 → 变成数组上传
```

因此 我们用chatgpt生成一个脚本测试

```python
import requests
url = "http://10.5.9.249:17777/file_parse"
files = [
    ('files', open('test.jpg', 'rb')),   # 换成你的文件路径
]
resp = requests.post(url, files=files)
print(resp.json())
```

将一个脚本所在文件夹的 test.jpg 文件中的 文字读出来并终端打印了出来。

### 修改Dify配置

停下Dify，修改 Dify 的docker 下.env文件。由于 Dify运行在A机器中，MinerU在B机器，这里涉及到了一个通信。

```bash
# 停下dify
docker compose down

vim .env 
# ---------------------------- .env ---------------------------- 
FILES_URL=http://api:5001 # 原来是=空的  FILES_URL=
# ---------------------------- .env ---------------------------- 

用户上传文件到 A机器(Dify)
    ↓
Dify 存储文件并生成文件链接：http://A_IP:5001/xxx
    ↓  
Dify 调用 B机器(MinerU) 的 API，传递文件链接
    ↓
MinerU 从 http://A_IP:5001/xxx 下载文件进行处理
    ↓
MinerU 返回处理结果给 Dify

# 重新运行dify
docker compose up -d
```

### Dify页面配置

1. 下载MinerU插件；

2. API Key 授权配置；

   ```bash
   http://10.5.9.249:17777  # MinerU服务的Base URL 
   本地部署                  # 服务类型
   ```

### Dify+MinerU测试

先开一个工作流；

在【开始】里面设置输入文件，定一个 变量；

下一个节点选【工具】里的【MinerU】；

点击 MinerU 节点下的 Parse File，设置 输入变量、解析方法 、开启 OCR 识别；

下一步直接回答，即可测试。



# 利用Cloudflare远程访问

这个以后再做吧，我找同事开启了一个公网的地址，用上了。