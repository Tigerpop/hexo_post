---
layout: posts
title: 使用docker让一个centos机器4块GPU被4个人用
date: 2025-12-2 09:13:22
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "docker"
- "容器"
- "构建镜像"
- "Dockerfile"
- "GPU"
- "centos"


---



## 

# 核心目标

**让 4 个人（或 4 个任务）在一台有 4 块 GPU 的机器上，各自独立使用 Jupyter + conda + Pytorch + GPU，彼此互不干扰（文件、环境、GPU 都隔离）。** 

实现：

使用docker 构建一个标准化的镜像（conda、jupyter、pytorch、GPU支持...），然后用这个镜像启动4个独立的容器实例，每个容器实例分1块GPU、1个专属目录，指定内存和CPU的占用。



# 前期准备

## 网络

``` bash
# 我在访问外部网络不通发现了是DNS问题，随后修改 DNS 配置
# 备份原配置
sudo cp /etc/resolv.conf /etc/resolv.conf.bak

# 编辑 DNS 配置
sudo vim /etc/resolv.conf
#________________________________  resolv.conf  ________________________________
search hpc.local.
nameserver 192.168.0.165
nameserver 8.8.8.8
nameserver 114.114.114.114
#________________________________  resolv.conf  ________________________________
# 测试 DNS 解析
ping -c 4 www.baidu.com
ping -c 4 mirrors.aliyun.com
```



## 磁盘

```bash
mkfs.xfs -f /dev/sdb # 格式化14T的盘，之前用lsblk 看见了这个空的 14T空间的盘。

mkdir -p /data/docker

# 设置开机自动挂载
echo "/dev/sdb /data/docker xfs defaults 0 0" >> /etc/fstab

mount -a

# 验证挂载
df -h /data/docker
```



## 安装Nvidia驱动（视情况）

https://www.nvidia.com/en-us/drivers/ 下载驱动

```bash
# 如果有必要可以重装驱动，先卸载 老驱动。
sudo nvidia-uninstall
sudo reboot

# 查看 GPU 
lspci | grep -i nvidia
```

> **应该选择的驱动类型是 ——「Linux 64-bit」版本（.run 文件）或者是RHEL 的版本的，不是 Ubuntu/RedHat/其他包格式）**
>
> 因为：
>
> - CentOS 7 属于 **RHEL 系 Linux**
> - NVIDIA **不再发布 CentOS 7 专用 RPM**（CentOS7 已 EOL，官方移除了）
> - 所以官网只提供 **Runfile（Linux 64-bit）**
> - 这个 run 文件是 **跨发行版通用的 installer**，支持：
>   - CentOS 7
>   - RHEL 7
>   - Fedora
>   - Ubuntu
>   - openSUSE
>   - 其他 Linux，只要是 x86_64
>
> 这是 NVIDIA 官方文档明确说明的。
>
> 但是如果没有你要找的版本的话，centos7 对应的还可以是RHEL 的版本，但是我用的是 
> https://www.nvidia.com/en-us/drivers/details/214092/
> Data Center Driver for Linux x64 525.147.05 | Linux 64-bit 选的是 12.0的cuda支持。
>
> centos7最高只支持到 cuda11.8 ，所以我使用的是支持到cuda12.0的Nvidia驱动。

```bash
# 先在Mac 下载好，因为我 工作中不能上外网。
scp ./NVIDIA-Linux-x86_64-525.147.05.run  root@172.26.1.20:/data/.  

sudo bash NVIDIA-Linux-x86_64-525.147.05.run
```



## 安装cuda

```bash
# 查看系统信息和架构信息。
cat /etc/os-release
uname -m

# 在Nvidia官网下载对应的 cuda toolkit 。
wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda_11.8.0_520.61.05_linux.run
sudo sh cuda_11.8.0_520.61.05_linux.run
# 安装的过程中 取消 驱动 ，单纯安装 cudatoolkit 就够了。

# 加入 PATH 和 LD_LIBRARY_PATH
echo 'export PATH=/usr/local/cuda-11.8/bin:$PATH' | sudo tee -a /etc/profile
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH' | sudo tee -a /etc/profile
source /etc/profile

# 测试
nvcc --version
# ———————————————————————————————— nvcc --version ——————————————————————————————————
[root@g01n01 local]# nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2022 NVIDIA Corporation
Built on Wed_Sep_21_10:33:58_PDT_2022
Cuda compilation tools, release 11.8, V11.8.89
Build cuda_11.8.r11.8/compiler.31833905_0
# ———————————————————————————————— nvcc --version ——————————————————————————————————
```



## centos装docker

```bash
# 设置docker yum源（二选一）设置为阿里云的源速度可以快一点（推荐）
sudo yum install -y yum-utils
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 下载安装
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# 启动docker，设置开机自启动
sudo systemctl start docker
sudo systemctl enable docker

# 编辑配置文件
sudo vi /etc/docker/daemon.json
# 添加以下内容
{
    "data-root": "/data/docker",
    "registry-mirrors": ["https://docker.m.daocloud.io","https://hub.daocloud.io"]
}


sudo systemctl daemon-reload
sudo systemctl restart docker

# 查看
docker -v
docker info
```

### 配置 Docker 自定义根目录

Docker 默认把所有数据存在 `/var/lib/docker`。我们要改成 `/data/docker`。

```bash
# 1. 停止 Docker
systemctl stop docker

# 2. 备份旧数据（如果已有重要容器，先备份！）
# （你刚装 Docker，应该没数据，可跳过）

# 3. 创建或编辑 daemon.json
cat > /etc/docker/daemon.json <<EOF
{
  "data-root": "/data/docker"
}
EOF

# 4. 重启 Docker
systemctl restart docker

# 验证，应输出：Docker Root Dir: /data/docker，
# 这就证明docker 的默认存储都在 这个14T的盘里面了。
docker info | grep "Docker Root Dir"

# 测试拉取
docker pull busybox
docker pull nvidia/cuda:12.0.1-base-ubuntu22.04
# 如果是实在是拉不到可能是 学校机房防火墙拦住了，我们只能在别的机器Download → Save → SCP → Load 

```

> 从此以后，**所有镜像、容器、卷、构建缓存**都会存到 13.1TB 盘上！ 



## 安装 NVIDIA Container Toolkit

（CentOS / RHEL 官方方法）

 添加 NVIDIA repo：

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
sudo yum-config-manager --add-repo \
    https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.repo
```

 安装：

```bash
sudo yum install -y nvidia-container-toolkit
```

配置 Docker 使其支持 GPU：

```
sudo nvidia-ctk runtime configure --runtime=docker
```

它会自动写入 `/etc/docker/daemon.json`：

例如：

```
{
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

重启 Docker：

```
sudo systemctl restart docker
```

测试：

```bash
# 官方镜像标签规则为 "主版本-子版本-基础环境"
# 拉取 CUDA 12.0.1 开发镜像
docker pull nvidia/cuda:12.0.1-devel-ubuntu22.04

# 检查测试，运行一个临时容器，显示应该是和 宿主机 一模一样才OK。
docker run --rm --gpus all nvidia/cuda:12.0.1-devel-ubuntu22.04 nvidia-smi

docker run --rm --gpus all nvidia/cuda:12.0.1-devel-ubuntu22.04 nvcc --version

# 可以额外测试。
# ---------------------------  测试  ---------------------------
docker run --rm --gpus all nvidia/cuda:12.0.1-devel-ubuntu22.04 /bin/bash -c "\
echo '#include <stdio.h>
#include <cuda_runtime.h>

__global__ void add(int *a, int *b, int *c, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if(i < N) c[i] = a[i] + b[i];
}

int main() {
    int N = 10;
    int a[N], b[N], c[N];
    for(int i=0;i<N;i++){ a[i]=i; b[i]=i*2; }

    int *d_a, *d_b, *d_c;
    cudaMalloc(&d_a, N*sizeof(int));
    cudaMalloc(&d_b, N*sizeof(int));
    cudaMalloc(&d_c, N*sizeof(int));

    cudaMemcpy(d_a, a, N*sizeof(int), cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, b, N*sizeof(int), cudaMemcpyHostToDevice);

    add<<<1, N>>>(d_a, d_b, d_c, N);

    cudaMemcpy(c, d_c, N*sizeof(int), cudaMemcpyDeviceToHost);

    printf(\"Vector addition result: \");
    for(int i=0;i<N;i++) printf(\"%d \", c[i]);
    printf(\"\\n\");

    cudaFree(d_a);
    cudaFree(d_b);
    cudaFree(d_c);

    int nDevices;
    cudaGetDeviceCount(&nDevices);
    printf(\"Number of CUDA devices: %d\\n\", nDevices);

    return 0;
}' > test_vector.cu && \
nvcc test_vector.cu -o test_vector && ./test_vector"
# ---------------------------  测试  ---------------------------
```



# **构建镜像**

以下操作都在 宿主机的 /data/gpu-lab/ 文件夹下进行。

## 创建启动脚本 `startJupyter.sh`

```bash
#!/bin/bash

# 检查是否设置了 USER_PASSWORD 环境变量
if [ -z "$USER_PASSWORD" ]; then
    echo "Error: USER_PASSWORD environment variable is not set."
    exit 1
fi

# 使用 Python 计算密码的 Hash 值
# 注意：Pytorch 镜像通常包含 jupyter_server 或 notebook 库
PASSWORD_HASH=$(python -c "from jupyter_server.auth import passwd; print(passwd('$USER_PASSWORD'))")

echo "Starting Jupyter Lab with user-defined password..."

# 启动 Jupyter Lab
# --NotebookApp.password: 使用哈希后的密码
# --NotebookApp.token='': 禁用 token，只准用密码登录
exec jupyter lab \
    --ip=0.0.0.0 \
    --port=8888 \
    --no-browser \
    --allow-root \
    --NotebookApp.token='' \
    --NotebookApp.password="$PASSWORD_HASH"
```

## 编辑 Dockerfile

查看自己的`nvidia-smi` 选好自己需要的pytorch 版本。

``` bash
 # 我的驱动
 # Driver Version: 525.147.05   CUDA Version: 12.0   
 # 我的cudatoolkit、nvcc
 # Cuda compilation tools, release 11.8, V11.8.89
```

打开 https://hub.docker.com/ 

搜索 pytorch，查找 pytorch 的docker镜像，点击 Tags 选版本（cuda11.8）。火得一批。

因为：（PyTorch 2.6 基本只提供 CUDA 12.1+ 镜像）

所以：我们选老一点的 镜像版本。

```bash
# 我选择的是 2.3.0-cuda11.8-cudnn8-devel
docker pull pytorch/pytorch:2.3.0-cuda11.8-cudnn8-devel
```

`Dockerfile`

```yaml
# 使用官方 PyTorch 镜像作为基础 (包含 CUDA, CuDNN, Conda, Python)
# 这里的标签可以根据你需要版本进行更换，例如 pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime 
FROM pytorch/pytorch:2.3.0-cuda11.8-cudnn8-devel

# 设置工作目录
WORKDIR /workspace

# 避免交互式安装时的卡顿
ENV DEBIAN_FRONTEND=noninteractive

# 更新系统并安装一些基础工具 (可选，根据需求添加)
RUN apt-get update && apt-get install -y \
    git \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*

# 安装 JupyterLab 和其他常用包
# 也可以在这里安装 matplotlib, pandas, scikit-learn 等
RUN pip install jupyterlab matplotlib pandas

# 暴露 Jupyter 的默认端口
EXPOSE 8888

# --- jupyter部分 ---
# 1. 把本地的 start.sh 复制到镜像里
COPY startJupyter.sh /usr/local/bin/startJupyter.sh

# 2. 给脚本赋予执行权限
RUN chmod +x /usr/local/bin/startJupyter.sh

# 3. 设置这个脚本为容器启动时的入口
CMD ["/usr/local/bin/startJupyter.sh"]
# ------------------
```

## 重新构建镜像

就是把 Dockerfile 所在文件夹下的文件 构建成一个 镜像 `my-gpu-lab:v1`。

```bash
docker build -t my-gpu-lab:v1 .

# 查看镜像
[root@g01n01 gpu-lab]$ docker images
REPOSITORY    TAG                        IMAGE ID       CREATED         SIZE
my-gpu-lab    v1                         9dc6a356894f   3 hours ago     18.2GB
busybox       latest                     08ef35a1c3f0   14 months ago   4.43MB
nvidia/cuda   12.0.1-devel-ubuntu22.04   678edd19cf8f   2 years ago     7.34GB
nvidia/cuda   12.0.1-base-ubuntu22.04    c41603a9ca98   2 years ago     237MB
```



# **规划资源**

## 规划宿主机目录（文件隔离）

```bash
# 在宿主机建立数据根目录
cd /data/gpu-lab

# 为4个用户创建各自的目录
sudo mkdir -p /data/gpu-lab/user1
sudo mkdir -p /data/gpu-lab/user2
sudo mkdir -p /data/gpu-lab/user3
sudo mkdir -p /data/gpu-lab/user4

# 修改权限，确保容器内可以读写 (简单粗暴给777，或者根据UID设置)
sudo chmod -R 777 /data/gpu-lab
```

## 限制每个容器的内存及CPU使用

Docker 原生支持资源限制，非常简单。

```bash
[root@g01n01 gpu-lab]# free -h
              total        used        free      shared  buff/cache   available
Mem:           502G         17G        453G         44M         31G        483G
Swap:            0B          0B          0B

[root@g01n01 gpu-lab]# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                48
On-line CPU(s) list:   0-47
Thread(s) per core:    1
Core(s) per socket:    24
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 85
Model name:            Intel(R) Xeon(R) Gold 5220R CPU @ 2.20GHz
```

修改你之前的 `docker-compose.yml`，为每个服务加上 `mem_limit`：502G 内存，4 个用户，每个分配 **64G~96G** 比较合理，留一些给系统。



## 编辑`docker-compose.yml`文件

```bash
docker compose version
Docker Compose version v2.27.1
```

根据这个docker compose 版本让大模型生成yaml文件。

```yaml
version: '3.8'

services:
  user1:
    image: my-gpu-lab:v1
    container_name: lab-user1
    restart: always
    ports:
      - "8881:8888"
    volumes:
      - /data/gpu-lab/user1:/workspace
    environment:
      - USER_PASSWORD=User1_Is_The_Best   # <--- 在这里设置 User 1 的明文密码
    shm_size: "8gb"
    deploy:
      resources:
        limits:
          cpus: '11'
          memory: 96G
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: ['gpu']

  user2:
    image: my-gpu-lab:v1
    container_name: lab-user2
    restart: always
    ports:
      - "8882:8888"
    volumes:
      - /data/gpu-lab/user2:/workspace
    environment:
      - USER_PASSWORD=User2_Is_The_Best   # <--- 在这里设置 User 2 的明文密码
    shm_size: "8gb"
    deploy:
      resources:
        limits:
          cpus: '11'
          memory: 96G
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['1']
              capabilities: ['gpu']
              
  user3:
    image: my-gpu-lab:v1
    container_name: lab-user3
    restart: always
    ports:
      - "8883:8888"
    volumes:
      - /data/gpu-lab/user3:/workspace
    environment:
      - USER_PASSWORD=User3_Is_The_Best   # <--- 在这里设置 User 3 的明文密码
    shm_size: "8gb"
    deploy:
      resources:
        limits:
          cpus: '11'
          memory: 96G
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['2']
              capabilities: ['gpu']

  user4:
    image: my-gpu-lab:v1
    container_name: lab-user4
    restart: always
    ports:
      - "8884:8888"
    volumes:
      - /data/gpu-lab/user4:/workspace
    environment:
      - USER_PASSWORD=User4_Is_The_Best   # <--- 在这里设置 User 4 的明文密码
    shm_size: "8gb"
    deploy:
      resources:
        limits:
          cpus: '11'
          memory: 96G
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['3']
              capabilities: ['gpu']
```



# **启动容器**

```bash
docker compose up -d
```

进来以后，会发现 这个 jupyter 的页面中的 terminal 和宿主机的会有不一样。

```bash
# pwd
/workspace
# ls
Untitled.ipynb
# conda env list 
# conda environments:
#
base                     /opt/conda
py3.10                   /opt/conda/envs/py3.10
```

我们需要先执行初始化conda，才能用conda。

```bash
# 执行初始化：
. /opt/conda/etc/profile.d/conda.sh

# 激活环境： 现在 Shell 已经认识 conda 命令了，可以正常激活环境了。
conda activate py3.10
```

如果一样要让在jupyter的web页面可选 新环境，和以前操作一样。

```bash
conda activate XXX  # 激活conda中的虚拟环境
conda install ipykernel  # 安装丘比特核心。

# 把丘比特核心和conda 建立的环境关联上。
python -m ipykernel install --user --name py3.10 --display-name "Py3.10 (conda)"


# 多加了或者 虚拟环境已经没有了，删除丘比特和虚拟环境的关联
jupyter kernelspec uninstall python3.7 
# 看关联内核
jupyter kernelspec list 
```



# 传输文件

1、通过 jupyter web页面 拖拽 ；

2、\# 在您的本地电脑上运行 :

`scp -P 22 /path/to/local/my_data.zip username@172.26.1.20:/data/gpu-lab/user4/`

传输完成后，文件会立即出现在您 Jupyter Terminal 中的 `/workspace` 目录下。















