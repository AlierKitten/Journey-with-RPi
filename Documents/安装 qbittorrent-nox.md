要配置 `qbittorrent-nox`（qBittorrent 无头/服务器版本）启动 **Web UI**，其实它默认就启用了 Web UI，你只需要知道如何访问、修改端口、设置账号密码等。下面是在树莓派5上使用 `qbittorrent-nox` 配置 Web UI 的完整指南：

------

## ✅ 一、确认已安装 `qbittorrent-nox`

```bash
sudo apt update
sudo apt install qbittorrent-nox
```

> 安装后不要直接运行 `qbittorrent`（那是 GUI 版），而是用 `qbittorrent-nox`。

------

## ✅ 二、首次启动（生成配置文件）

第一次运行会自动生成配置目录和默认配置：

```bash
qbittorrent-nox
```

看到类似输出：

```
Web UI: Now listening on IP: *, port: 8080
...
```

按 `Ctrl+C` 停止它。此时配置文件已生成在：

```
~/.local/share/data/qBittorrent/qBittorrent.conf
```

------

## ✅ 三、修改 Web UI 设置（可选但推荐）

编辑配置文件：

```bash
nano ~/.local/share/data/qBittorrent/qBittorrent.conf
```

找到 `[Preferences]` 段落，添加或修改以下关键参数：

```ini
[Preferences]
WebUI\Port=8080
WebUI\Username=admin
WebUI\Password=@ByteArray(密文)  ; ← 先别改这个！见下方说明
WebUI\LocalHostAuth=false        ; 允许非本机访问（重要！）
WebUI\UseUPnP=true
WebUI\AlternativeUIEnabled=false
WebUI\HTTPS\Enabled=false
```

### 🔐 关于密码：

- **不要手动写明文密码**！  
- 正确做法：先用默认账号登录 Web UI，然后在网页里改密码，程序会自动加密保存。
- 默认账号密码是：
  - 用户名：`admin`
  - 密码：`adminadmin`

> ⚠️ 如果你强行手写明文密码（如 `WebUI\Password=admin123`），**不会生效**，因为 qBittorrent 要求密码是加密后的 `@ByteArray(...)` 格式。

------

## ✅ 四、正确设置密码的方法

1. 启动 `qbittorrent-nox`：

   ```bash
   qbittorrent-nox
   ```

2. 在另一台电脑或手机浏览器访问：

   ```
   http://<树莓派IP>:8080
   ```

   例如：`http://192.168.1.100:8080`

3. 用默认账号登录：

   - 用户名：`admin`
   - 密码：`adminadmin`

4. 进入 Web UI 后，点击右上角 **工具 → 选项 → Web UI**

   - 修改用户名（可选）
   - **设置新密码**
   - 勾选 **“允许远程连接”**（即 `Bypass authentication for clients on localhost` 可不勾，但确保 `Use alternative Web UI` 不开除非你需要）

5. 点击 **保存**

> 此时配置文件中的 `WebUI\Password` 会自动更新为加密格式，无需手动编辑。

------

## ✅ 五、配置 systemd 开机自启（推荐）

创建服务文件（以用户 `pi` 运行）：

```bash
sudo nano /etc/systemd/system/qbittorrent.service
```

内容如下：

```ini
[Unit]
Description=qBittorrent NoX Service
After=network.target

[Service]
Type=exec
User=pi
Group=pi
ExecStart=/usr/bin/qbittorrent-nox --profile=/home/pi/.local/share/data/qBittorrent
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable qbittorrent.service
sudo systemctl start qbittorrent.service
```

查看状态：

```bash
systemctl status qbittorrent.service
```

------

## ✅ 六、防火墙（如有启用）

如果启用了 `ufw`，放行 8080 端口：

```bash
sudo ufw allow 8080/tcp
```

------

## ✅ 七、访问 Web UI

浏览器打开：

```
http://<你的树莓派IP>:8080
```

例如：

```
http://192.168.1.50:8080
```

输入你设置的用户名和密码即可管理下载任务！

------

## 💡 小贴士

- Web UI 功能完整，支持添加种子、限速、分类、RSS 等，和桌面版几乎一致。
- 你可以通过手机浏览器随时管理下载，无需开图形界面，省资源。
- 如果想换端口（比如 8081），在 Web UI 设置里改，或直接改 `qBittorrent.conf` 中的 `WebUI\Port=8081`。

------

需要我帮你写一个完整的配置模板，或者设置 HTTPS、反向代理（如 Nginx）？欢迎继续问！