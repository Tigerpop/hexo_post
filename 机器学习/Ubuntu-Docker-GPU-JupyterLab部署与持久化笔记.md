---
layout: posts
title: Ubuntu 部署独立 GPU JupyterLab：Docker Compose 与 Conda 持久化
date: 2026-10-9 16:55:27
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "GPU"
- "docker"



---



# Ubuntu 部署独立 GPU JupyterLab：Docker Compose 与 Conda 持久化

> 用途：保留已有学生的本地 JupyterLab，为另一名学生新增独立的 Docker JupyterLab；支持共享 GPU，并在容器重建后保留工作文件和新建 Conda 环境。
>
> 本文根据一次已完成部署的聊天整理。主流程采用最终配置，已合并下载超时的修复，去掉中途废弃方案。额外补充了环境变量一致性、内核注册、验证与日常维护。命令供目标 Ubuntu 机器执行，整理期间没有实际操作该服务器。

## 1. 最终方案与适用范围

甲继续通过宿主机的 `8888` 使用原来的 Conda / JupyterLab；乙通过 `8889` 使用独立容器 `lab-zf`。两人共享第 0 张 GPU，乙只能通过本方案的挂载访问自己的宿主机数据目录。

这里真正要解决的是**环境隔离和数据持久化**。单纯换一个端口，无法隔离同一系统账号下的文件和 Python 环境。当前人数较少，使用每人一个容器即可，不必把 JupyterHub 引入本次部署。

### 原机器的已知条件

| 项目 | 原聊天中的值 |
|---|---|
| 宿主机系统 | Ubuntu 24.04.1 LTS，x86_64 |
| 宿主机账号 | `cys`，UID/GID 均为 `1000` |
| 服务器内网 IP | `10.5.9.253` |
| GPU | NVIDIA GeForce RTX 3090，24GB 显存 |
| NVIDIA 驱动 | `550.120`，`nvidia-smi` 显示 CUDA 12.4 |
| Docker / Compose | Docker 28.0.2 / Compose v2.34.0 |
| GPU 容器运行条件 | 已有 NVIDIA runtime，测试容器运行 `nvidia-smi` 成功 |
| CPU / 内存 | 24 个逻辑 CPU，约 251GiB 内存 |
| Docker 存储目录 | `/var/lib/docker` |

本文不是从零安装 Ubuntu、Docker 或显卡驱动的教程。若换机器，先检查上述条件，尤其是 UID/GID、IP、GPU 通路和磁盘空间。

### 最终部署参数

| 项目 | 配置 |
|---|---|
| 配置文件目录 | `/home/cys/docker/gpu-lab-zf` |
| 乙的持久化数据目录 | `/home/cys/zf-data` |
| 容器挂载目录 | `/workspace` |
| Compose 服务名 | `zf` |
| Docker 容器名 | `lab-zf` |
| 自建镜像 | `zf-gpu-lab:v1` |
| 基础镜像 | `pytorch/pytorch:2.5.1-cuda12.4-cudnn9-devel` |
| 浏览器入口 | `http://10.5.9.253:8889/lab` |
| CPU / 内存上限 | 8 个 CPU 的计算额度 / 32GB 内存 |
| 共享内存 | 2GB |
| GPU | 第 0 张，与甲共享 |

`8 CPU` 是计算额度，不是固定绑定 8 个物理核心；CPU、内存限制不会给 GPU 显存分区。两人同时训练仍可能变慢或显存不足，需要约定用量或错峰运行。

## 2. 先理解三个关键点

### 2.1 宿主机不用额外安装 CUDA Toolkit

本次宿主机驱动和 NVIDIA Container Toolkit 已正常工作。CUDA 开发工具由容器内的 `devel` 镜像提供，不需要照搬旧 CentOS 笔记安装宿主机 CUDA 11.8。

- NVIDIA 驱动：宿主机使用显卡的基础。
- NVIDIA Container Toolkit：让容器访问宿主机 GPU。
- CUDA Toolkit：提供 `nvcc` 等编译工具，本方案放在镜像内。

`nvidia-smi` 中的 CUDA Version 是驱动支持能力的标识，不能据此认定宿主机已安装相同版本的 Toolkit。容器系统版本也不必与宿主机完全相同。GPU 容器的前置配置参考 [NVIDIA 官方文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)。

### 2.2 宿主机 8889，容器内仍是 8888

```text
浏览器访问 10.5.9.253:8889
              ↓ Docker 端口映射
容器 lab-zf 内的 JupyterLab :8888
```

Compose 中的 `10.5.9.253:8889:8888` 表示“宿主机指定 IP : 宿主机端口 : 容器端口”。因此启动脚本保留 `--port=8888` 是正确的，不会占用甲的宿主机 `8888`。

### 2.3 持久化的关键是文件实际存放位置

```text
/home/cys/docker/gpu-lab-zf/       # 部署配置
├── Dockerfile
├── startJupyter.sh
└── docker-compose.yml

/home/cys/zf-data/                # 挂载为容器 /workspace
├── 项目文件、Notebook、数据集等
├── conda_envs/                   # 新建 Conda 环境
├── .conda_pkgs/                  # Conda 包缓存
├── .condarc                     # 写入 Conda 配置后才可能出现
└── .home/                       # 用户家目录
    ├── .bashrc
    ├── .jupyter/                # Jupyter 配置
    └── .local/                  # 用户级内核注册等
```

仅挂载工作文件夹，却把环境安装到 `/opt/conda/envs`，无法保证环境在重建后保留。本方案把环境、缓存和用户配置一起放入 `/workspace`。相关配置项见 [Conda 官方配置说明](https://docs.conda.io/projects/conda/en/latest/user-guide/configuration/settings.html)。

**镜像内的 base 环境仍在 `/opt/conda`，不属于持久化目录。** 项目依赖应装进新建环境；公共基础依赖写进 Dockerfile。显式使用 `conda create -p` 时，也应把路径放在 `/workspace` 下。

`docker compose up -d` 并非每次都清空环境；配置或镜像变化导致容器被重建时，旧容器可写层里的文件才会丢失。仅做普通 restart，不能证明重建后的持久化有效。持久化也不等于备份，误删挂载目录中的文件会同步影响宿主机文件。

## 3. 部署前检查

以下命令在 **Ubuntu 宿主机的 Bash 终端**执行，以 `cys` 登录。后文只有明确标注的命令才在容器或 Notebook 内执行。

```bash
cat /etc/os-release
uname -m
id
nvidia-smi
sudo docker info
sudo docker compose version
ip -br addr
free -h
nproc
df -h /
sudo docker system df
```

确认 `10.5.9.253` 属于当前服务器。换机器时修改后文 Compose 的 IP；不要直接照抄一个不存在的地址。

测试 Docker GPU 通路：

```bash
sudo docker run --rm --gpus all \
  nvidia/cuda:12.4.1-base-ubuntu22.04 \
  nvidia-smi
```

应能看到 RTX 3090。原聊天已经通过该测试。若这里失败，先解决驱动、容器运行时或镜像下载问题，再继续。原机器不需要重装驱动或重启 Docker 服务。

创建目录并检查权限与端口：

```bash
mkdir -p /home/cys/docker/gpu-lab-zf
mkdir -p /home/cys/zf-data
cd /home/cys/docker/gpu-lab-zf

id
ls -ld /home/cys/docker/gpu-lab-zf /home/cys/zf-data
test -w /home/cys/zf-data && echo 'zf-data 可写' || echo 'zf-data 不可写'
sudo ss -ltnp 'sport = :8889'
```

最后一条没有监听记录，表示端口空闲。下面 Dockerfile 使用 UID/GID `1000:1000`，与原机器 `cys` 一致；其他机器必须相应调整。不要用 `chmod -R 777` 掩盖权限问题。

首次下载、解压 `devel` 镜像和构建会占用较多空间。原聊天是在基础镜像已下载的情况下，清理后剩余约 23GB 才继续构建成功；这个数字不是全新部署的容量保证，还应为学生数据、环境和缓存留空间。

## 4. 创建三个配置文件

以下写文件命令适用于首次部署；已有配置时先备份再覆盖。三个文件都放在 `/home/cys/docker/gpu-lab-zf`。

### 4.1 创建 startJupyter.sh

保留原来的脚本名，方便与旧笔记对应。脚本负责指定持久化路径、初始化 Bash 的 Conda 支持、生成密码哈希并启动 JupyterLab。

```bash
cat > startJupyter.sh <<'EOF'
#!/bin/bash
set -euo pipefail

# 用户配置、新建 Conda 环境及包缓存均放在持久化目录
export HOME=/workspace/.home
export CONDARC=/workspace/.condarc
export CONDA_ENVS_PATH=/workspace/conda_envs
export CONDA_PKGS_DIRS=/workspace/.conda_pkgs

mkdir -p "$HOME" "$CONDA_ENVS_PATH" "$CONDA_PKGS_DIRS"

# 初始化终端中的 conda activate 功能
if [ ! -f "$HOME/.bashrc" ]; then
    cat > "$HOME/.bashrc" <<'BASHRC'
source /opt/conda/etc/profile.d/conda.sh
BASHRC
fi

# 从环境变量读取密码，避免特殊字符被当成 Python 代码
: "${USER_PASSWORD:?请设置 USER_PASSWORD}"

python - <<'PY'
import json
import os
from pathlib import Path
from jupyter_server.auth import passwd

config_dir = Path.home() / ".jupyter"
config_dir.mkdir(parents=True, exist_ok=True)

config = {
    "PasswordIdentityProvider": {
        "hashed_password": passwd(os.environ["USER_PASSWORD"]),
        "password_required": True,
    },
    "IdentityProvider": {
        "token": "",
    },
}

config_file = config_dir / "jupyter_server_config.json"
config_file.write_text(json.dumps(config), encoding="utf-8")
config_file.chmod(0o600)
PY

unset USER_PASSWORD

exec jupyter lab \
    --ip=0.0.0.0 \
    --port=8888 \
    --no-browser \
    --ServerApp.root_dir=/workspace \
    --ServerApp.port_retries=0
EOF
```

这里从环境变量读取密码，而非把密码拼进 Python 源代码，可避免引号等特殊字符破坏脚本。密码配置文件权限设为 `600`，以普通用户启动 Jupyter，不使用 `--allow-root`。

密码模式下关闭 token，但保留密码认证。此处使用 Jupyter Server 的身份提供器配置，相关说明见 [Jupyter Server 官方文档](https://jupyter-server.readthedocs.io/en/latest/operators/public-server.html)。脚本每次启动会重新写入这份 JSON，因此需要长期保留的额外配置也应纳入脚本。

### 4.2 创建 Dockerfile

已把聊天最后的 PyPI 镜像源、超时和重试参数合并进来，无需先失败再执行 `sed` 修补。

**整理时补充的改进：** 在 Dockerfile 中同时声明 Conda 路径。原启动脚本的 `export` 会被 Jupyter 和它启动的终端继承，但 `docker exec` 新进程不自动继承启动脚本修改过的环境；写入镜像环境后，两种入口使用一致的路径。

```bash
cat > Dockerfile <<'EOF'
FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-devel

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends git curl vim \
    && rm -rf /var/lib/apt/lists/*

RUN python -m pip install --no-cache-dir --timeout 120 --retries 5 \
    -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple \
    "jupyterlab>=4,<5" ipykernel matplotlib pandas scikit-learn

# 与宿主机 cys 的 UID/GID 一致，保证挂载目录可写
RUN groupadd -g 1000 student \
    && useradd -m -u 1000 -g student -s /bin/bash student

COPY startJupyter.sh /usr/local/bin/startJupyter.sh
RUN chmod 755 /usr/local/bin/startJupyter.sh

ENV HOME=/workspace/.home
ENV CONDARC=/workspace/.condarc
ENV CONDA_ENVS_PATH=/workspace/conda_envs
ENV CONDA_PKGS_DIRS=/workspace/.conda_pkgs
ENV PATH=/workspace/.home/.local/bin:${PATH}

WORKDIR /workspace
USER student

EXPOSE 8888
CMD ["/usr/local/bin/startJupyter.sh"]
EOF
```

选 `devel` 是为了保留 CUDA 扩展编译能力；它比 `runtime` 大。此处沿用原聊天已成功构建的版本，不把升级基础镜像混入复现步骤。JupyterLab 等依赖没有逐项锁定补丁版本，因此日后重新构建不保证与当时完全一致；需要严格复现时，应保存已构建镜像并记录依赖版本。

### 4.3 创建 docker-compose.yml

```bash
cd /home/cys/docker/gpu-lab-zf

cat > docker-compose.yml <<'EOF'
services:
  zf:
    image: zf-gpu-lab:v1
    container_name: lab-zf
    restart: unless-stopped
    ports:
      - "10.5.9.253:8889:8888"
    volumes:
      - /home/cys/zf-data:/workspace
    environment:
      USER_PASSWORD: "${USER_PASSWORD:?请先设置登录密码}"
    cpus: 8
    mem_limit: 32g
    shm_size: "2gb"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [gpu]
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
EOF
```

`capabilities: [gpu]` 与 `device_ids: ['0']` 为服务指定可见 GPU，符合 [Docker Compose GPU 配置方式](https://docs.docker.com/compose/how-tos/gpu-support/)。`restart: unless-stopped` 让容器在异常退出或 Docker 恢复后按策略启动；手动停止的容器不会因此自动恢复。

密码由启动时环境变量传入，不写进 YAML。但环境变量仍保存在 Docker 容器配置中，拥有 Docker 管理权限的人可以读取；`unset` 只清理对应进程环境，不会抹除 Docker 元数据中的密码。

## 5. 构建、启动、登录

### 5.1 构建镜像

在宿主机执行：

```bash
cd /home/cys/docker/gpu-lab-zf
sudo docker build -t zf-gpu-lab:v1 .
```

末尾的 `.` 表示当前目录是构建上下文。看到构建完成并命名为 `zf-gpu-lab:v1` 后再启动。首次构建可能较慢，下载进度仍在增加就继续等待；失败后可复用已完成步骤的缓存。

### 5.2 输入密码并启动

以下整段在同一个 Bash 会话执行。密码输入时不回显。

```bash
cd /home/cys/docker/gpu-lab-zf
read -r -s -p '请输入乙的 Jupyter 登录密码: ' USER_PASSWORD
echo
export USER_PASSWORD

sudo --preserve-env=USER_PASSWORD docker compose up -d zf
unset USER_PASSWORD

sudo docker ps --filter name=lab-zf
sudo docker logs --tail=60 lab-zf
```

此处故意使用 `docker ps` 和 `docker logs` 查看状态，而不是 Compose 命令：Compose 读取 YAML 时会再次解析 `${USER_PASSWORD:?...}`，清除变量后会报错，详见排错部分。

### 5.3 浏览器登录

打开：

```text
http://10.5.9.253:8889/lab
```

使用刚才输入的密码。原方案是内网 HTTP 访问；HTTP 不加密密码与会话流量。需要跨不可信网络使用时，应另行配置 HTTPS、VPN 或 SSH 隧道，而不是直接开放公网端口。

## 6. 验证与学生实际使用（补充）

原聊天明确记录了 GPU 测试容器成功、镜像构建成功、`lab-zf` 启动成功，用户最后反馈“好了”。未见最终 Notebook CUDA 运算和强制重建验证的完整输出。以下用于复现后自行验收，不代表原聊天已经逐项执行。

### 6.1 在 Notebook 中验证 PyTorch GPU

先使用镜像自带的 Python 内核运行：

```python
import sys
import torch

print("Python:", sys.executable)
print("PyTorch:", torch.__version__)
print("PyTorch CUDA:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
    x = torch.ones(3, device="cuda")
    print(x)
```

预期 CUDA 可用、显卡为 RTX 3090，并能生成 CUDA 张量。容器中 `nvidia-smi` 成功只证明显卡通路正常，不能替代这里的框架验证。

### 6.2 新建持久化 Conda 环境，并注册 Notebook 内核

在 **JupyterLab 的 Terminal** 中执行：

```bash
source /opt/conda/etc/profile.d/conda.sh

# 示例环境名 zf-py310；如已有同名环境，不重复创建
conda create -n zf-py310 python=3.10 -y
conda activate zf-py310

python -m pip install --timeout 120 --retries 5 \
  -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple \
  ipykernel

python -m ipykernel install --user \
  --name zf-py310 \
  --display-name 'Python (zf-py310)'

conda env list
python -c 'import sys; print(sys.executable)'
```

路径应是 `/workspace/conda_envs/zf-py310/bin/python`。随后在 Notebook 中切换到 `Python (zf-py310)`，运行 `import sys; print(sys.executable)`，确认选对环境。必要时刷新 JupyterLab 页面。

**新环境不会自动继承 base 的 PyTorch。** 上面只安装了 Python 和 ipykernel；深度学习依赖需要按项目要求另外安装。不要把新环境没有 `torch` 误判为 GPU 失效。

### 6.3 验证文件和环境在重建后仍存在

先在 JupyterLab Terminal 中创建标记，并记录环境列表：

```bash
printf 'persistence check\n' > /workspace/persistence-check.txt
conda env list
```

确认乙没有正在执行的训练或 Notebook 任务后，在 **宿主机**重建乙的容器。此操作会中断乙的运行中进程：

```bash
cd /home/cys/docker/gpu-lab-zf
read -r -s -p '请输入乙当前的 Jupyter 登录密码: ' USER_PASSWORD
echo
export USER_PASSWORD
sudo --preserve-env=USER_PASSWORD docker compose up -d --no-deps --force-recreate zf
unset USER_PASSWORD

sudo docker logs --tail=60 lab-zf
sudo docker exec lab-zf cat /workspace/persistence-check.txt
sudo docker exec lab-zf /opt/conda/bin/conda env list
sudo docker exec lab-zf /workspace/conda_envs/zf-py310/bin/python --version
```

再登录并确认自建内核可用。验证成功意味着挂载目录中的文件和环境被保留；运行中的计算进度、未保存内容和内存状态不会因文件持久化而自动恢复。

## 7. 日常运维速查

### 状态、日志和终端

在宿主机执行，无需重新输入 Jupyter 密码：

```bash
sudo docker ps -a --filter name=lab-zf
sudo docker logs --tail=100 lab-zf
sudo docker stats --no-stream lab-zf
sudo docker exec -it lab-zf bash
```

进入容器后可执行 `source /opt/conda/etc/profile.d/conda.sh`、`conda env list`、`conda activate zf-py310`；执行 `exit` 回到宿主机。

### 停止、启动、重启

以下是独立操作，按需选择，不要把三条连着执行：

| 用途 | 宿主机命令 |
|---|---|
| 停止乙的服务 | `sudo docker stop lab-zf` |
| 启动已存在的容器 | `sudo docker start lab-zf` |
| 重启乙的容器 | `sudo docker restart lab-zf` |

这些命令沿用容器已保存的环境，不需要再设置 `USER_PASSWORD`。停止或重启会中断乙的任务；甲的本地 Jupyter 不属于这个容器。

### 修改密码或更新镜像

修改密码时，重新执行第 5.2 节，输入新密码；Compose 会根据环境配置变化更新容器。该过程应安排在乙没有任务时进行。仅在容器内手改生成的密码 JSON，下次启动会被脚本覆盖。

修改 Dockerfile 或 `startJupyter.sh` 后，必须重新构建镜像。建议使用新标签：

```bash
cd /home/cys/docker/gpu-lab-zf
sudo docker build -t zf-gpu-lab:v2 .
```

然后把 Compose 中 `image: zf-gpu-lab:v1` 改成 `image: zf-gpu-lab:v2`，输入密码并执行：

```bash
cd /home/cys/docker/gpu-lab-zf
read -r -s -p '请输入乙的 Jupyter 登录密码: ' USER_PASSWORD
echo
export USER_PASSWORD
sudo --preserve-env=USER_PASSWORD docker compose up -d --no-deps zf
unset USER_PASSWORD
sudo docker logs --tail=60 lab-zf
```

确认服务正常。不要只执行 `docker restart`，它不会让已有容器改用新镜像。多服务项目中，保留服务名 `zf` 可避免更新其他学生服务；不要为更新乙而执行整个项目的 `down`。

### 简单备份

需要备份的是两部分：部署配置目录，以及整个 `zf-data`（包含隐藏目录）。备份到另一块可靠存储；备份前让乙停止写入，避免环境正在装包或文件正在保存。仅保存 Notebook 不能恢复完整 Conda 环境与内核配置。

## 8. 本次遇到的问题与有效处理

### 8.1 apt 报 invalid signature，实际发现根分区已满

本次在安装 `git/curl/vim` 时，多个源同时提示：

```text
At least one invalid signature was encountered.
```

排查发现 `/` 可用空间为 0，inode 仅使用约 5%。磁盘写入失败很可能是此次异常的原因；不能把所有签名错误都归因于空间不足。

先检查，不关闭签名验证：

```bash
df -h /var/lib/docker
df -i /var/lib/docker
sudo docker system df
sudo du -xhd1 / 2>/dev/null | sort -h
sudo du -xhd1 /home/cys 2>/dev/null | sort -h
```

本次最终通过清理宿主机 Conda 缓存释放约 25GB，随后 `df -h /` 显示约 23GB 可用，再继续构建。以下在 **宿主机原有 Conda 可用的终端**执行；先看预览再决定是否清理：

```bash
conda clean --all --dry-run
conda clean --all

df -h /
du -sh /home/cys/miniconda3/pkgs
```

不要直接删除 `envs` 或整个 `pkgs`，不要使用 `--force-pkgs-dirs`。如果现有环境通过软链接引用缓存包，清理缓存也可能影响环境，应先确认。学生的训练结果、数据集和权重不应仅凭目录名删除。本次没有证据表明执行过实验目录删除或磁盘迁移。

### 8.2 换到另一个目录，并不等于换到另一块磁盘

原机器的 `/home/cys/data/docker-data` 和 `/home/cys/zf-data` 仍在同一个根分区。移动目录不会凭空增加容量。

```bash
df -h / /home/cys/data/docker-data /home/cys/zf-data
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,MODEL
```

配置文件放在哪里，与 Docker 镜像存在哪里也是两回事：`docker build` 的目录是构建上下文，镜像和容器层由 Docker Root Dir 管理；学生数据则由 bind mount 的宿主机路径决定。

### 8.3 pip 下载超时

本次出现：

```text
files.pythonhosted.org: Read timed out
```

有效处理是给 pip 指定清华源，并增加 `--timeout 120 --retries 5`。第 4.2 节已合并修复，重新运行构建即可，不需要再次套用聊天中的 `sed` 命令，也不必清空 Docker 缓存。

这个 PyPI 源只影响该次 pip 安装，不会同时改变 apt、Conda 或 Docker 镜像下载源。

### 8.4 容器已启动，但 Compose 查看状态提示缺少密码

报错：

```text
required variable USER_PASSWORD is missing a value
```

原因是 YAML 使用了必填变量表达式；执行 `unset USER_PASSWORD` 后，`docker compose ps` 或 `docker compose logs` 也会解析配置并报错。它不代表已经启动的容器失败。

直接查看：

```bash
sudo docker ps --filter name=lab-zf
sudo docker logs --tail=60 lab-zf
```

以后需要 `compose up`、`compose config` 等解析配置的操作，再在同一个 Bash 会话输入并导出密码，同时让 `sudo` 保留该变量。配置输出可能包含密码，不要随意公开。

### 8.5 其他快速定位

| 现象 | 先检查什么 |
|---|---|
| 容器启动后退出 | `sudo docker logs --tail=100 lab-zf`，检查密码和目录写权限 |
| 宿主机端口绑定失败 | IP 是否属于本机、8889 是否被占用 |
| 宿主机正常但客户端打不开 | Docker 端口映射、客户端到服务器的网络和防火墙；不要直接关闭整个防火墙 |
| `Permission denied` | 宿主机目录的数字 UID/GID 是否与容器 `1000:1000` 一致 |
| 自建环境重建后找不到 | 原环境是否实际在 `/workspace/conda_envs`，挂载源是否变了 |
| Notebook 没用到新环境 | 查看 `sys.executable`，检查是否注册并切换到对应内核 |
| `import torch` 失败 | 当前环境是否安装了 PyTorch；新 Conda 环境不会继承 base 的包 |
| CUDA 显存不足 | 查看 `nvidia-smi`，与甲协调任务；不要因为进程栏为空就重置 GPU |

## 9. 后续扩展的边界

### 新增学生

沿用同一结构，为每个人分配独立服务名、容器名、宿主机端口、数据目录和密码。镜像可以共用，数据目录不能误指向同一个位置。当前方案不是统一登录平台，人数明显增加后再考虑 JupyterHub。

容器内普通用户与宿主机 `cys` 数字 UID 一致，便于写挂载目录；这不是宿主机不同 Linux 用户间的严格权限隔离。不要把宿主机根目录或 Docker socket 挂入学生容器，也不要把宿主机 Docker 管理权限交给学生。

### 以后增加磁盘

新增磁盘只是提供容量，还需要确认设备、建立文件系统并可靠挂载，再迁移数据。本次没有完成这项操作，因此不提供可误执行的格式化命令。

迁移 `zf-data` 时：先停止乙的写入，完整复制数据和隐藏目录并保留权限，确认新盘挂载和副本完整，然后只修改 Compose 冒号左侧的宿主机路径。容器路径继续使用 `/workspace`，以保持 Conda 前缀和内核路径稳定。验证并备份后再处理旧副本。新盘还应配置开机挂载及启动依赖，避免磁盘未挂载时容器写入根分区的同名目录。

迁移 Docker Root Dir 是另一项工作，可能需要停止 Docker 并影响其他容器；不能与仅迁移学生数据混为一谈。

### 已有旧 Conda 环境

本次是新部署，不需要迁移旧容器环境。如果旧环境仍在旧容器可写层里，应在删除或重建前导出并验证迁移。不要直接照搬旧笔记的 `cp -r /opt/conda/envs/* ...`：环境可能包含绝对路径或前缀绑定。可按条件使用环境导出后重建或专门迁移工具，再验证项目依赖和 Notebook 内核。

## 10. 下次操作时的最短导航

- **首次部署：** 检查 GPU、目录和空间 → 创建三个配置文件 → 构建 → 输入密码启动 → 浏览器登录 → 验证 GPU 与持久化。
- **日常查看：** `docker ps`、`docker logs`，无需重新输入密码。
- **创建项目环境：** 在容器终端创建 → 确认路径在 `/workspace` → 注册 ipykernel → Notebook 选择对应内核。
- **更新配置或镜像：** 保存任务 → 备份 → 重新构建（如有必要）→ 输入密码 → 仅更新 `zf` → 验证。

原始记录：[Jupyter 多用户方案建议](https://chatgpt.com/share/6ac8a44a-55b4-83e9-888b-9f00d2da45e2)。本文保留最终有效主线，未沿用早期 runtime/token 方案、旧 CentOS 路径、777 权限设置或直接复制旧 Conda 环境的做法。
