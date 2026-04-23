```markdown
# OpenWrt for 360 V6 (Qualcomm IPQ6000)

基于 [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) 构建，针对奇虎 360 路由器 V6 定制编译。

---

## 硬件规格

| 项目 | 参数 |
|------|------|
| 芯片 | Qualcomm IPQ6000 (4× Cortex-A53 @ 1.2GHz) |
| 内存 | 512MB DDR3 |
| 存储 | 128MB NAND Flash |
| 无线 | AX1800 双频（2.4GHz 574Mbps + 5GHz 1201Mbps） |
| 有线 | 1× WAN (GbE) + 3× LAN (GbE) |
| USB | 1× USB 3.0 |

---

## 固件内置功能

### 基础系统
- **界面语言**：中文
- **主题**：Argon（深色/浅色自适应）
- **管理地址**：`http://192.168.2.1`
- **默认账户**：用户名 `root`，密码为空
- **DNS 架构**：dnsmasq(53) → SmartDNS(5335) → 上游，兜底 223.5.5.5 / 8.8.8.8

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
- **内核模块内置**：kmod-veth / kmod-br-netfilter / kmod-nf-nat 等全部编入固件
- **dockerd + docker-compose**：完整 Docker 运行环境
- **luci-app-dockerman**：图形化容器管理界面
- **数据目录**：优先存储到 U 盘（`/mnt/sda1/docker-data`），无 U 盘则存 Flash

### 其他工具
- **ttyd**：浏览器网页终端，无需 SSH 客户端即可操作命令行
- **流量监控**：nlbwmon，按设备统计上下行流量
- **定时重启**：luci-app-autoreboot
- **网络唤醒**：WoL
- **CPU 频率调节**：luci-app-cpufreq

---

## 首次开机自动安装（需联网）

路由器 WAN 口上线后，后台自动触发安装以下软件包，**无需手动操作**，安装完成后 LuCI 界面自动刷新：

| 软件包 | 说明 |
|--------|------|
| AdGuard Home | DNS 级广告过滤，安装后需手动在 LuCI 中启用并配置上游 |
| Samba4 | Windows 网络共享（SMB），配合 USB 硬盘使用 |
| bash / htop / fdisk / lsblk | 常用命令行工具 |
| smartmontools | 硬盘 S.M.A.R.T. 检测 |

> 安装日志可在 **LuCI → 状态 → 系统日志** 中查看，搜索 `install-extras`。
> 若首次安装失败（如网络不稳定），重新插拔 WAN 口网线或重启路由器后会自动重试。

---

## 刷机方法

### 从原厂固件刷入

1. 进入 360 路由器后台（通常是 `192.168.0.1`）
2. 找到固件升级页面，上传 `.bin` 格式的固件文件
3. 等待路由器重启（约 3 分钟）

### 从 OpenWrt 刷入（已有 OpenWrt 系统）

```sh
# 上传固件到路由器
scp openwrt-*.bin root@192.168.2.1:/tmp/

# SSH 登录后执行刷写
ssh root@192.168.2.1
sysupgrade -n /tmp/openwrt-*.bin
# -n 表示不保留配置，建议全新刷入
```

---

## 刷完固件后的操作步骤

### 第一步：连接路由器

- 用网线连接路由器任意 **LAN 口**，或连接 WiFi（默认 SSID 见路由器背面标签）
- 浏览器访问 `http://192.168.2.1`
- 用户名 `root`，密码留空，直接登录

### 第二步：设置 WAN 口上网

进入 **网络 → 接口 → WAN → 编辑**：

- **家用宽带（光猫拨号）**：协议选 `DHCP 客户端`，直接保存
- **路由器拨号（PPPoE）**：协议选 `PPPoE`，填入宽带账号和密码
- **固定 IP**：协议选 `静态地址`，填入 IP / 网关 / DNS

保存并应用后，等待 WAN 口获取到 IP 地址。

### 第三步：等待软件包自动安装

WAN 口上线后约 **1～3 分钟**，后台开始自动安装 AdGuard Home、Samba4 等软件包。

查看进度：
```sh
# 方法一：LuCI 系统日志（推荐）
# 打开 状态 → 系统日志，搜索 install-extras

# 方法二：SSH 实时查看
logread -f | grep install-extras
```

看到 `🎉 install-extras 全部完成` 后刷新 LuCI 页面，新菜单项会出现。

### 第四步：修改 WiFi 密码

进入 **网络 → 无线**，分别编辑 2.4GHz 和 5GHz：
- 修改 SSID（网络名称）
- 在 **无线安全** 标签页设置 WPA2 密码
- 保存并应用

### 第五步：配置 HomeProxy（可选）

进入 **服务 → HomeProxy**：
1. **节点管理** → 添加节点或导入订阅链接
2. **基本设置** → 启用 HomeProxy，选择出站节点
3. 保存并应用

### 第六步：配置 AdGuard Home（可选）

AdGuard Home 安装完成后：
1. 进入 **服务 → AdGuard Home** → 启用并启动
2. 浏览器访问 `http://192.168.2.1:3000` 完成初始化向导
3. 建议将 AdGuard Home 监听端口设为 `5353`，避免与 dnsmasq 冲突

### 第七步：挂载 U 盘（可选）

插入 USB 存储设备后：
1. 进入 **系统 → 挂载点**，确认 U 盘已自动挂载
2. 或通过 **系统 → DiskMan** 查看磁盘状态
3. Docker 数据目录会自动检测并使用 U 盘存储

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

# 重启路由器
reboot

# 查看 Docker 容器
docker ps

# 查看磁盘空间
df -h
```

---

## 注意事项

- **不要通过 LuCI 软件包管理界面安装软件**，有 JSON 解析 bug 容易卡死，请始终使用命令行 `apk add`
- Flash 空间有限（约 90MB 可用），建议 Docker 镜像存放于 U 盘
- `lucky`、`luci-app-fileassistant`、`luci-app-crontabs` 在 ImmortalWrt 官方源中不存在，无法通过 apk 安装
- 固件使用 `apk-openssl` 编译，`apk update` 可正常工作

---

## 构建信息

| 项目 | 内容 |
|------|------|
| 源码 | [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) master 分支 |
| 构建平台 | GitHub Actions (ubuntu-22.04) |
| 目标架构 | qualcommax / ipq60xx |
| 内核版本 | Linux 6.12.x |
```
