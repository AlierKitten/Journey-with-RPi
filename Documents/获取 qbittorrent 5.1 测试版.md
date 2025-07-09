你当前使用的是 Debian 12（Bookworm），其官方仓库里默认只有 4.5.x，而 5.1.0-1 已经迁移到了 Debian 测试分支（Trixie），但还没回流到稳定版仓库里 ([packages.debian.org](https://packages.debian.org/qbittorrent?utm_source=chatgpt.com))。

------

## ✅ 方案：通过启用“testing”仓库 + apt pin 获取 5.1 测试版

1. **启用 testing (Trixie) 仓库**
    在 `/etc/apt/sources.list.d/testing.list`（你可以自定义文件名）加入以下内容：

   ```
   deb http://deb.debian.org/debian trixie main
   ```

   保存后运行：

   ```bash
   sudo apt update
   ```

2. **设置 apt pin 只从 testing 安装 qbittorrent**
    创建 `/etc/apt/preferences.d/qbittorrent`，内容如下：

   ```
   Package: qbittorrent qbittorrent-nox
   Pin: release a=trixie
   Pin-Priority: 990
   ```

   这样只会从 Trixie 拉取 qbittorrent 相关包，不会影响系统其他组件。

3. **安装/升级到 5.1 版本**：

   ```bash
   sudo apt install qbittorrent
   ```

   或者直接执行：

   ```bash
   sudo apt install -t trixie qbittorrent
   ```

4. **确认版本**

   ```bash
   qbittorrent --version
   ```

   应输出 `5.1.0`。

------

## ⚠️ 注意事项

- **系统稳定性**：Qt 库等依赖会拉入 testing 分支版本，虽影响不大，但仍可能带来兼容性风险。
- **周期性更新 testing**：建议定期运行 `sudo apt update` 和 `sudo apt upgrade`，关注 testing 仓库影响。

------

## 🛠 替代方案（安全隔离）

| 方案           | 优点                   | 缺点                   |
| -------------- | ---------------------- | ---------------------- |
| **AppImage**   | 独立运行，与系统无冲突 | 启动稍慢；不自动更新   |
| **Flatpak**    | 沙箱运行，依赖完整     | 占空间；界面隔离       |
| **编译源代码** | 最大灵活性             | 需手动打补丁、维护依赖 |

如果你仅关心获取 5.1 测试版并接受轻微风险，推荐使用上面的 **testing 仓库 + pin** 方案。这样可以继续用 `apt-get` 自动升级，而无需手动下载 `.deb`。

------

## ✅ 总结

要通过 `apt-get` 安装 qbittorrent 5.1 而不使用手动 `.deb`，你只需：

1. 加入 `trixie` 仓库；
2. 为 `qbittorrent` 设置 pin 优先级；
3. 然后正常安装它。

如果你需要 Pin 文件模板或 AppImage/Flatpak 安装指导，也可以继续问我！