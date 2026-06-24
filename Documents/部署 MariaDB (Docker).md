这是一份在 Raspberry Pi 5 上使用 Docker 部署 MariaDB 的详细标准作业程序 (SOP)。

Raspberry Pi 5 使用的是 ARM64 架构，因此在选择镜像时需要注意架构兼容性。

---

### SOP: 在 Raspberry Pi 5 上部署 MariaDB (Docker)

#### 1. 前置条件
在开始之前，请确保你的 Raspberry Pi 5 已安装好操作系统，并且具备以下条件：
*   **操作系统**：Raspberry Pi OS (基于 Debian)。
*   **网络**：能够连接互联网（用于拉取 Docker 镜像）。
*   **权限**：拥有 sudo 权限。

#### 2. 安装 Docker 和 Docker Compose
如果你还没有安装 Docker，请执行以下命令：

```bash
# 更新软件包列表
sudo apt update && sudo apt upgrade -y

# 安装 Docker (使用官方脚本，推荐)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 安装 Docker Compose 插件 (推荐使用 v2 版本)
sudo apt install docker-compose-plugin -y

# 将当前用户添加到 docker 组 (可选，避免每次使用 sudo)
sudo usermod -aG docker $USER
# 注意：执行完此命令后需要重新登录或运行 newgrp docker 生效
```

#### 3. 创建项目目录结构
为了方便管理，建议创建一个专门的文件夹来存放配置和数据。

```bash
# 创建主目录
mkdir -p ~/mariadb-docker
cd ~/mariadb-docker

# 创建 docker-compose.yml 文件
nano ~/mariadb-docker/docker-compose.yml

# 创建子目录用于数据持久化和配置
mkdir data
mkdir conf
```

#### 4. 编写 `docker-compose.yml` 文件
在 `~/mariadb-docker` 目录下创建一个名为 `docker-compose.yml` 的文件。

**重要提示**：

*   **架构**：必须指定 `platform: linux/arm64`，因为 Pi 5 是 ARM 架构。
*   **数据持久化**：将 `./data` 挂载到容器内，防止重启后数据丢失。
*   **配置文件**：将 `./conf` 挂载出来，方便修改配置而不需要进入容器。

```yaml
services:
  mariadb:
    image: mariadb:latest
    container_name: pi5-mariadb
    platform: linux/arm64
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: PWD
    ports:
      - "3306:3306"
    volumes:
      - /home/pi/nvme/database:/var/lib/mysql
      - ./conf:/etc/mysql/conf.d
    healthcheck:
      test: ["CMD-SHELL", "mariadb -h 127.0.0.1 -u root -p$$MYSQL_ROOT_PASSWORD -e 'SELECT 1;' --silent"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

#### 5. 启动 MariaDB 容器
使用 Docker Compose 启动服务。

```bash
# 在 ~/mariadb-docker 目录下执行
docker compose up -d
```

*   `-d` 参数表示在后台运行。

#### 6. 验证部署状态
检查容器是否正在运行，以及日志是否有错误。

```bash
# 查看容器状态
docker compose ps

# 查看日志 (如果启动失败，查看日志是第一步)
docker compose logs -f mariadb
```

**预期结果**：
*   `docker compose ps` 显示 `Status: Up X seconds`。
*   日志末尾显示 `ready for connections`。

#### 7. 测试连接
你可以使用命令行工具或数据库管理工具（如 DBeaver, phpMyAdmin, Navicat）连接到数据库。

**连接参数**：
*   **Host**: `localhost` (或 `127.0.0.1`)
*   **Port**: `3306`
*   **User**: `root` (或你在 compose 文件中设置的 `MYSQL_USER`)
*   **Password**: `your_strong_password_here`

#### 8. 常用维护操作

**停止服务**：
```bash
cd ~/mariadb-docker
docker compose down
```

**重启服务**：
```bash
docker compose restart
```

**更新 MariaDB 版本**：
```bash
# 1. 拉取最新镜像
docker compose pull

# 2. 重启容器
docker compose up -d
```

**备份数据库**：
由于数据存储在 `./data` 目录中，最简单的备份方式是打包该目录。
```bash
# 备份到 tar.gz 文件
tar -czvf mariadb-backup-$(date +%Y%m%d).tar.gz ./data
```

---

### ⚠️ 重要建议

1.  **SD 卡寿命问题**：
    Raspberry Pi 5 通常使用 SD 卡作为存储。频繁的写入操作（如 MariaDB 的日志和事务日志）会缩短 SD 卡寿命。
    *   **强烈建议**：将 `./data` 目录移动到 USB 3.0 外接硬盘或 SSD 上，以获得更好的性能和寿命。

2.  **性能调优**：
    如果你的 Pi 5 运行多个服务，MariaDB 可能会占用较多内存。默认配置通常足够，但如果遇到卡顿，可以在 `./conf` 目录下创建自定义配置文件（如 `my.cnf`）来调整 `innodb_buffer_pool_size`。

3.  **安全加固**：
    生产环境中，不要将数据库端口（3306）直接暴露在公网。建议使用反向代理或仅在局域网内访问。



### **安装dpanel Lite 版**

```
docker run -d --name dpanel --restart=always \
 -p 9000:8080 -e APP_NAME=dpanel \
 -v /var/run/docker.sock:/var/run/docker.sock \
 -v /home/dpanel:/dpanel dpanel/dpanel:lite
```

### **💡 部署后提示：**

1. 运行上述命令后，Lite 版会自动拉取并启动。
2. 打开浏览器访问 `http://<您的树莓派IP>:9000`。
3. 因为是全新安装，系统会要求您**重新创建管理员账号和密码**，设置完成后即可愉快地管理您的 Docker 容器了！