FROM mcr.microsoft.com/devcontainers/cpp:0-debian-11

# 1. 换源：使用阿里云镜像源 (速度快)
RUN sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list && \
    sed -i 's/security.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list

# 2. 安装所有依赖 (包括 Clangd)
#    注意：删除了所有 COPY xxx.sh 的命令，全部改用 apt 安装，稳定且不会缺文件
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends \
    curl \
    build-essential \
    git \
    gdb \
    cmake \
    protobuf-compiler \
    libprotobuf-dev \
    libtinyxml-dev \
    clangd \
    && rm -rf /var/lib/apt/lists/*

# 3. 设置用户
USER vscode