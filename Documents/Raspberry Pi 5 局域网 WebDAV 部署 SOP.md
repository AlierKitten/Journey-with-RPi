# Raspberry Pi 5 局域网 WebDAV 部署 SOP

## 1. 环境说明

本文适用于 Raspberry Pi 5 + Raspberry Pi OS，使用 Apache 提供 WebDAV 服务，Nginx 作为局域网入口。

当前实际环境：

- Raspberry Pi：Pi 5
- WebDAV 数据目录：`/home/pi/nvme/webdav`
- NVMe 挂载点：`/home/pi/nvme`
- Nginx HTTP 端口：`878`
- Apache WebDAV 后端端口：`4448`
- Apache WebDAV 仅监听：`127.0.0.1:4448`
- 局域网访问地址：

```text
http://10.168.10.236:878/webdav/
```

架构：

```text
局域网客户端
      |
      | HTTP :878
      v
    Nginx
      |
      | 127.0.0.1:4448
      v
    Apache
      |
      | WebDAV
      v
/home/pi/nvme/webdav
      |
      v
     NVMe
```

设计目标：

1. WebDAV 只允许通过局域网访问。
2. Apache 后端不直接暴露到局域网。
3. 不使用 HTTPS。
4. 使用 Basic Authentication 保护 WebDAV。
5. 不修改 NVMe/FUSE 挂载目录的所有者。
6. 不影响现有 AriaNg/Nginx 服务。

---

# 2. 检查 NVMe 挂载

确认 NVMe 已经挂载：

```bash
findmnt /home/pi/nvme
```

也可以检查文件系统：

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /home/pi/nvme
```

本环境实际为 FUSE 挂载。

检查 WebDAV 目录：

```bash
ls -ld /home/pi/nvme
ls -ld /home/pi/nvme/webdav
```

创建目录：

```bash
mkdir -p /home/pi/nvme/webdav
```

测试文件：

```bash
echo "WebDAV test" > /home/pi/nvme/webdav/test.txt
```

---

# 3. FUSE 挂载的重要说明

本环境的 `/home/pi/nvme` 是 FUSE 挂载。

因此不要按照普通 Linux 磁盘的方式执行：

```bash
sudo chown -R webdav:webdav /home/pi/nvme/webdav
```

在当前 FUSE 挂载配置下，这类 `chown` 可能直接失败：

```text
Operation not permitted
```

这是正常现象，不应该继续通过 `chown` 解决。

本环境使用：

```text
www-data
```

作为 Apache 运行用户，并通过 ACL 解决 Apache 对 `/home/pi` 的目录穿越权限。

---

# 4. 解决 /home/pi 的 Apache 访问权限

首先检查：

```bash
ls -ld /home/pi
```

如果类似：

```text
drwx------ pi pi /home/pi
```

那么 Apache 的 `www-data` 即使拥有 `/home/pi/nvme/webdav` 的访问权限，也无法穿过 `/home/pi`。

检查 `setfacl`：

```bash
which setfacl
```

如果不存在：

```bash
sudo apt update
sudo apt install acl
```

给 `www-data` 仅增加 `/home/pi` 的执行/穿越权限：

```bash
sudo setfacl -m u:www-data:--x /home/pi
```

检查：

```bash
getfacl /home/pi
```

应该能看到：

```text
user:www-data:--x
```

注意：

这里的 `--x` 只允许 `www-data` 穿过 `/home/pi`，不会赋予其读取 `/home/pi` 目录列表的权限。

不要为了 WebDAV 把 `/home/pi` 改成：

```bash
chmod 755 /home/pi
```

ACL 是更合适的解决方式。

---

# 5. 安装 Apache

如果尚未安装：

```bash
sudo apt update
sudo apt install apache2
```

确认 Apache：

```bash
apache2 -v
```

启用 WebDAV 模块：

```bash
sudo a2enmod dav
sudo a2enmod dav_fs
```

启用 Basic Authentication：

```bash
sudo a2enmod auth_basic
```

如果修改了模块：

```bash
sudo systemctl restart apache2
```

---

# 6. 配置 Apache WebDAV 监听端口

编辑：

```bash
sudo nano /etc/apache2/ports.conf
```

加入：

```apache
Listen 127.0.0.1:4448
```

这里必须监听 `127.0.0.1`，而不是：

```apache
Listen 0.0.0.0:4448
```

原因是 Apache 只是 Nginx 的内部 WebDAV 后端，不应该直接暴露到局域网。

检查：

```bash
grep -n "4448" /etc/apache2/ports.conf
```

---

# 7. 创建 WebDAV 用户

创建认证目录：

```bash
sudo mkdir -p /etc/apache2/webdav
```

创建用户：

```bash
sudo htpasswd -c /etc/apache2/webdav/.htpasswd webdav
```

系统会要求输入密码。

注意：

`-c` 只在第一次创建密码文件时使用。

以后增加用户：

```bash
sudo htpasswd /etc/apache2/webdav/.htpasswd username
```

检查：

```bash
sudo ls -l /etc/apache2/webdav/.htpasswd
```

---

# 8. Apache WebDAV 虚拟主机

创建：

```bash
sudo nano /etc/apache2/sites-available/webdav.conf
```

内容：

```apache
<VirtualHost 127.0.0.1:4448>
    ServerName localhost

    Alias /webdav /home/pi/nvme/webdav

    <Directory /home/pi/nvme/webdav>
        Options Indexes
        AllowOverride None

        Dav On

        AuthType Basic
        AuthName "WebDAV"
        AuthUserFile /etc/apache2/webdav/.htpasswd
        Require valid-user
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/webdav_error.log
    CustomLog ${APACHE_LOG_DIR}/webdav_access.log combined
</VirtualHost>
```

启用：

```bash
sudo a2ensite webdav.conf
```

检查：

```bash
sudo apache2ctl configtest
```

必须得到：

```text
Syntax OK
```

检查虚拟主机：

```bash
sudo apache2ctl -S
```

应该能看到：

```text
127.0.0.1:4448
```

重启：

```bash
sudo systemctl restart apache2
```

检查监听：

```bash
sudo ss -lntp | grep 4448
```

期望：

```text
127.0.0.1:4448
```

---

# 9. 首先直接测试 Apache

这一阶段不要经过 Nginx。

测试目录：

```bash
curl -i -u webdav http://127.0.0.1:4448/webdav/
```

正常应该：

```text
HTTP/1.1 200 OK
```

并看到目录列表。

如果这里出现：

```text
403 Forbidden
```

优先检查：

```bash
ls -ld /home/pi
getfacl /home/pi
```

确认存在：

```text
user:www-data:--x
```

然后再次测试。

---

# 10. 正确测试 WebDAV PROPFIND

不要直接执行：

```bash
curl -X PROPFIND http://127.0.0.1:4448/webdav/
```

因为 Apache 对默认的：

```text
Depth: infinity
```

会拒绝请求。

正确测试：

```bash
curl -i -u webdav -X PROPFIND \
  -H "Depth: 1" \
  http://127.0.0.1:4448/webdav/
```

正常结果：

```text
HTTP/1.1 207 Multi-Status
```

`207 Multi-Status` 才是 WebDAV PROPFIND 的正常成功状态。

如果看到：

```text
403 Forbidden

PROPFIND requests with a Depth of "infinity" are not allowed
```

这不代表 WebDAV 权限配置错误，只代表请求没有指定合适的 `Depth`。

---

# 11. 配置 Nginx 反向代理

现有 Nginx 配置文件：

```text
/etc/nginx/sites-enabled/default
```

在现有 `server` 中加入：

```nginx
location /webdav/ {
    proxy_pass http://127.0.0.1:4448/webdav/;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_request_buffering off;
    proxy_buffering off;

    client_max_body_size 0;

    proxy_connect_timeout 60s;
    proxy_send_timeout 3600s;
    proxy_read_timeout 3600s;
}
```

特别注意：

必须是：

```nginx
proxy_pass http://127.0.0.1:4448/webdav/;
```

不是：

```nginx
proxy_pass http://127.0.0.1:8080/webdav/;
```

当前 Apache WebDAV 实际使用的是 `4448`。

---

# 12. Nginx 配置检查

执行：

```bash
sudo nginx -t
```

正常：

```text
syntax is ok
test is successful
```

然后：

```bash
sudo systemctl reload nginx
```

检查：

```bash
sudo systemctl status nginx --no-pager
```

---

# 13. 测试 Nginx + Apache + WebDAV

先测试浏览器：

```text
http://10.168.10.236:878/webdav/
```

应该可以看到：

```text
Index of /webdav
```

并看到例如：

```text
test.txt
```

如果浏览器能看到目录，说明：

```text
客户端
  ↓
Nginx :878
  ↓
Apache :4448
  ↓
/home/pi/nvme/webdav
```

基本链路已经正常。

---

# 14. 从 Nginx 入口测试 PROPFIND

执行：

```bash
curl -i -u webdav -X PROPFIND \
  -H "Depth: 1" \
  http://10.168.10.236:878/webdav/
```

正常：

```text
HTTP/1.1 207 Multi-Status
```

---

# 15. 测试文件上传

创建测试文件：

```bash
echo "WebDAV test" > /tmp/webdav-test.txt
```

通过 Nginx 上传：

```bash
curl -i -u webdav \
  -T /tmp/webdav-test.txt \
  http://10.168.10.236:878/webdav/webdav-test.txt
```

正常通常为：

```text
HTTP/1.1 201 Created
```

检查实际目录：

```bash
ls -l /home/pi/nvme/webdav/
```

应该看到：

```text
webdav-test.txt
```

---

# 16. 测试下载

```bash
curl -i -u webdav \
  http://10.168.10.236:878/webdav/webdav-test.txt
```

应该返回：

```text
WebDAV test
```

---

# 17. 测试删除

```bash
curl -i -u webdav \
  -X DELETE \
  http://10.168.10.236:878/webdav/webdav-test.txt
```

正常通常为：

```text
HTTP/1.1 204 No Content
```

确认：

```bash
ls -l /home/pi/nvme/webdav/
```

测试文件应该消失。

---

# 18. 最终访问地址

局域网客户端使用：

```text
http://10.168.10.236:878/webdav/
```

用户名：

```text
webdav
```

密码：

```text
创建 htpasswd 时设置的密码
```

---

# 19. 安全边界

当前结构：

```text
LAN
 |
 | :878
 v
Nginx
 |
 | 127.0.0.1:4448
 v
Apache WebDAV
 |
 v
NVMe
```

Apache 不监听：

```text
0.0.0.0:4448
```

因此局域网设备不能直接访问：

```text
http://10.168.10.236:4448/
```

WebDAV 的唯一入口是：

```text
http://10.168.10.236:878/webdav/
```

同时，Nginx 现有的 AriaNg 配置继续保留。

注意：

Nginx 原本的：

```nginx
location / {
    auth_basic "Aria2 Web Access";
    auth_basic_user_file /etc/nginx/.aria2_passwd;
}
```

与：

```nginx
location /webdav/
```

是两个独立的 location。

WebDAV 的认证由 Apache 的：

```apache
AuthType Basic
AuthUserFile /etc/apache2/webdav/.htpasswd
Require valid-user
```

负责。

因此 WebDAV 使用的是 Apache 的 `webdav` 用户，而不是 Nginx 的 `.aria2_passwd` 用户。

---

# 20. 常用检查命令

## 检查 Apache

```bash
sudo systemctl status apache2 --no-pager
```

## 检查 Apache 监听

```bash
sudo ss -lntp | grep 4448
```

## 检查 Apache 虚拟主机

```bash
sudo apache2ctl -S
```

## Apache 配置检查

```bash
sudo apache2ctl configtest
```

## Apache WebDAV 错误日志

```bash
sudo tail -n 50 /var/log/apache2/webdav_error.log
```

## Apache WebDAV 访问日志

```bash
sudo tail -n 50 /var/log/apache2/webdav_access.log
```

## 检查 Nginx

```bash
sudo nginx -t
```

```bash
sudo systemctl status nginx --no-pager
```

## 检查 Nginx WebDAV 请求

```bash
sudo tail -n 50 /var/log/nginx/access.log
```

## 检查 WebDAV 目录

```bash
ls -la /home/pi/nvme/webdav
```

## 检查 /home/pi ACL

```bash
getfacl /home/pi
```

---

# 21. 故障排查顺序

遇到 WebDAV 无法访问时，严格按照下面顺序排查。

### 第一层：检查目录

```bash
ls -ld /home/pi
ls -ld /home/pi/nvme
ls -ld /home/pi/nvme/webdav
```

### 第二层：检查 ACL

```bash
getfacl /home/pi
```

必须确认：

```text
user:www-data:--x
```

### 第三层：直接访问 Apache

```bash
curl -i -u webdav \
  http://127.0.0.1:4448/webdav/
```

必须先确保这里正常。

### 第四层：测试 WebDAV

```bash
curl -i -u webdav -X PROPFIND \
  -H "Depth: 1" \
  http://127.0.0.1:4448/webdav/
```

期望：

```text
207 Multi-Status
```

### 第五层：检查 Nginx

```bash
sudo nginx -t
```

确认：

```nginx
proxy_pass http://127.0.0.1:4448/webdav/;
```

### 第六层：通过 Nginx 测试

```bash
curl -i -u webdav -X PROPFIND \
  -H "Depth: 1" \
  http://10.168.10.236:878/webdav/
```

---

# 22. 当前已验证结果

本次实际部署过程中已经验证：

1. Apache WebDAV 虚拟主机成功加载。
2. Apache 实际监听 `127.0.0.1:4448`。
3. `/home/pi` 原本的 `700` 权限导致 `www-data` 无法穿越目录。
4. 使用：

```bash
sudo setfacl -m u:www-data:--x /home/pi
```

解决了 Apache 对 WebDAV 目录的访问问题。
5. Apache 直接访问：

```text
http://127.0.0.1:4448/webdav/
```

已经返回：

```text
HTTP/1.1 200 OK
```

6. 浏览器访问：

```text
http://10.168.10.236:878/webdav/
```

已经可以看到 WebDAV 目录。
7. 直接使用不带 `Depth` 的 PROPFIND 得到：

```text
403 Forbidden
```

原因是 Apache 拒绝：

```text
Depth: infinity
```

正确测试方式是指定：

```text
Depth: 1
```

预期：

```text
207 Multi-Status
```

---

# 23. 最终配置文件

## `/etc/apache2/ports.conf`

```apache
Listen 127.0.0.1:4448
```

## `/etc/apache2/sites-enabled/webdav.conf`

```apache
<VirtualHost 127.0.0.1:4448>
    ServerName localhost

    Alias /webdav /home/pi/nvme/webdav

    <Directory /home/pi/nvme/webdav>
        Options Indexes
        AllowOverride None
        Require all granted
        Dav On

        AuthType Basic
        AuthName "WebDAV"
        AuthUserFile /etc/apache2/webdav/.htpasswd
        Require valid-user
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/webdav_error.log
    CustomLog ${APACHE_LOG_DIR}/webdav_access.log combined
</VirtualHost>
```

## `/etc/nginx/sites-enabled/default`

在原有 `server` 中增加：

```nginx
location /webdav/ {
    proxy_pass http://127.0.0.1:4448/webdav/;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_request_buffering off;
    proxy_buffering off;

    client_max_body_size 0;

    proxy_connect_timeout 60s;
    proxy_send_timeout 3600s;
    proxy_read_timeout 3600s;
}
```

---

# 24. 维护建议

不要随意修改：

```text
/home/pi
```

的权限。

不要对当前 FUSE 挂载中的：

```text
/home/pi/nvme/webdav
```

执行依赖 `chown` 的权限方案。

如果以后重新挂载 NVMe，重点确认：

```bash
findmnt /home/pi/nvme
```

以及：

```bash
ls -ld /home/pi/nvme/webdav
```

如果 Apache 再次出现 403，首先检查：

```bash
getfacl /home/pi
```

确保：

```text
user:www-data:--x
```

仍然存在。

如果更换 Apache、Nginx、NVMe 挂载方式或用户，需要重新确认 WebDAV 运行用户以及目录穿越权限。
