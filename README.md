# OpenWrt for 360 V6 (Qualcomm IPQ6000)

基于 [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) 构建，针对奇虎 360 路由器 V6 定制编译。

## 写在前面

原厂固件的无线 Mesh 组网体验一般，网上找的现成 OpenWrt 固件也不太合适。后来借助 AI 定制了一个只带 HomeProxy 的最简固件，用着还不错，但慢慢想进一步压榨这台路由器的性能。

折腾过程中发现不少包在 ImmortalWrt 官方源里根本不存在，只能放弃：

- `lucky` / `luci-app-lucky` — 第三方包，官方源无
- `luci-app-fileassistant` — 同上
- `luci-app-crontabs` — 同上

另外受限于 360 V6 只有 128MB NAND Flash（可用约 90MB），`dockerd`、`docker-compose`、AdGuard Home 等体积较大的包无法直接集成进固件，改为刷完系统后联网自动安装。**没有 U 盘时不安装 dockerd**，docker-compose 需要时由用户自行安装（需 U 盘空间充足或扩展 overlay）。

---

## 硬件规格

| 项目 | 参数 |
|------|------|
| 芯片 | Qualcomm IPQ6000（4× Cortex-A53 @ 1.2GHz） |
| 内存 | 512MB DDR3 |
| 存储 | 128MB NAND Flash |
| 无线 | AX1800 双频（2.4GHz 574Mbps + 5GHz 1201Mbps） |
| 有线 | 1× WAN GbE + 3× LAN GbE |
| USB | 1× USB 3.0 |

---

## 固件内置功能

### 基础系统

- **界面语言**：中文
- **主题**：Argon（深色/浅色自适应）
- **管理地址**：`http://192.168.2.1`
- **默认账户**：用户名 `root`，密码为空
- **DNS 架构**：dnsmasq（53）→ SmartDNS（5335）→ 上游，兜底 223.5.5.5 / 8.8.8.8

### 代理

- **HomeProxy**：基于 sing-box 的透明代理，支持 VLESS / VMess / Trojan / Hysteria2 等主流协议，开箱即用

### 网络

- **SmartDNS**：监听 127.0.0.1:5335，dnsmasq 转发，防污染 + 测速优选
- **DDNS**：支持 Cloudflare 动态域名解析
- **UPnP**：自动端口映射（miniupnpd，基于 nftables）
- **IPv6**：odhcp6c + odhcpd + ipv6helper，支持 DHCPv6 / SLAAC
- **BBR**：TCP 拥塞控制算法，提升高延迟网络吞吐量

### 存储

- **USB 自动挂载**：支持 ext4 / NTFS3 / exFAT / FAT32 / btrfs
- **DiskMan**：磁盘管理 LuCI 界面

### Docker

Flash 空间有限，固件内仅包含 `docker` 客户端，不含 `docker-compose`。`dockerd` 在检测到 U 盘后由首次开机脚本自动安装，Docker 数据目录也会自动指向 U 盘。

如需 `docker-compose`，手动安装到 U 盘：

```sh
mkdir -p /mnt/sda1/bin
wget -O /mnt/sda1/bin/docker-compose \
    https://github.com/docker/compose/releases/latest/download/docker-compose-linux-aarch64
chmod +x /mnt/sda1/bin/docker-compose
ln -sf /mnt/sda1/bin/docker-compose /usr/local/bin/docker-compose
```

### 其他工具

- **ttyd**：浏览器网页终端，无需 SSH 客户端即可操作命令行
- **流量监控**：nlbwmon，按设备统计上下行流量
- **定时重启**：luci-app-autoreboot
- **网络唤醒**：WoL
- **CPU 频率调节**：luci-app-cpufreq

---

## 首次开机自动安装（需联网）

WAN 口上线后，后台自动触发安装以下软件包，**无需手动操作**，安装完成后 LuCI 界面自动刷新。

| 软件包 | 说明 |
|--------|------|
| AdGuard Home | DNS 级广告过滤，安装后需在 LuCI 中启用并配置上游 |
| Samba4 | Windows 网络共享（SMB），配合 USB 硬盘使用 |
| dockerd + luci-app-dockerman | 仅在检测到 U 盘时安装，数据目录自动指向 U 盘 |
| bash / htop / fdisk / lsblk | 常用命令行工具 |
| smartmontools | 硬盘 S.M.A.R.T. 检测 |

> 安装日志：**LuCI → 状态 → 系统日志**，搜索 `install-extras`。
> 若首次安装失败（如网络不稳定），重新插拔 WAN 口网线或重启路由器后会自动重试。

---

## 刷机方法

### 从不死boot刷入

1. 一直按rest键插电，8秒左右进入界面，192.168.1.1
2. 上传squashfs-factory .ubi文件
3. 等待路由器重启（约 2 分钟）

### 从 OpenWrt 刷入（已有 OpenWrt 系统）

```sh
# 上传固件到路由器
菜单-系统-备份与更新-拉到最底-更新固件-刷写固件，选择sysupgrade.bin文件上传

# SSH 登录后执行刷写
ssh root@192.168.2.1
sysupgrade -n /tmp/openwrt-*.bin
# -n 表示不保留配置，建议全新刷入
```

---

## 刷完固件后的操作步骤

### 第一步：连接路由器

用网线连接路由器任意 **LAN 口**，或连接 WiFi（默认 SSID 见路由器背面标签），浏览器访问 `http://192.168.2.1`，用户名 `root`，密码留空。

### 第二步：设置 WAN 口上网

进入 **网络 → 接口 → WAN → 编辑**：

- **家用宽带（光猫拨号）**：协议选 `DHCP 客户端`，直接保存
- **路由器拨号（PPPoE）**：协议选 `PPPoE`，填入宽带账号和密码
- **固定 IP**：协议选 `静态地址`，填入 IP / 网关 / DNS

保存并应用后，等待 WAN 口获取到 IP 地址。

### 第三步：等待软件包自动安装

WAN 口上线后约 **1～3 分钟**，后台开始自动安装 AdGuard Home、Samba4 等软件包。

```sh
# 实时查看安装进度
logread -f | grep install-extras
```

看到 `🎉 install-extras 全部完成` 后刷新 LuCI 页面，新菜单项会出现。

### 第四步：修改 WiFi 密码

进入 **网络 → 无线**，分别编辑 2.4GHz 和 5GHz，在 **无线安全** 标签页设置 WPA2 密码。

### 第五步：配置 HomeProxy（可选）

进入 **服务 → HomeProxy**：

1. **节点管理** → 添加节点或导入订阅链接
2. **基本设置** → 启用 HomeProxy，选择出站节点
3. 保存并应用

### 第六步：配置 AdGuard Home（可选）

1. 进入 **服务 → AdGuard Home** → 启用并启动
2. 浏览器访问 `http://192.168.2.1:3000` 完成初始化向导
3. 建议将 AdGuard Home 监听端口设为 `5353`，避免与 dnsmasq 冲突

### 第七步：挂载 U 盘（可选）

插入 USB 存储设备后，进入 **系统 → 挂载点** 确认已自动挂载，或通过 **系统 → DiskMan** 查看磁盘状态。Docker 数据目录会自动检测并使用 U 盘存储。

---

## 常用命令

```sh
# 更新软件包索引
apk update

# 安装软件包
apk add <包名>

# 查看已安装软件包
apk info

# 搜索软件包
apk search <关键词>

# 查看系统日志
logread | tail -50

# 实时跟踪日志
logread -f

# 重启路由器
reboot

# 查看 Docker 容器
docker ps

# 查看磁盘空间
df -h
```

---

## 注意事项

- **不要通过 LuCI 软件包管理界面安装软件**，存在 JSON 解析 bug 容易卡死，请始终使用命令行 `apk add`
- Flash 空间有限（约 90MB 可用），建议 Docker 镜像存放于 U 盘
- `lucky`、`luci-app-fileassistant`、`luci-app-crontabs` 在 ImmortalWrt 官方源中不存在，无法通过 apk 安装
- 固件使用 `apk-openssl` 编译，`apk update` 可正常工作

---

## 构建信息

| 项目 | 内容 |
|------|------|
| 源码 | [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) master 分支 |
| 构建平台 | GitHub Actions（ubuntu-22.04） |
| 目标架构 | qualcommax / ipq60xx |
| 内核版本 | Linux 6.12.x |
