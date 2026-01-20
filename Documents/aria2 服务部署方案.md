下面按**树莓派 5（Raspberry Pi OS / Debian 系）\**给你一套\**标准、稳定的 aria2 服务部署方案**，适合长期后台运行、远程 RPC 使用。

------

## 一、安装 aria2

```bash
sudo apt update
sudo apt install -y aria2
```

验证是否安装成功：

```bash
aria2c -v
```

------

## 二、创建 aria2 配置目录

```bash
mkdir -p ~/.config/aria2
cd ~/.config/aria2
```

创建以下 3 个文件：

```bash
touch aria2.conf aria2.session aria2.log
```

------

## 三、编写 aria2.conf（核心配置）

编辑配置文件：

```bash
nano ~/.config/aria2/aria2.conf
```

**推荐配置（可直接复制）：**

```ini
# 基本
daemon=true
enable-rpc=true
rpc-listen-all=true
rpc-allow-origin-all=true
rpc-listen-port=6800
#rpc-secret=

# 下载设置
dir=/home/pi/nvme/Downloads/
file-allocation=trunc
continue=true

# 会话
input-file=/home/pi/.config/aria2/aria2.session
save-session=/home/pi/.config/aria2/aria2.session
save-session-interval=60

# 日志
log=/home/pi/.config/aria2/aria2.log
log-level=notice

# 连接数优化
split=16
min-split-size=5M
max-connection-per-server=16
```

说明：

- `rpc-secret`：RPC 密码（一定要改）
- `dir`：下载目录（确保存在）
- `daemon=true`：后台运行

创建下载目录：

```bash
mkdir -p ~/downloads
```

------

## 四、手动启动测试

```bash
aria2c --conf-path=/home/pi/.config/aria2/aria2.conf
```

查看端口是否监听：

```bash
ss -tunlp | grep 6800
```

如果看到 aria2 在 6800 端口监听，说明成功。

------

## 五、配置 systemd 服务（推荐，开机自启）

创建服务文件：

```bash
sudo nano /etc/systemd/system/aria2.service
```

内容如下（**注意 User=pi**）：

```ini
[Unit]
Description=Aria2 Service
After=network.target

[Service]
Type=forking
User=pi
ExecStart=/usr/bin/aria2c --conf-path=/home/pi/.config/aria2/aria2.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

加载并启动服务：

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable aria2
sudo systemctl start aria2
```

查看状态：

```bash
systemctl status aria2
```

------

## 六、常用管理命令

```bash
# 启动
sudo systemctl start aria2

# 停止
sudo systemctl stop aria2

# 重启
sudo systemctl restart aria2

# 查看日志
tail -f ~/.config/aria2/aria2.log
```

------

## 七、远程控制

### 1. Web UI（AriaNg）

```bash
sudo apt install -y nginx
```

下载 AriaNg：

```bash
cd /var/www/html
sudo wget https://github.com/mayswind/AriaNg/releases/latest/download/AriaNg-AllInOne.zip
sudo unzip AriaNg-AllInOne.zip
sudo chown -R www-data:www-data /var/www/html
```

浏览器访问：

```
http://树莓派IP/
```

RPC 设置：

- 地址：`http://树莓派IP:6800/jsonrpc`
- 密钥：`rpc-secret`

------

## 八、Web UI鉴权

### 1.修改 nginx 监听端口（878）

#### 1️⃣ 找到 nginx 的 server 配置

通常在：

```
sudo nano /etc/nginx/sites-available/default
```

#### 2️⃣ 修改为监听 878 端口

把原来的：

```
listen 80 default_server;
listen [::]:80 default_server;
```

改成：

```
listen 878;
listen [::]:878;
```

完整示例（基础版）：

```
server {
    listen 878;
    listen [::]:878;

    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

------

#### 3️⃣ 重启 nginx

```
sudo nginx -t
sudo systemctl restart nginx
```

此时访问：

```
http://10.168.10.236:878/
```

应该已经能看到 AriaNg 页面。

------

### 2.添加账号密码验证（HTTP Basic Auth）

#### 1️⃣ 安装认证工具

```
sudo apt install -y apache2-utils
```

------

#### 2️⃣ 创建密码文件

```
sudo htpasswd -c /etc/nginx/.aria2_passwd admin
```

会提示你输入密码。

说明：

- `admin` 是用户名（可换）
- `-c` 只在第一次用，**以后新增用户不要加 -c**

例如新增用户：

```
sudo htpasswd /etc/nginx/.aria2_passwd user1
```

------

#### 3️⃣ 修改 nginx 配置，启用认证

编辑 nginx 配置：

```
sudo nano /etc/nginx/sites-available/default
```

在 `location /` 中加入认证配置：

```
server {
    listen 878;
    listen [::]:878;

    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        auth_basic "Aria2 Web Access";
        auth_basic_user_file /etc/nginx/.aria2_passwd;

        try_files $uri $uri/ =404;
    }
}
```

------

#### 4️⃣ 重载 nginx

```
sudo nginx -t
sudo systemctl reload nginx
```

现在访问：

```
http://10.168.10.236:878/
```

浏览器会弹出 **用户名 / 密码** 登录框。

------

### 3.重要安全提醒（必须看）

#### 1️⃣ 这是**前端认证，不是 aria2 认证**

- nginx 的账号密码 👉 **保护 WebUI**
- aria2 的 `rpc-secret` 👉 **保护下载控制权**

👉 **两者都要有**

------

#### 2️⃣ 确认 aria2 不暴露到公网

检查 `aria2.conf`：

```
rpc-listen-all=true   # 只在局域网使用可以
rpc-secret=强密码
```

如果你想更严格（推荐）：

```
rpc-listen-all=false
```