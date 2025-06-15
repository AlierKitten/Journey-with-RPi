- 自动挂载 `/dev/nvme0n1p1`
- 文件系统类型是 **NTFS**
- 希望挂载到 `/home/pi/`（可以用 `/home/pi/nvme` 子目录，避免覆盖）

我们可以通过编辑 `/etc/fstab` 实现 **开机自动挂载、免密码访问**，兼顾**权限和稳定性**。

------

## ✅ 设置步骤

### 🔹 第 1 步：创建挂载目录

```bash
mkdir -p /home/pi/nvme
```

确保你有权限访问它：

```bash
sudo chown pi:pi /home/pi/nvme
```

------

### 🔹 第 2 步：编辑挂载配置

编辑 `/etc/fstab`：

```bash
sudo nano /etc/fstab
```

在文件末尾添加这一行：

```fstab
UUID=A236E77336E746BF /home/pi/nvme ntfs defaults,uid=1000,gid=1000,noatime,nofail 0 0
```

含义说明：

| 参数                         | 说明                                  |
| ---------------------------- | ------------------------------------- |
| `UUID=...`                   | 使用硬盘的 UUID，系统识别稳定         |
| `/home/pi/nvme`              | 挂载路径                              |
| `ntfs`                       | 文件系统类型                          |
| `defaults,uid=1000,gid=1000` | 挂载后属于 pi 用户（默认 ID 是 1000） |
| `noatime`                    | 提高性能（不记录访问时间）            |
| `nofail`                     | 如果挂载失败不会阻止系统启动          |
| `0 0`                        | 不做 fsck 检查                        |

------

### 🔹 第 3 步：测试是否成功

不需要重启，可以手动测试挂载：

```bash
sudo mount -a
```

然后查看是否成功：

```bash
ls /home/pi/nvme
```

如果能正常访问，挂载就成功了，而且是**以 pi 用户身份、无密码自动挂载**。

------

## ✅ 总结

你现在的 `/etc/fstab` 应该添加如下这一行即可：

```
UUID=A236E77336E746BF /home/pi/nvme ntfs defaults,uid=1000,gid=1000,noatime,nofail 0 0
```

之后，系统每次开机都会自动挂载这块 NTFS 格式的 M.2 硬盘，并且你用 `pi` 用户访问不会再有权限问题或提示密码。

------

# Q1:

我发现挂载后ssh地址显示/home/pi/nvme/，而我在图形端的文件管理器中看到是/home/RasPi/，其中RasPi是我硬盘的名字\

# A1:

你看到的行为其实是 Raspberry Pi OS（特别是图形界面）**自动挂载行为和手动挂载冲突**导致的。

------

## ✅ 现象解析

| 情况                                | 原因                                             |
| ----------------------------------- | ------------------------------------------------ |
| 💻 图形界面看到 `/home/RasPi/`       | 是系统 **自动挂载** 的路径，使用硬盘卷标 `RasPi` |
| 🔧 手动挂载到 `/home/pi/nvme`        | 是你通过 `/etc/fstab` 指定的挂载点               |
| ⚠️ SSH 中看到 `/home/pi/nvme/`       | 是你访问你手动指定的挂载路径                     |
| 🎭 实际只有一个设备 `/dev/nvme0n1p1` | 但如果自动挂载没关闭，它会抢先挂载，阻止手动生效 |

------

## ✅ 你的目标是？

你大概希望实现以下效果：

> 硬盘 **挂载到 `/home/pi/nvme`**，且所有方式（SSH、文件管理器）访问时路径一致，不出现 `/home/RasPi`。

------

## ✅ 推荐解决方案

你只需 **禁用系统图形界面自动挂载功能**，完全由你设置的 `/etc/fstab` 控制即可。

### 🔧 方法 1：禁用 `udisks2` 自动挂载（推荐）

创建一个规则文件：

```bash
sudo nano /etc/udev/rules.d/99-no-automount.rules
```

写入以下内容：

```bash
ENV{ID_FS_UUID}=="A236E77336E746BF", ENV{UDISKS_IGNORE}="1"
```

保存后执行：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

这会告诉系统：对 UUID 为 `A236E77336E746BF` 的设备（你的硬盘）**不要自动挂载**。

------

### 🔧 方法 2：如果使用桌面图形环境，也可以在设置中关闭自动挂载

在 Raspberry Pi OS 桌面上：

- 打开“文件管理器”
- 菜单栏中点击 `Edit -> Preferences -> Volume Management`
- 取消勾选 “Mount removable media automatically when they are inserted”

------

## ✅ 最终效果

- 开机时硬盘将**只被挂载到你配置的 `/home/pi/nvme`**
- 不会再有图形界面自动挂载到 `/home/RasPi` 的情况
- 无论通过 SSH 还是 GUI，都是访问统一的路径

------

## 📝 可选建议

你也可以顺便把自动挂载的目录 `/home/RasPi` 删掉（如果不再使用）：

```bash
sudo umount /home/RasPi
sudo rm -rf /home/RasPi
```

