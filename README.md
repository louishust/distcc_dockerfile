# DistCC Docker 镜像

提供三个 GCC 版本的 distcc 分布式编译服务镜像。

## 镜像列表

| 镜像 | GCC 版本 | 基础系统 |
|------|----------|----------|
| distcc-gcc10 | GCC/G++ 10 | Ubuntu 22.04 |
| distcc-gcc11 | GCC/G++ 11 | Ubuntu 22.04 |
| distcc-gcc12 | GCC/G++ 12 | Ubuntu 22.04 |

## 快速开始

### 1. 构建镜像

```bash
# 构建 GCC 10 版本
docker build -f Dockerfile.gcc10 -t distcc-gcc10 .

# 构建 GCC 11 版本
docker build -f Dockerfile.gcc11 -t distcc-gcc11 .

# 构建 GCC 12 版本
docker build -f Dockerfile.gcc12 -t distcc-gcc12 .
```

### 2. 运行容器

```bash
# 运行 GCC 12 版本
docker run -d -p 3632:3632 --name distcc-server distcc-gcc12
```

### 3. 客户端配置

在需要使用分布式编译的机器上配置环境变量：

```bash
# 设置 distcc 服务器地址
export DISTCC_HOSTS="192.168.1.100:3632"

# 设置编译器
export CC=gcc
export CXX=g++

# 使用 distcc 进行编译
make -j8
```

## 高级配置

### 多服务器负载均衡

```bash
# 多个服务器
export DISTCC_HOSTS="192.168.1.100:3632 192.168.1.101:3632 192.168.1.102:3632"
```

### 使用 pump 模式（推荐）

Pump 模式可以让 distcc 自动分发头文件，提高编译效率：

```bash
# 在服务器端已默认启用 pump 模式
# 客户端使用 pump 模式
distcc-pump make -j8
```

### 自定义并行任务数

默认配置为 4 个并行 jobs，可通过环境变量调整：

```bash
docker run -d -p 3632:3632 -e DISTCC_JOBS=8 distcc-gcc12
```

### 限制访问 IP

默认允许所有 IP 访问，生产环境建议限制：

```bash
docker run -d -p 3632:3632 distcc-gcc12 \
    --allow 192.168.1.0/24
```

## Docker Compose 示例

```yaml
version: '3.8'

services:
  distcc-gcc10:
    build:
      context: .
      dockerfile: Dockerfile.gcc10
    ports:
      - "3632:3632"
    restart: unless-stopped

  distcc-gcc11:
    build:
      context: .
      dockerfile: Dockerfile.gcc11
    ports:
      - "3633:3632"
    restart: unless-stopped

  distcc-gcc12:
    build:
      context: .
      dockerfile: Dockerfile.gcc12
    ports:
      - "3634:3632"
    restart: unless-stopped
```

## 镜像打包与迁移

在无法访问 Docker Hub 或内网环境中，可以将构建好的镜像打包成 tar 文件进行迁移。

### 导出镜像

```bash
# 导出镜像为 tar 文件
docker save -o distcc-gcc10.tar distcc-gcc10
docker save -o distcc-gcc11.tar distcc-gcc11
docker save -o distcc-gcc12.tar distcc-gcc12

# 也可以将多个镜像打包成一个文件
docker save -o distcc-all.tar distcc-gcc10 distcc-gcc11 distcc-gcc12
```

### 导入镜像

```bash
# 从 tar 文件加载镜像
docker load -i distcc-gcc12.tar

# 或者从标准输入导入
docker load < distcc-gcc12.tar
```

### 传输镜像

#### 方法一：使用 SCP

```bash
# 传输到其他机器
scp distcc-gcc12.tar user@192.168.1.100:~/
```

#### 方法二：使用 rsync（推荐大文件）

```bash
# 带进度显示传输
rsync -avP distcc-gcc12.tar user@192.168.1.100:~/
```

#### 方法三：构建私有镜像仓库

```bash
# 启动本地私有仓库
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# 标记镜像
docker tag distcc-gcc12 localhost:5000/distcc-gcc12

# 推送镜像
docker push localhost:5000/distcc-gcc12

# 在其他机器上拉取
docker pull 192.168.1.100:5000/distcc-gcc12
```

### 完整迁移流程示例

```bash
# 1. 在有网络的机器上构建并导出
git clone https://github.com/louishust/distcc_dockerfile.git
cd distcc_dockerfile
docker build -f Dockerfile.gcc12 -t distcc-gcc12 .
docker save -o distcc-gcc12.tar distcc-gcc12

# 2. 传输到目标机器
rsync -avP distcc-gcc12.tar user@192.168.1.100:~/

# 3. 在目标机器上导入并运行
docker load -i distcc-gcc12.tar
docker run -d -p 3632:3632 --name distcc-server distcc-gcc12
```

## 注意事项

1. **网络要求**：确保客户端与服务器之间的 3632 端口互通
2. **GCC 版本一致性**：客户端和服务器的 GCC 版本应保持一致，避免 ABI 兼容性问题
3. **安全性**：生产环境请限制 `--allow` 参数的 IP 范围
4. **性能**：建议使用千兆网络或更快的网络环境

## 故障排查

### 连接失败

```bash
# 检查容器是否运行
docker ps

# 查看日志
docker logs distcc-server

# 测试连接
telnet 192.168.1.100 3632
```

### 编译失败

```bash
# 检查 DISTCC 主机配置
echo $DISTCC_HOSTS

# 验证编译器版本
distcc --version
gcc --version
```

## 扩展阅读

- [DistCC 官方文档](https://distcc.github.io/)
- [GCC 官方文档](https://gcc.gnu.org/onlinedocs/)
