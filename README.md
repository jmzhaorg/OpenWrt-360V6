# OpenWrt for 360 V6 (Qualcomm IPQ6000)

> 基于 [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) 构建，针对奇虎 360 路由器 V6 定制编译。

---

## 📌 项目初衷

原厂固件的无线 Mesh 组网体验一般，网上现成的 OpenWrt 固件也不太符合需求。借助 AI 定制了一个只带 HomeProxy 的最简固件，体验不错，后续想进一步压榨这台路由器的性能。

### 🔍 折腾过程中的发现与取舍

**❌ 以下包在 ImmortalWrt 官方源中不存在，无法集成：**
- `lucky` / `luci-app-lucky`
- `luci-app-fileassistant`
- `luci-app-crontabs`

**⚠️ 存储限制：**
360 V6 仅 128MB NAND Flash（可用约 90MB），`dockerd`、`AdGuard Home` 等大体积包无法直接集成进固件。

**✅ 我的解决方案：**
| 策略 | 说明 | 目的 |
|------|------|------|
| **联网自动安装** | 刷完系统后，首次联网自动拉取额外组件 | 避免固件臃肿，按需加载 |
| **路径先行** | 脚本先锁定 U 盘存储路径，再执行安装 | 防止重型应用挤爆 Flash |
| **无 U 盘保护** | 未检测到 U 盘时，跳过 `dockerd` / `AdGuard Home` 安装 | 确保系统稳定不崩溃 |
| **Compose 手动装** | `docker-compose` 不提供自动安装，附手动指引 | 把选择权交给用户 |

> 💡 **核心理念**：最小化固件 + 按需扩展。让路由器先「能稳定跑起来」，再让用户根据需求「想装什么装什么」。

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

## 📦 固件内置功能

### 🖥️ 基础系统
- **界面语言**：中文
- **主题**：Argon（深色/浅色自适应）
- **管理地址**：`http://192.168.2.1`
- **默认账户**：用户名 `root`，密码为空
- **DNS 架构**：`dnsmasq (:53)` → `SmartDNS (:5335)` → 上游，兜底 `223.5.5.5` / `8.8.8.8`

### 🌐 代理
- **HomeProxy**：基于 sing-box 的透明代理，支持 VLESS / VMess / Trojan / Hysteria2 等主流协议，开箱即用

### 🌍 网络
- **SmartDNS**：监听 `127.0.0.1:5335`，dnsmasq 转发，防污染 + 测速优选
- **DDNS**：支持 Cloudflare 动态域名解析
- **UPnP**：自动端口映射（miniupnpd，基于 nftables）
- **IPv6**：odhcp6c + odhcpd + ipv6helper，支持 DHCPv6 / SLAAC
- **BBR**：TCP 拥塞控制算法，提升高延迟网络吞吐量

### 💾 存储
- **USB 自动挂载**：支持 ext4 / NTFS3 / exFAT / FAT32 / btrfs
- **DiskMan**：磁盘管理 LuCI 界面

### 🐳 Docker & AdGuard Home
> ⚠️ Flash 空间极度有限，重型应用必须配合 U 盘使用。

- **路径先行策略**：脚本在安装前自动将 Docker 的 `data-root` 和 AGH 的 `work_dir` 指向 U 盘，避免在 Flash 产生任何运行数据
- **Swap 自动创建**：检测到 U 盘后自动在 U 盘根目录创建并激活 256MB `swapfile`，缓解 512MB 内存运行 Docker 的压力
- **docker-compose**：按需手动安装到 U 盘：
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

## 🚀 首次开机自动安装（96-install-extras Final v3）

### 工作原理

脚本以两层结构编译进固件，**全程无需人工操作**：

```
96-install-extras            ← 编译进固件，系统启动时执行一次
└── 安装 WAN UP 触发器
    └── 99-install-extras    ← WAN / WAN6 / PPPoE 接口 UP 时触发
        └── run.sh           ← 真正的安装逻辑
```

触发条件：WAN 接口 UP 且 `/etc/install-extras-done` 不存在时自动运行。全部成功后写入该标志文件，之后不再重复执行。任意包安装失败时不写标志文件，下次 WAN UP 自动重试。

### 执行流程详解

**① 清理无效软件源**
删除 repositories 中含 `/video/` 的条目，避免 `apk update` 报错。

**② 等待网络就绪（最长 3 分钟）**
轮询 `223.5.5.5`（阿里）和 `8.8.8.8`（Google），超时后退出等待下次重试。同步注入公共 DNS，绕过 SmartDNS 启动时序问题。

**③ 检测 U 盘（最长等待 30 秒）**
依次探测以下挂载点，找到即停止：
```
/mnt/sda1  /mnt/sda  /mnt/usb  /mnt/mmcblk0p1
```
**无 U 盘时绝不 fallback 到 /overlay**，Docker / AdGuardHome 全部跳过，保护 Flash。

**④ I/O 与内存优化（有 U 盘时）**

| 参数 | 值 | 作用 |
|------|----|------|
| I/O 调度器 | `mq-deadline` / `deadline` | 降低读写延迟，延长 U 盘寿命 |
| `vm.swappiness` | `10` | 减少非必要换页，降低 U 盘写入压力 |
| `vm.min_free_kbytes` | `16384`（16MB） | 保留内存缓冲，防止 OOM |

**⑤ Swap 自动管理（有 U 盘时）**
- 已有 `swapfile`：直接激活
- 没有且内存 < 1GB：自动创建 256MB Swap 并激活

```
U 盘根目录/
└── swapfile        ← 自动创建，256MB，权限 600
```

**⑥ apk update（带 HTTPS→HTTP 自动降级）**
若 HTTPS 镜像返回包数 < 800（判断为异常），自动切换 HTTP 重试。

**⑦ Flash 空间预检（三档保护）**

安装前检测 `/overlay` 剩余空间，按三档决策：

| 剩余空间 | 策略 |
|----------|------|
| > 30MB | 正常安装，含全部 LuCI 插件 |
| 15MB ~ 30MB | 跳过可选 LuCI（dockerman / adguardhome），服务仍可用 |
| < 15MB | 跳过全部 LuCI，仅装服务端二进制 |

> Flash 紧张时 Docker 可用 CLI 管理，AdGuardHome 可用 `:3000` Web 界面管理。

**⑧ 基础工具（必装，写 Flash）**
```
smartmontools  bash  htop  fdisk  lsblk
```

**⑨ Samba（写 Flash，LuCI 视 Flash 情况）**

`samba4-server` 必装。装完后自动执行 dbus 重载 + avahi 重启，修复热安装后的 **avahi 崩溃循环** 已知 bug。

> 根因：dbus 启动时 avahi 策略文件尚不存在，热装后必须重载 dbus 才能识别新策略，否则 avahi 会以每 5 秒一次的频率崩溃重启。重启路由器后问题自动消失，但热安装时必须手动修复，脚本已自动处理。

**⑩ 重型应用（仅有 U 盘时，绝不写 Flash）**

*Docker：*
先写 `daemon.json` 锁定数据路径，再安装包，防止 dockerd 首次启动将数据写入 Flash。`dockerd` 单独以 `--force-broken-world` 安装，绕过 kmod 版本依赖冲突。

```json
{
  "data-root": "<U盘>/docker-data",
  "log-driver": "json-file",
  "log-opts": {"max-size": "5m", "max-file": "2"}
}
```

*AdGuardHome：*
配置和数据全部迁移至 U 盘。有旧数据保护机制——重刷固件后 U 盘上的旧配置不会被覆盖，直接复用。

安装完成后 U 盘目录结构如下：

```
U 盘根目录/
├── swapfile                  ← 256MB 交换文件
├── docker-data/              ← Docker 镜像、容器、volumes
└── AdGuardHome/
    ├── config/
    │   └── adguardhome.yaml  ← AGH 配置文件
    └── data/                 ← 数据库、查询日志等
```

### 安装的软件包汇总

| 软件包 | 存储位置 | 安装条件 |
|--------|----------|----------|
| `smartmontools` `bash` `htop` `fdisk` `lsblk` | Flash | 无条件必装 |
| `samba4-server` | Flash | 无条件必装 |
| `luci-app-samba4` `luci-i18n-samba4-zh-cn` | Flash | Flash > 30MB |
| `docker` `dockerd` | Flash（二进制）/ U 盘（数据） | 有 U 盘 |
| `luci-app-dockerman` `luci-i18n-dockerman-zh-cn` | Flash | 有 U 盘 且 Flash > 30MB |
| `adguardhome` | Flash（二进制）/ U 盘（数据+配置） | 有 U 盘 |
| `luci-app-adguardhome` | Flash | 有 U 盘 且 Flash > 30MB |

> 🔍 **查看安装日志**：`LuCI → 状态 → 系统日志`，搜索 `install-extras`
> 🔁 **安装失败自愈**：网络不稳定导致失败时，重插网线或重启路由器后自动重试

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
   - ⚠️ **不要勾选「保留配置」**（跨固件 / 大版本升级必须清空）

4. **等待重启**
   - 进度条走完后路由器自动重启（约 2~3 分钟）
   - 首次启动较慢，耐心等待指示灯稳定

### 方式二：从 OpenWrt 刷入（已有系统升级）

#### 🖥️ Web 界面方式
```
菜单 → 系统 → 备份/升级 → 刷写固件
→ 上传 *-sysupgrade.bin 文件
→ 取消勾选「保留配置」（强烈建议）
→ 点击「刷写固件」，等待自动重启
```

#### 💻 SSH 命令行方式
```sh
# 上传固件到 /tmp（内存，不占 Flash）
scp openwrt-xxx-sysupgrade.bin root@192.168.2.1:/tmp/

# SSH 登录并刷写（-n = 不保留配置）
ssh root@192.168.2.1
sysupgrade -n /tmp/openwrt-xxx-sysupgrade.bin
```

> ⚠️ **通用注意事项**：
> - 🔌 刷机全程使用原装电源，避免断电变砖
> - 🖥️ 优先用**网线**连接，避免 WiFi 中断
> - 🧹 刷完后建议清空配置，避免旧配置冲突

---

## 🎯 刷完固件后的操作步骤

### 第一步：连接路由器
用网线连接路由器任意 **LAN 口**，浏览器访问 `http://192.168.2.1`，用户名 `root`，密码留空。

### 第二步：设置 WAN 口上网
进入 **网络 → 接口 → WAN → 编辑**，配置好拨号或 DHCP，确保路由器已连接互联网。

### 第三步：插好 U 盘并确认挂载
插入 **ext4 格式**的 U 盘（推荐 SanDisk / 三星等品牌）。进入 **系统 → 挂载点** 确认已挂载为 `/mnt/sda1`。

> ⚠️ FAT32 / exFAT 格式 U 盘无法存放 Docker 数据（不支持 Linux 文件权限），建议提前在电脑上格式化为 ext4。

### 第四步：等待软件包自动安装
WAN 口上线且 U 盘就绪后，脚本自动开始工作，无需任何操作。

```sh
# 实时查看安装进度
logread -f | grep install-extras
```

看到以下日志说明全部完成：
```
🎉 Final v3 部署完成！Flash 使用率: XX%
```

刷新 LuCI 页面后新安装的插件即可出现在菜单中。

### 第五步：后续配置（可选）
- **修改 WiFi 密码**：`网络 → 无线`
- **配置 HomeProxy**：`服务 → HomeProxy`
- **初始化 AdGuard Home**：`服务 → AdGuard Home` 或浏览器访问 `http://192.168.2.1:3000`
- **配置 Samba 共享**：`服务 → 网络共享`

### 重刷固件后恢复（U 盘数据保留）
重刷固件后，U 盘上的 Docker 数据和 AdGuardHome 配置会被自动复用，无需重新配置。只需触发一次重新安装：

```sh
rm -f /etc/install-extras-done && reboot
```

---

## 📋 常用命令

```sh
# ===== 软件包管理 =====
apk update                    # 更新软件包索引
apk add <包名>                # 安装软件包
apk info                      # 查看已安装软件包
apk search <关键词>           # 搜索软件包
apk cache clean               # 清理缓存（Flash 紧张时使用）

# ===== 系统状态 =====
logread | grep install-extras # 查看自动安装日志
logread -f                    # 实时跟踪系统日志
df -h                         # 查看磁盘 / Flash 空间
free -h                       # 查看内存与 Swap 使用
reboot                        # 重启路由器

# ===== 自动安装控制 =====
# 重新触发自动安装（换了 U 盘或重刷固件后）
rm -f /etc/install-extras-done && reboot

# ===== Docker =====
docker ps                     # 查看运行中的容器
docker images                 # 查看本地镜像
docker stats                  # 查看容器资源占用
docker system prune           # 清理无用镜像和容器
```

---

## ⚠️ 注意事项

- ❌ **严禁在无 U 盘时手动 `apk add dockerd` 或 `adguardhome`**，会瞬间耗尽 Flash 空间导致系统不稳定
- 💾 **U 盘格式**：必须为 ext4，FAT32 / exFAT 不支持 Docker 所需的 Linux 文件权限
- ❌ **LuCI 软件包界面**：存在 JSON 解析 bug 容易卡死，请始终使用命令行 `apk add` 安装新软件
- 🔌 **不要热拔 U 盘**：系统运行时强行拔出会导致 ext4 journal 错误，安装了 Docker / AdGuardHome 后热拔会直接导致服务崩溃
- 🔐 **HTTPS 软件源**：已集成 `apk-openssl`，`apk update` 走 HTTPS 更加安全

---

## 🏗️ 构建信息

| 项目 | 内容 |
|------|------|
| 源码 | [ImmortalWrt](https://github.com/immortalwrt/immortalwrt) master 分支 |
| 构建平台 | GitHub Actions（ubuntu-22.04） |
| 目标架构 | qualcommax / ipq60xx |
| 内核版本 | Linux 6.12.x |

---
