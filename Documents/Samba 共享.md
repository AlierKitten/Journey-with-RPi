好的！你想将 **树莓派5 的 `/home/pi/` 目录** 通过 Samba 共享出来，在 Windows 上通过 `\\10.168.10.236\pi` 访问，并使用 **用户名 `pi` 和其密码（Samba 密码）进行身份验证**。下面为你提供完整、安全、可直接操作的配置步骤。

------

## ✅ 目标说明

- 共享路径：`/home/pi`
- 共享名称：`pi`
- 访问方式：Windows 输入 `\\10.168.10.236\pi`
- 身份验证：必须使用用户名 `pi` + Samba 密码（不启用 guest）
- 权限：`pi` 用户拥有读写权限

------

## 🔧 步骤详解

### 1. 确保系统最新

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. 安装 Samba

```bash
sudo apt install samba samba-common-bin -y
```

### 3. 为 `pi` 用户设置 Samba 密码

> ⚠️ 注意：Samba 使用独立的密码数据库，即使你有系统密码，也需单独设置！

```bash
sudo smbpasswd -a pi
```

系统会提示你输入并确认一个 **Samba 密码**（建议与系统密码一致以便记忆，但非强制）。

然后启用该用户（通常默认已启用）：

```bash
sudo smbpasswd -e pi
```

### 4. 备份并编辑 Samba 配置文件

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak  # 备份
sudo nano /etc/samba/smb.conf
```

### 5. 修改配置文件

#### ✅ 在 `[global]` 部分确保包含以下内容（可替换或补充）：

```ini
[global]
   workgroup = WORKGROUP
   server string = Samba Server %v
   netbios name = raspberrypi
   security = user
   map to guest = never          # 禁用访客访问
   guest account = nobody
   server min protocol = SMB2    # 提高安全性，禁用老旧 SMB1
   obey pam restrictions = yes
   unix password sync = yes
   pam password change = yes
```

> 关键点：`map to guest = never` 表示 **必须登录**，不允许匿名访问。

#### ✅ 在文件末尾添加共享定义：

```ini
[pi]
   comment = Home directory of user pi
   path = /home/pi
   browseable = yes
   writable = yes
   read only = no
   valid users = pi
   create mask = 0644
   directory mask = 0755
```

> 说明：
>
> - `valid users = pi`：只允许 `pi` 用户访问
> - `browseable = yes`：在网络邻居中可见
> - 权限掩码较安全（避免 777）

⚠️ **重要**：确保每行配置 **不要在值后面加注释或空格**，例如不能写成：

```ini
guest ok = no  # 不允许访客 ← ❌ 错误！Samba 会报“value is not boolean”
```

### 6. 保存并退出

- `Ctrl+O` → 回车 → `Ctrl+X`

### 7. 检查配置语法

```bash
testparm
```

如果没有报错，会显示你的配置摘要。如有错误，请根据提示修正。

### 8. 重启 Samba 服务

```bash
sudo systemctl restart smbd nmbd
sudo systemctl enable smbd nmbd  # 开机自启
```

### 9. 确认树莓派 IP 是 `10.168.10.236`

```bash
hostname -I
```

如果 IP 不是 `10.168.10.236`，请检查网络设置（可能是 DHCP 分配问题，建议在路由器中为树莓派设置静态 IP 或在树莓派中配置静态 IP）。

------

## 💻 Windows 访问方式

1. 打开 **文件资源管理器**

2. 地址栏输入：

   ```
   \\10.168.10.236\pi
   ```

3. 弹出登录窗口时：

   - 用户名：`pi`
   - 密码：你刚才用 `smbpasswd -a pi` 设置的密码
   - 勾选“记住凭据”（可选）

✅ 成功后即可像访问本地文件夹一样操作 `/home/pi`。

------

## 🔒 安全建议

- 不要使用 `guest ok = yes` 或 `777` 权限，除非在完全可信的内网。
- 定期更新系统：`sudo apt upgrade`
- 如果仅内网使用，无需额外防火墙；若暴露到公网，请务必加强安全（如禁用 SMB、改用 SSHFS/SFTP）。

------

## 🛠 故障排查

### Q：提示“找不到网络路径”？

- 检查 IP 是否正确
- 检查树莓派是否能 ping 通：`ping 10.168.10.236`
- 检查 Samba 是否运行：`sudo systemctl status smbd`

### Q：提示“没有权限”？

- 确认 `smbpasswd -a pi` 已设置
- 确认配置中 `valid users = pi`
- 确认 Windows 输入的用户名是 `pi`（不是 `raspberrypi\pi`，除非你加了域名）

> 如果仍失败，可在 Windows 凭据管理器中删除旧的凭据，重新连接。

------

Samba 的 `map to guest` 参数用于控制 **当用户认证失败时** 的行为。它的常见取值有：

| 值             | 行为                                                         |
| -------------- | ------------------------------------------------------------ |
| `never`        | 从不映射为 guest。认证失败就直接拒绝连接。✅ 这是真正“禁用访客”的方式。 |
| `bad user`     | 如果提供的用户名 在系统中根本不存在，则自动转为 guest 用户登录。⚠️ 存在安全风险。 |
| `bad password` | 如果用户名存在但密码错误，也转为 guest（更危险，一般不用）。 |