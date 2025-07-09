非常好！

从你贴出来的内容：

```
inet 192.168.85.251/24 ...
```

说明你已经**成功通过手机 USB 网络共享获取到了 IPv4 地址**，设备现在已经联网 ✅。

------

### 接下来你可以做的：

#### ✅ 1. 测试联网是否畅通：

```bash
ping -c 3 8.8.8.8
ping -c 3 google.com
```

------

#### ✅ 2. 固定 USB 网卡接口名（可选，但强烈建议）

因为你发现每次重启设备，`enx...` 接口名都会变，为了避免每次都要改 `netplan`，我们现在就来固定它的名字为 `usbnet0`：

##### 步骤一：获取 MAC 地址（你已经有了）

比如：`16:39:0f:3d:32:2e`

##### 步骤二：创建 udev 规则

```bash
sudo nano /etc/udev/rules.d/70-usb-net.rules
```

填入以下内容（用你的 MAC 地址）：

```bash
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="16:39:0f:3d:32:2e", NAME="usbnet0"
```

保存后退出。

##### 步骤三：修改 netplan 文件

```bash
sudo nano /etc/netplan/99-usb-tether.yaml
```

将接口名改为 `usbnet0`：

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    usbnet0:
      dhcp4: true
```

保存后退出，并运行：

```bash
sudo netplan apply
```

##### 步骤四：重启验证

```bash
sudo reboot
```

之后再执行：

```bash
ip a
```

确认 `usbnet0` 是否自动启用并获得 IP。

------

需要我生成完整文件内容，你只复制粘贴也可以告诉我。你现在的系统已经恢复联网，可以继续配置和下载需要的软件了。需要我帮你回退到树莓派官方桌面吗？