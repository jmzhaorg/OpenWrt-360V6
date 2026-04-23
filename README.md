# OpenWrt for 360 V6 (Qualcomm IPQ6000)

> 基于 [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) 构建，针对奇虎 360 路由器 V6 定制编译。

---

## 📌 项目初衷

原厂固件的无线 Mesh 组网体验一般，网上现成的 OpenWrt 固件也不太符合需求。借助 AI 定制了一个只带 HomeProxy 的最简固件，体验不错，后续想进一步压榨这台路由器的性能。

折腾过程中的发现与取舍：

❌ 以下包在 ImmortalWrt 官方源中不存在，无法集成：

lucky / luci-app-lucky

luci-app-fileassistant

luci-app-crontabs

⚠️ 360 V6 仅 128MB NAND Flash（可用约 90MB），dockerd、AdGuard Home 等大体积包无法直接集成进固件

✅ 改为刷完系统后联网自动安装

✅ 重型应用（Docker/AGH）强依赖 U 盘：脚本采用“路径先行”策略，先锁定 U 盘存储路径再执行安装，防止挤爆 Flash

✅ 无 U 盘保护：若未检测到 U 盘，将跳过重型应用安装，确保系统稳定

---

## 🔧 硬件规格

| 项目 | 参数 |
|------|------|
| 芯片 | Qualcomm IPQ6000（4× Cortex-A53 @ 1.2GHz） |
| 内存 | 512MB DDR3 |
| 存储 | 128MB NAND Flash |
| 无线 | AX1800 双频（2.4GHz 574Mbps + 5GHz 1201Mbps） |
| 有线 | 1× WAN GbE + 3× LAN GbE |
| USB | 1× USB 3.0 |

---

🖥️ 基础系统
界面语言：中文

主题：Argon（深色/浅色自适应）

管理地址：http://192.168.2.1

默认账户：用户名 root，密码为空

DNS 架构：dnsmasq (:53) → SmartDNS (:5335) → 上游，兜底 223.5.5.5 / 8.8.8.8

🌐 代理
HomeProxy：基于 sing-box 的透明代理，支持 VLESS / VMess / Trojan / Hysteria2 等主流协议，开箱即用

🌍 网络
SmartDNS：监听 127.0.0.1:5335，dnsmasq 转发，防污染 + 测速优选

DDNS：支持 Cloudflare 动态域名解析

UPnP：自动端口映射（miniupnpd，基于 nftables）

IPv6：odhcp6c + odhcpd + ipv6helper，支持 DHCPv6 / SLAAC

BBR：TCP 拥塞控制算法，提升高延迟网络吞吐量

💾 存储
USB 自动挂载：支持 ext4 / NTFS3 / exFAT / FAT32 / btrfs

DiskMan：磁盘管理 LuCI 界面

🐳 Docker & AdGuard Home
Flash 空间极度有限，重型应用必须配合 U 盘使用。

路径先行策略：脚本在安装前会自动将 Docker 的 data-root 和 AGH 的 work_dir 指向 U 盘，避免在 Flash 产生任何运行数据。

Swap 支持：自动检测 U 盘根目录下的 swapfile 文件并挂载，缓解 512MB 内存运行 Docker 的压力。

docker-compose：按需手动安装到 U 盘：

```sh
mkdir -p /mnt/sda1/bin
wget -O /mnt/sda1/bin/docker-compose \
    https://github.com/docker/compose/releases/latest/download/docker-compose-linux-aarch64
chmod +x /mnt/sda1/bin/docker-compose
ln -sf /mnt/sda1/bin/docker-compose /usr/local/bin/docker-compose
```

### 🛠️ 其他工具

- **ttyd**：浏览器网页终端，无需 SSH 客户端即可操作命令行
- **流量监控**：nlbwmon，按设备统计上下行流量
- **定时重启**：luci-app-autoreboot
- **网络唤醒**：WoL
- **CPU 频率调节**：luci-app-cpufreq

---

🚀 首次开机自动安装（需联网）WAN 口上线后，后台自动触发安装以下软件包。软件包说明AdGuard Home需检测到 U 盘。安装后配置与运行数据自动指向 U 盘，支持复用旧配置。Samba4Windows 网络共享（SMB），配合 USB 硬盘使用。dockerd + luci-app-dockerman仅检测到 U 盘时安装。数据目录强制指向 U 盘。bash / htop / fdisk / lsblk常用命令行工具。smartmontools硬盘 S.M.A.R.T. 检测。🔍 安装日志：LuCI → 状态 → 系统日志，搜索 install-extras🔁 安装失败：若因网络不稳定导致失败，重新插拔 WAN 口网线或重启后会自动重试。

| 软件包 | 说明 |
|--------|------|
| `AdGuard Home` | DNS 级广告过滤，安装后需在 LuCI 中启用并配置上游 |
| `Samba4` | Windows 网络共享（SMB），配合 USB 硬盘使用 |
| `dockerd` + `luci-app-dockerman` | **仅检测到 U 盘时安装**，数据目录自动指向 U 盘 |
| `bash` / `htop` / `fdisk` / `lsblk` | 常用命令行工具 |
| `smartmontools` | 硬盘 S.M.A.R.T. 检测 |

> 🔍 **安装日志**：`LuCI → 状态 → 系统日志`，搜索 `install-extras`  
> 🔁 **安装失败**：若因网络不稳定导致失败，重新插拔 WAN 口网线或重启路由器后会自动重试

---

## 🔧 刷机方法

### 方式一：从不死 Boot（Breed）刷入（推荐新手）

1. **进入 Breed 界面**
   - 路由器断电 → 按住 `Reset` 键不放 → 插电 → 持续按住约 8 秒
   - 浏览器访问 `192.168.1.1`（确保电脑设为自动获取 IP）

2. **备份（重要！）**
   - `备份与恢复` → 备份 `EEPROM` 和 `完整固件`（救砖必备）

3. **刷写固件**
   - `固件更新` → 选择 `squashfs-factory.ubi` 文件
   - ✅ 勾选「刷写后重启」（如有）
   - ⚠️ **不要勾选「保留配置」**（跨固件/大版本升级必须清空）

4. **等待重启**
   - 进度条走完 + 路由器自动重启（约 2~3 分钟）
   - 首次启动较慢，耐心等待指示灯稳定

### 方式二：从 OpenWrt 刷入（已有系统升级）

#### 🖥️ Web 界面方式
```
菜单 → 系统 → 备份/升级 → 刷写固件...
→ 上传 *-sysupgrade.bin 文件
→ 取消勾选「保留配置」（强烈建议）
→ 点击「刷写固件」，等待自动重启
```

#### 💻 SSH 命令行方式
```sh
# 1. 上传固件到 /tmp（tmpfs 内存空间）
scp openwrt-xxx-sysupgrade.bin root@192.168.2.1:/tmp/

# 2. SSH 登录并执行刷写（-n = 不保留配置）
ssh root@192.168.2.1
sysupgrade -n /tmp/openwrt-xxx-sysupgrade.bin

# 3. 等待重启，勿中断连接
```

> ⚠️ **通用注意事项**：
> - 🔌 刷机全程使用原装电源，避免断电变砖
> - 🖥️ 优先用**网线**连接，避免 WiFi 中断
> - 🧹 刷完后建议 `Reset` 清空配置，避免旧配置冲突

---

🎯 刷完固件后的操作步骤
第一步：连接路由器
用网线连接路由器任意 LAN 口，浏览器访问 http://192.168.2.1，用户名 root，密码留空。

第二步：设置 WAN 口上网
进入 网络 → 接口 → WAN → 编辑，设置好拨号或 DHCP，确保路由器已连接互联网。

第三步：插好 U 盘并确认挂载
插入 ext4 格式的 U 盘（推荐）。进入 系统 → 挂载点 确认挂载为 /mnt/sda1（或 sda/usb 等）。

💡 建议：在 U 盘根目录创建 Swap 文件以提升稳定性：
dd if=/dev/zero of=/mnt/sda1/swapfile bs=1M count=512 && mkswap /mnt/sda1/swapfile

第四步：等待软件包自动安装
WAN 口上线且 U 盘就绪后，脚本会自动开始工作。

Bash
# 实时查看安装进度
logread -f | grep install-extras
看到 🎉 install-extras 全部完成 后刷新 LuCI 页面即可。

## 📋 常用命令

```sh
# 软件包管理
apk update                    # 更新软件包索引
apk add <包名>                # 安装软件包
apk info                      # 查看已安装软件包
apk search <关键词>           # 搜索软件包

# 系统操作
logread | tail -50            # 查看系统日志
logread -f                    # 实时跟踪日志
reboot                        # 重启路由器
df -h                         # 查看磁盘空间

# Docker
docker ps                     # 查看运行中的容器
docker images                 # 查看本地镜像
docker-compose up -d          # 启动 compose 项目（需手动安装）
```

---

⚠️ 注意事项
❌ 严禁在无 U 盘时通过 apk add 安装 dockerd 或 adguardhome，这会瞬间耗尽 Flash 空间导致系统崩溃。

💾 存储建议：所有镜像和插件日志建议存放于 U 盘。若 Flash 空间依然不足，建议在挂载点中将 U 盘分区挂载为 /overlay 实现彻底扩容。

❌ LuCI 软件包界面：存在 JSON 解析 bug 容易卡死，请始终使用命令行 apk add 安装新软件。

🔐 固件特性：已集成 apk-openssl，apk update 走 HTTPS 更加安全。

---

## 🏗️ 构建信息

| 项目 | 内容 |
|------|------|
| 源码 | [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) master 分支 |
| 构建平台 | GitHub Actions（ubuntu-22.04） |
| 目标架构 | qualcommax / ipq60xx |
| 内核版本 | Linux 6.12.x |

---

> 💡 **反馈与支持**：如遇问题，请先查看系统日志 `logread | grep -i error`，或提交 Issue 时附上固件版本 + 操作步骤 + 日志截图。
