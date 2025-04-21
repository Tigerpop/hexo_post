---
layout: posts
title: huggingface嵌入模型
date: 2025-4-21 022:08:08
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "ritrieve_zh_v1"
- "ONNX"
- "嵌入模型"
- "Text Embeddings Inference"
- "Safetensors"

---





# 查看嵌入模型排名

访问网址：

 https://huggingface.co/spaces/mteb/leaderboard 

![截屏2025-04-15 16.41.21](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-04-15%2016.41.21.jpg)

选择中文

![截屏2025-04-15 17.29.55](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-04-15%2017.29.55.jpg)

![截屏2025-04-15 17.30.19](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-04-15%2017.30.19.jpg)

![截屏2025-04-15 18.33.17](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-04-15%2018.33.17.jpg)

注意，英文的嵌入模型好，不代表 中文的向量化的效果好。



# huggingface下载模型

参考：

https://zhuanlan.zhihu.com/p/663712983

推荐这样用huggingface推荐工具优雅的下载。
![截屏2025-04-15 18.56.30](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-04-15%2018.56.30.jpg)

```bash
# 下载工具，最好提前给终端开一下代理。
pip install -U huggingface_hub

# 搞里头，richinfoai/ritrieve_zh_v1 在前面看排名的地方点进去可见。
huggingface-cli download richinfoai/ritrieve_zh_v1 --local-dir ./ritrieve_zh_v1

# 然后检查一下 是不是 和前面仓库中显示的一样。
ls ./ritrieve_zh_v1/
```

传输，这一步可能是因为服务器不能科学上网，我们把下载好的模型传到服务器中。

我这里是从MacBook 传到Linux中。

```bash
tar -czvf ritrieve_zh_v1.tar.gz ritrieve_zh_v1/

# Mac原生支持md5命令，输出格式为 MD5 (文件名) = 校验值
md5 ritrieve_zh_v1.tar.gz > ritrieve_zh_v1.tar.gz.md5
# 如需兼容Linux校验
awk '{print $4"  "$2}' ritrieve_zh_v1.tar.gz.md5 > temp.md5 && mv temp.md5 ritrieve_zh_v1.tar.gz.md5

scp ritrieve_zh_v1.tar.gz* cys@10.5.9.252:/home/cys/data/models/embeddingModel/.
```

在Linux服务器中校验

```bash 
cd /home/cys/data/models/embeddingModel

# 如果有括号就去掉，标准格式是 
# f0c60651c2347bb4c000e715920b7dbd ritrieve_zh_v1.tar.gz
vim ritrieve_zh_v1.tar.gz.md5 

# 校验，正确结果是 ritrieve_zh_v1.tar.gz: OK
md5sum -c ritrieve_zh_v1.tar.gz.md5

tar -xzvf ritrieve_zh_v1.tar.gz

# 检查文件
ls
```



# 模型中介（huggingface/text-embeddings-inference）

在这里可以查看使用嵌入模型的通用中介工具的相关内容：

https://github.com/huggingface/text-embeddings-inference/pkgs/container/text-embeddings-inference

![截屏2025-04-16 15.33.27](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E6%88%AA%E5%B1%8F2025-04-16%2015.33.27.jpg)



# 配置 yml 文件

```yaml
cd /home/cys/docker_data/embeddingModel/ritrieve_zh_v1

vim docker-compose.yml 
--------------------------------- docker-compose.yml ------------------------------------
services:
  text-embeddings-inference:
    user: "0:0"
    # 使用 CPU 版镜像，确保不调用 GPU 资源
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-1.7  #ghcr.io/huggingface/text-embeddings-inference:cpu-1.6
    ports:
      - "8002:80"
    volumes:
      - "/home/cys/data/models/embeddingModel:/data"
    environment:
      - EMBEDDING_API_KEY="wiekdoid@JIDj124123"
      # 设置分词工作线程数，建议等于或接近 CPU 线程数（24线程）
      - TOKENIZATION_WORKERS=18
      # 设置最大批处理 token 数，利用大内存提高吞吐量（根据实际负载可调高）
      - MAX_BATCH_TOKENS=32768
      # 客户端单次批处理请求的最大输入数（可根据实际情况调优）
      - MAX_CLIENT_BATCH_SIZE=64
    command: ["--model-id", "/data/ritrieve_zh_v1", "--auto-truncate"]
    networks:
      - my-network

networks:
  my-network:
    external: true
    name: vllm-network
--------------------------------- docker-compose.yml -----------------------------------

docker compose up -d
```



# open webUI控制页面修改

在设置、文档、中选择 “语意向量模型” ，重排模型等等修改成 ritrieve_zh_v1

![微信图片_20250421105834](huggingface%E5%B5%8C%E5%85%A5%E6%A8%A1%E5%9E%8B/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250421105834.png)



# 检查模型是否可用

```sh
curl -X POST "http://localhost:8002/embed" -H "Content-Type: application/json" -d '{
  "inputs": ["你好，世界"],
  "truncate": false
}'
# 看看 会不会返回false。
```



# 可能遇到的报错 

## onnx/model.onnx does not exist

`onnx/model.onnx does not exist`

错误根因：ORT 后端找不到 ONNX 文件

TEI 支持 Safetensors，但生产级部署推荐 ONNX

将 Safetensors 模型导出为 ONNX

```sh
# 安装 Optimum ONNX Runtime 支持：
pip install --upgrade optimum[onnxruntime]
pip install --upgrade sentence-transformers optimum[exporters]

# 导出模型（示例以 richinfoai/ritrieve_zh_v1 为例，--task feature-extraction 用于嵌入模型）：
optimum-cli export onnx \
  --model /home/cys/data/models/embeddingModel/ritrieve_zh_v1 \
  --task feature-extraction \
  /data/onnx
  
mv /data/onnx /home/cys/data/models/embeddingModel/ritrieve_zh_v1/onnx


# 重新 docker compose up -d
```

