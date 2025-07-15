---
layout: post
title: Codalab配置教程
date: 2025-07-15 00:40:16
description: Codalab服务器和客户端配置
tags: 工具
categories: tools
pretty_table: true
giscus_comments: false
toc:
  sidebar: right
---
> Codalab可以方便地管理在多台服务器的运行实验，多台服务器的实验结果可以汇总到网页，并通过表格的形式呈现。
> 同时也可以将实验的全流程公开，便于复现。

### 1. 服务器Worker配置
---

#### 1.1 环境依赖

```shell
# 1. 安装yum-utils
yum install -y yum-utils
# 2. 配置docker镜像
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# 3. 查看docker-ce版本
yum list docker-ce --showduplicates | sort -r 
# 4. docker版本不能超过19, 选择18.03
yum install docker-ce-18.03.1.ce-1.el7.centos docker-ce-cli-18.03.1.ce-1.el7.centos containerd.io docker-buildx-plugin docker-compose-plugin
# 5. 测试docker是否正确安装
sudo systemctl start docker
sudo docker run hello-world
# 6. 配置nvidia-docker仓库
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
# 7. 安装nvidia-docker
sudo yum install -y nvidia-docker2
# 8. docker开机自启动，重启docker
systemctl enable docker.service
systemctl restart docker
# 9. 验证gpu是否可用,网址列表在这里: https://catalog.ngc.nvidia.com/orgs/nvidia/containers/cuda,对于11.7版本的cuda,选择
docker run --runtime=nvidia --rm nvidia/cuda:11.7.1-runtime-centos7 nvidia-smi
```

#### 1.2 worker配置

```shell
# 1. 安装codalab
pip install codalab
# 2. 启动worker,并且输入用户名和密码
# --verbose 输出日志
# --tag-exclusive 只接受--request-queue <worker-tag>
# --tag 指定<worker-tag>
# --work-dir 指定数据存放地址
# --cpuset 指定cpu数量
# --gpuset 指定gpu数量
# 目前只在210.28.133.13:21678中开启了worker
cl-worker --tag 21678 --tag-exclusive --work-dir /root/tz/codalab --cpuset "0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20" --gpuset "0,1,2,3" --verbose
# 3. 测试worker运行
cl run date
# 4. 测试在codalab中调用nvidia docker
cl run --request-docker-image nvidia/cuda:11.7.1-runtime-centos7 --request-gpus 1 "nvidia-smi"
```
---

### 2. Docker镜像打包

需要的文件:

- [x] [requirements.txt](/assets/codes/codalab中Docker镜像配置文件/requirements.txt)
- [x] [Dockerfile](/assets/codes/codalab中Docker镜像配置文件/Dockerfile)
- [ ] cached_hf_models ## 模型缓存，解决huggingface被墙问题
- [x] [default_config.yaml](/assets/codes/codalab中Docker镜像配置文件/default_config.yaml) ## 配置accelerate

然后，使用以下的命令将镜像上传至dockerhub:

```shell
docker login -u zetang
docker build -t codecom .
docker image tag codecom zetang/codecom:latest
docker image push zetang/codecom:latest
```

在Python代码中加载模型时，可以使用以下代码进行替换：

```python
from huggingface_hub import snapshot_download
def get_hf_model(repo_name, cache="E:/cached_hf_models"):
    return snapshot_download(repo_name, cache_dir=cache)
AutoModel.from_pretrained(get_hf_model("model_name"))
```

测试镜像:
```shell
docker ps -a
```

---
### 3. 客户端运行
准备工作:

- [x] 客户端安装codalab
- [x] 上传代码和数据

通过以下脚本可以在worker中运行实验:

```shell
# This is an example script to start a CodaLab run. There are often several
# things to configure, including the docker image, compute resources, bundle
# dependencies (code and data), and custom arguments to pass to the command.
# Factoring all this into a script makes it easier to run and track different
# configurations.

### CodaLab arguments
CODALAB_ARGS="cl run"

# Name of bundle (can customize however you want)
CODALAB_ARGS="$CODALAB_ARGS --name bigcode-santacoder-Android"
# Docker image (default: codalab/default-cpu)
CODALAB_ARGS="$CODALAB_ARGS --request-docker-image zetang/code_com_llm:latest"
# Explicitly ask for a worker with at least one GPU
CODALAB_ARGS="$CODALAB_ARGS --request-gpus 4"
CODALAB_ARGS="$CODALAB_ARGS --request-cpus 10"
# Control the amount of RAM your run needs
CODALAB_ARGS="$CODALAB_ARGS --request-memory 128g"
# Kill job after this many days (default: 1 day)
CODALAB_ARGS="$CODALAB_ARGS --request-time 2d"
# Use own GPU
CODALAB_ARGS="$CODALAB_ARGS --request-queue 21678"

# Bundle dependencies
CODALAB_ARGS="$CODALAB_ARGS :src"      # Code
CODALAB_ARGS="$CODALAB_ARGS :data"                             # Dataset
# CODALAB_ARGS="$CODALAB_ARGS word-vectors.txt:glove.840B.300d"  # Word vectors

### Command to execute (these flags can be overridden) from the command-line
CMD="accelerate launch src/main.py"
CMD="$CMD --model bigcode/santacoder"
### task is Android line com
CMD="$CMD --tasks android"
CMD="$CMD --max_length_generation 512 --temperature 0.8 --precision fp16 --do_sample True --n_samples 1 --batch_size 1 --save_generations"
CMD="$CMD --save_generations_path android-generation.json"
CMD="$CMD --metric_output_path android-evaluation.json"

# Pass the command-line arguments through to override the above
if [ -n "$1" ]; then
  CMD="$CMD $@"
fi

# Create the run on CodaLab!
FINAL_COMMAND="$CODALAB_ARGS '$CMD'"
echo $FINAL_COMMAND
exec bash -c "$FINAL_COMMAND"

pause
```
---

#### 参考资料

1. [在centos上安装docker](https://docs.docker.com/engine/install/centos/) 
2. [在centos上安装nvidia-docker](https://github.com/keineahnung2345/Dockerfiles/blob/master/How%20to%20install%20Nvidia-Docker%202.0%20on%20CentOS%207.md)
3. [官网指南](https://codalab-worksheets.readthedocs.io/en/latest/examples/quickstart/)
4. [个人配置](https://worksheets.codalab.org/worksheets/0x17d24480c9cc42fa9489279bc1e7bb36)