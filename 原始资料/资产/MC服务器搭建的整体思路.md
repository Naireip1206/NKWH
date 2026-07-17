# MC服务器搭建的整体思路

> 随着 AI Agent 的飞速发展，搭建一个 MC 服务器的门槛越来越低，甚至搭建者不需要任何相关知识便能完成搭建。本文的目的是避免"有枪不会使"的情况出现——让你在享受便利的同时，真正理解每一步在做什么。

---

## 一、选择硬件

### 物理机自建

自建物理机是自由度最高的方案。MC 服务器最看重的两个硬件指标：

| 硬件 | 重要性 | 说明 |
|------|--------|------|
| **CPU 单核算力** | ⭐⭐⭐⭐⭐ | MC 的主逻辑（实体运算、红石、区块加载）基本是单线程的，单核性能直接决定服务器能承载多少玩家和复杂程度 |
| **内存 (RAM)** | ⭐⭐⭐⭐⭐ | 每个玩家、每个加载的区块、每个实体都占用内存。内存不足会导致卡顿甚至崩溃 |
| **硬盘读写速度** | ⭐⭐⭐ | 影响世界加载速度、区块保存速度。NVMe SSD >> SATA SSD >> HDD |
| **网络带宽 & 延迟** | ⭐⭐⭐⭐ | 影响玩家的连接稳定性与延迟体验 |

**CPU 选购建议（截至 2026 年）：**

- **入门级（~5 人小型服）**：Intel i3 或 AMD Ryzen 3，单核性能够用，成本低
- **进阶级（10-30 人）**：Intel i5 / AMD Ryzen 5，单核性能优秀，性价比最高
- **高端（30-100+ 人）**：Intel i7 / i9 或 AMD Ryzen 7 / 9，追求极致单核性能
- **特殊需求（多开或大服）**：AMD Ryzen 9 或 Intel Xeon，核心数多可同时跑多个实例

> 💡 **要点**：MC 不看核心数（8 核够用），单核频率和 IPC（每时钟周期指令数）才是王道。选 CPU 时优先看单核跑分（如 PassMark 单核分数）。

**内存建议：**

| 场景 | 建议内存 | 备注 |
|------|---------|------|
| 原版小服（1-5 人） | 4-6 GB | 含系统开销 |
| 原版中等服（10-30 人） | 8-12 GB | |
| 插件服（Paper/Bukkit） | 8-16 GB | 取决于插件数量 |
| 整合包/Mod 服 | 12-32 GB | 大型 Mod 包（如 ATM 系列）需更多 |
| 大型公开服 | 32 GB+ | 含预生成地形缓存 |

### 云服务器

各大云厂商均提供适合跑 MC 的云服务器实例：

| 厂商 | 适合方案 | 特点 |
|------|---------|------|
| **阿里云** | 轻量应用服务器 / ECS | 国内延迟低，轻量服务器性价比高 |
| **腾讯云** | 轻量应用服务器 / CVM | 类似阿里云，新用户优惠多 |
| **华为云** | HECS / ECS | 稳定性好，学生优惠 |
| **AWS** | Lightsail / EC2 | 国际服首选，支持按需付费 |
| **Vultr / Linode** | 高频 CPU 实例 | 单核性能优化，适合 MC |

**云服务器选购要点：**
- 选择 **高频 CPU 实例**（如阿里云的计算型、AWS 的 compute-optimized）
- **按量付费 vs 包年包月**：测试期用按量，稳定后包年
- **带宽**：5-10 Mbps 足够 20-50 人，大型公开服需 100 Mbps 以上
- **防御 DDoS**：国内厂商一般自带基础防御，海外可用 TCPShield

---

## 二、环境准备

### Java 版本

运行标准的 Minecraft Java Edition 服务器需要对应版本的 Java：

| Minecraft 版本 | 所需 Java | 获取方式 |
|---------------|----------|---------|
| 1.0 - 1.16.5 | Java 8 (JDK 8) | `sudo apt install openjdk-8-jdk` |
| 1.17 - 1.20.4 | Java 17 (JDK 17) | `sudo apt install openjdk-17-jdk` |
| 1.20.5 - 1.21.x | Java 21 (JDK 21) | `sudo apt install openjdk-21-jre-headless` |

> ⚠️ **注意**：Mojang 频繁更新 Java 依赖要求，搭建前务必查阅 [Minecraft 官方文档](https://minecraft.wiki) 确认最新版本的 Java 要求。建议安装主版本后也准备好最新 LTS 版本的 JDK。

**安装 Java 的通用方式（Linux）：**

```bash
# 查看已安装的 Java 版本
java -version

# 安装 OpenJDK（Ubuntu/Debian）
sudo apt update
sudo apt install openjdk-21-jre-headless   # 仅运行时
sudo apt install openjdk-21-jdk            # 包含开发工具

# 切换默认 Java 版本（多版本共存时）
sudo update-alternatives --config java
```

### 操作系统选择

| 系统 | 适用场景 | 优缺点 |
|------|---------|--------|
| **Ubuntu Server 22.04/24.04 LTS** | 推荐首选 | 社区活跃，文档多，稳定性好 |
| **Debian 12** | 稳定至上 | 比 Ubuntu 更保守，资源占用更少 |
| **CentOS Stream / Rocky Linux** | 企业级 | RHEL 系，习惯 yum 的可用 |
| **Windows Server** | 不会 Linux 的选择 | 操作简单但资源占用大，不稳定 |
| **Docker** | 隔离环境 | 快速部署，方便迁移和备份 |

### 基础安全配置

```bash
# 创建非 root 用户
adduser mcserver
usermod -aG sudo mcserver
su - mcserver

# 防火墙（以 ufw 为例）
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 25565/tcp     # Minecraft
sudo ufw allow 25565/udp     # Minecraft（一些情况需要）
sudo ufw enable

# 使用 screen/tmux 保持后台运行
sudo apt install screen tmux
```

### 选择合适的服务端核心

不同核心的侧重点完全不同：

| 核心 | 类型 | 特点 |
|------|------|------|
| **Vanilla** | 原版 | Mojang 官方 jar，纯净无修改 |
| **Paper** | 优化版 | 最流行的性能优化核心，插件兼容性好 |
| **Purpur** | 高自定义 | 基于 Paper，可配置更多原版机制 |
| **Fabric** | 轻量 Mod 加载器 | 启动快，Mod 更新及时，兼容性好 |
| **Quilt** | Fabric 分支 | 社区分叉，更开放的治理 |
| **Forge** | 重量级 Mod 加载器 | 兼容最多 Mod，但启动慢、优化差 |
| **NeoForge** | Forge 分支 | Forge 分裂后的新主线 |
| **Spigot** | 插件兼容 | Paper 的前身，已被 Paper 取代 |
| **Velocity** | 代理端 | 高性能代理，取代 BungeeCord |
| **BungeeCord** | 代理端 | 老牌代理，用于跨服 |

**如何选择：**
- **纯生存小服（≤10 人）**：Paper 或 Vanilla
- **插件生存服**：Paper 或 Purpur
- **轻量 Mod 服**：Fabric
- **大型 Mod 包（ATM、GTNH 等）**：Forge 或 NeoForge
- **跨服网络**：Velocity + 若干个后端实例

---

## 三、配置启动文件

### 基础启动命令

最简单的 MC 服务器启动命令：

```bash
java -jar server.jar nogui
```

`nogui` 参数表示不启动图形界面（纯命令行模式），在服务器上推荐使用。

### 进阶启动配置（推荐）

实际生产环境中，JVM 的启动参数对性能影响巨大。以下是一份经过实战检验的配置模板：

```bash
#!/bin/bash
# start.sh — MC 服务器启动脚本

# ===== 配置区域 =====
JAR_NAME="paper-1.21.1-130.jar"    # 你的服务端 jar 文件名
MEM_MIN="2G"                        # 初始堆大小（Xms）
MEM_MAX="8G"                        # 最大堆大小（Xmx）
BACKUP_DIR="./backups"              # 备份目录
# ===================

# Aikar's Flags — 业界公认的 MC 服务器 JVM 优化参数
# 详见: https://docs.papermc.io/paper/aikars-flags
java -Xms$MEM_MIN -Xmx$MEM_MAX \
     -XX:+UseG1GC \
     -XX:+ParallelRefProcEnabled \
     -XX:MaxGCPauseMillis=200 \
     -XX:+UnlockExperimentalVMOptions \
     -XX:+DisableExplicitGC \
     -XX:+AlwaysPreTouch \
     -XX:G1HeapWastePercent=5 \
     -XX:G1MixedGCCountTarget=4 \
     -XX:G1MixedGCLiveThresholdPercent=90 \
     -XX:G1RSetUpdatingPauseTimePercent=5 \
     -XX:SurvivorRatio=32 \
     -XX:+PerfDisableSharedMem \
     -XX:MaxTenuringThreshold=1 \
     -Dusing.aikars.flags=https://mcflags.emc.gs \
     -Daikars.new.flags=true \
     -jar $JAR_NAME nogui

# 如果服务器关闭，自动重启
# 将上面整个 java 命令包在 while 循环中即可实现
```

### 各参数详解

| 参数 | 作用 |
|------|------|
| `-Xms` / `-Xmx` | 初始堆 / 最大堆大小。建议设为相同值避免 GC 时堆扩容的开销 |
| `-XX:+UseG1GC` | 使用 G1 垃圾回收器，MC 最适合的 GC 策略 |
| `-XX:+ParallelRefProcEnabled` | 并行处理引用，加速 GC |
| `-XX:MaxGCPauseMillis=200` | 目标 GC 暂停时间 200ms |
| `-XX:+DisableExplicitGC` | 禁用 System.gc() 调用，防止意外触发 Full GC |
| `-XX:+AlwaysPreTouch` | 启动时预分配内存，避免运行时向 OS 申请内存造成延迟 |
| `-XX:+PerfDisableSharedMem` | 禁用 perf 共享内存，防止 Java 进程意外挂起 |
| `-XX:MaxTenuringThreshold=1` | 对象晋升阈值，减少长期存活对象的维护开销 |

### 自动重启脚本

```bash
#!/bin/bash
# restart-loop.sh — 崩溃自动重启

while true; do
    echo "[$(date)] 启动 MC 服务器..."
    java -Xms4G -Xmx8G -XX:+UseG1GC -jar server.jar nogui
    echo "[$(date)] 服务器停止，10 秒后重启..."
    sleep 10
done
```

### Systemd 服务（生产环境推荐）

```ini
[Unit]
Description=Minecraft Server
After=network.target

[Service]
User=mcserver
WorkingDirectory=/home/mcserver/server
ExecStart=/usr/bin/java -Xms4G -Xmx8G -XX:+UseG1GC -jar /home/mcserver/server/server.jar nogui
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

保存为 `/etc/systemd/system/minecraft.service`，然后：

```bash
sudo systemctl daemon-reload
sudo systemctl enable minecraft
sudo systemctl start minecraft
sudo systemctl status minecraft   # 查看状态
```

---

## 四、服务器配置（server.properties）

MC 服务器的核心配置文件 `server.properties` 包含最重要的运行时设置：

```properties
# 基础设置
server-port=25565               # 服务器端口
server-ip=                      # 绑定 IP（留空 = 所有网卡）
motd=A Minecraft Server         # 服务器列表显示的信息
max-players=20                  # 最大玩家数
difficulty=easy                 # 难度: peaceful/easy/normal/hard
gamemode=survival               # 默认游戏模式: survival/creative/adventure/spectator

# 世界设置
level-type=default              # 世界类型: default/flat/largebiomes/amplified
seed=                           # 世界种子（留空 = 随机）
spawn-protection=16             # 出生点保护半径
view-distance=10                # 视距（区块数），越小性能越好
simulation-distance=8           # 模拟距离，影响实体加载

# PvP 与玩家交互
pvp=true                        # 是否开启 PvP
white-list=false                # 是否开启白名单
enforce-whitelist=false         # 启动时是否强制执行白名单

# 网络与性能
network-compression-threshold=256  # 网络压缩阈值
max-tick-time=-1                # 最大 tick 时间（-1 = 禁用看门狗）
enable-query=false              # 是否启用 GameSpy 查询协议
```

> 🔧 **常用修改**：`view-distance` 和 `simulation-distance` 是性能影响最大的两个参数，每降低 1 个值都能显著降低 CPU 负载。

---

## 五、网络与安全

### 端口转发（家庭自建服）

1. 在路由器管理后台找到 **端口转发 / 虚拟服务器** 功能
2. 添加规则：
   - 外部端口：`25565`
   - 内部 IP：你的服务器内网 IP（如 `192.168.1.100`）
   - 内部端口：`25565`
   - 协议：`TCP/UDP`
3. 使用 [WhatIsMyIP](https://whatismyip.com) 获取公网 IP
4. 玩家通过 `你的公网IP:25565` 连接

> ⚠️ **注意**：家庭宽带通常没有固定公网 IP，需要 DDNS（动态 DNS）或使用 FRP/Ngrok 等内网穿透工具。国内 ISP 可能封禁 25565 端口，可改为其他端口（如 25566）。

### SRV 记录（自定义域名）

在 DNS 解析中添加 SRV 记录，让玩家通过域名连接：

```
类型: SRV
名称: _minecraft._tcp.yourdomain.com
目标: yourdomain.com
端口: 25565
优先级: 0
权重: 5
```

### 反向代理与安全

```bash
# 使用 TCPShield 保护服务器（免费层可用）
# 或使用 Cloudflare Proxy（仅限域名，需关闭橙色云朵或使用 Tunnel）

# 对于大服，推荐前置代理架构：
# 玩家 -> TCPShield/Cloudflare -> Velocity/BungeeCord -> 后端服务器
```

---

## 六、维护与管理

### 备份策略

```bash
#!/bin/bash
# backup.sh — 自动备份脚本

# 在服务端执行 /save-off 和 /save-all 后运行
# 或直接使用 screen 发送命令

screen -S minecraft -X stuff "say 服务器正在备份，可能短暂卡顿...\n"
screen -S minecraft -X stuff "save-all\n"
sleep 5
screen -S minecraft -X stuff "save-off\n"

# 备份世界数据
tar -czf "./backups/world_$(date +%Y%m%d_%H%M%S).tar.gz" world world_nether world_the_end

screen -S minecraft -X stuff "save-on\n"
screen -S minecraft -X stuff "say 备份完成！\n"

# 删除 7 天前的旧备份
find ./backups -name "*.tar.gz" -mtime +7 -delete
```

### 推荐定时备份（crontab）

```cron
# 每天凌晨 4 点备份
0 4 * * * /home/mcserver/backup.sh
```

### 性能监控

```bash
# 基础监控
htop                              # 查看 CPU 和内存
df -h                             # 查看磁盘空间
du -sh ./world                    # 查看世界文件大小

# MC 特定监控（在控制台输入）
/tps                              # 查看服务器 TPS
/mspt                             # 查看每 tick 耗时
/paper timings                    # Paper 性能分析
```

### 更新流程

1. **备份世界数据**（永远第一步）
2. 下载新版服务端 jar
3. 测试新版本的核心配置（新版 Paper 通常可以直接替换）
4. 依次更新插件/Mod（注意兼容性）
5. 启动并测试

---

## 七、插件与 Mod 推荐

### 必备插件（Paper/Spigot）

| 插件 | 用途 |
|------|------|
| **LuckPerms** | 权限管理，标准选择 |
| **EssentialsX** | 基础命令套装（home、tpa、warp 等） |
| **CoreProtect** | 方块日志与回滚，反熊必备 |
| **Vault** | 经济系统接口 |
| **PlaceholderAPI** | 变量占位符，很多插件的前置 |
| **BlueMap / Dynmap** | 网页地图，方便玩家浏览世界 |

### 性能优化

| 插件/工具 | 用途 |
|-----------|------|
| **Chunky** | 区块预生成，避免新玩家进入时卡顿 |
| **ClearLag** | 定时清理掉落物 |
| **Spark** | 性能分析，定位卡顿原因 |

### 优化 Mod（Fabric）

| Mod | 用途 |
|-----|------|
| **Lithium** | CPU 优化，无副作用 |
| **Phosphorus** | 光照引擎优化 |
| **Starlight** | 光照计算重写，大幅提升性能 |
| **FerriteCore** | 内存优化 |
| **Krypton** | 网络优化 |

---

## 八、常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|---------|---------|
| 玩家连接超时 | 端口未转发 / 防火墙未放行 | 检查路由器设置和 iptables |
| 服务器卡顿 / TPS 低 | CPU 瓶颈 / 内存不足 / GC 频繁 | 减少视距，增加内存，检查 GC 日志 |
| 存档损坏 | 非正常关闭 / 磁盘错误 | 定期备份，使用 `/save-all` |
| Mod 冲突 | Mod 版本不兼容 | 检查 Crash Report，逐个排除 |
| 内存溢出 (OOM) | 堆太小 / 内存泄漏 | 增加 -Xmx 值，检查插件内存占用 |

---

## 九、进阶方向

- **跨服网络**：使用 Velocity 搭建多世界/多游戏模式的跨服网络
- **Docker 化部署**：将服务器放在 Docker 容器中，方便迁移
- **Git 化配置管理**：用 Git 管理所有配置文件，记录变更历史
- **CI/CD 自动部署**：更新时自动拉取、测试、部署
- **分布式服务器**：配合 Redis 实现多台物理机共同承载一个服务器

---

> 本文持续更新中。Minecraft 的版本迭代很快，建议定期查阅 [PaperMC Docs](https://docs.papermc.io)、[Minecraft Wiki](https://minecraft.wiki) 和对应的服务端官方文档获取最新信息。
