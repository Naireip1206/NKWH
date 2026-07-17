---
标题: MC服务器搭建指南
标签: [专题, Minecraft, 服务器, 运维]
创建日期: 2026-07-17
来源数量: 1
相关实体: []
相关概念: [知识库/概念/MC服务器, 知识库/概念/服务器性能优化]
类型: 专题
状态: 活跃
---

# MC 服务器搭建指南

> 本文是 MC 服务器搭建的综合专题研究，梳理了从零开始搭建一个可用的 Minecraft 服务器的完整知识体系。内容涵盖硬件选型、环境部署、性能优化、网络配置和日常运维。

---

## 一、搭建决策树

搭建 MC 服务器需要根据目标场景来做决策，以下是快速决策流程：

```
你想开什么样的服？
├── 📦 原版 + 插件生存服 → Paper / Purpur
│   ├── 1-10 人 → 低配物理机 / 轻量云服务器（4C8G）
│   ├── 10-30 人 → 中配物理机 / 云服务器（4C16G）
│   └── 30-100+ 人 → 高性能专用机（6C+ 高频、32G+）
├── 🔧 轻量 Mod 服 → Fabric
│   ├── 少量 Mod → 中等配置（4C12G）
│   └── 大量 Mod → 高配置（8C+、16G+）
├── ⚙️ 大型整合包 → Forge / NeoForge
│   └── ATM、GTNH 等 → 需要 8C+ 高频、16-32G 内存
├── 🌐 跨服网络 → Velocity 代理 + 多后端
│   └── 按子服独立扩缩容
└── 🤖 纯测试 / 局域网 → Vanilla / 任意核心
```

---

## 二、硬件选型详解

### 核心公式

```
可承载玩家数 ∝ CPU单核性能 × 内存容量 × 优化系数 / 玩家行为复杂度
```

其中优化系数（按照经验值）：
- Vanilla = 1.0
- Paper = ~1.8（相同硬件可承载约 1.8 倍于 Vanilla 的玩家）
- Paper + 预生成 + JVM 优化 = ~2.5

### 成本效益分析

| 方案 | 月成本（估算） | 适合场景 | 维护难度 |
|------|--------------|---------|---------|
| 物理机自建 | 一次性 ~¥2000+ | 长期拥有，高自由度 | ⭐⭐⭐ |
| 轻量云服务器 | ¥50-150/月 | 小型生存服 | ⭐⭐ |
| ECS/CVM 按量 | ¥200-500/月 | 中型服，可弹性扩缩 | ⭐⭐ |
| 高性能云主机 | ¥500-2000+/月 | 大型公开服 | ⭐⭐ |
| 托管服（Minecraft Hosting） | ¥50-300/月 | 不想折腾 | ⭐ |

### 网络要求

| 玩家人数 | 上行带宽需求 |
|---------|------------|
| 1-10 人 | 5-10 Mbps |
| 10-30 人 | 10-30 Mbps |
| 30-50 人 | 30-50 Mbps |
| 50-100 人 | 50-100 Mbps |

---

## 三、环境部署速查

### Java 安装对比

| 发行版 | 包名 | 适用 |
|-------|------|------|
| OpenJDK | `openjdk-21-jre-headless` | Ubuntu/Debian 官方源 |
| Eclipse Temurin | `temurin-21-jdk` | 跨平台一致性高 |
| Amazon Corretto | `java-21-amazon-corretto` | AWS 环境集成好 |
| GraalVM | — | 实验性优化，不推荐生产 |

### 服务端核心获取

```bash
# Paper（推荐）
wget https://api.papermc.io/v2/projects/paper/versions/1.21.1/builds/130/downloads/paper-1.21.1-130.jar -O server.jar

# Fabric
wget https://meta.fabricmc.net/v2/versions/loader/1.21.1/0.15.11/0.11.2/server/jar -O server.jar

# Forge（需先运行安装器）
wget https://maven.minecraftforge.net/net/minecraftforge/forge/1.21.1-52.0.37/forge-1.21.1-52.0.37-installer.jar
java -jar forge-1.21.1-52.0.37-installer.jar --installServer
```

---

## 四、配置优化列表

### 必做清单

- [ ] 选择合适的服务端核心（推荐 Paper）
- [ ] 使用 Aikar's Flags 启动参数
- [ ] `view-distance` 设置为 6-8
- [ ] `simulation-distance` 设置为 5-6
- [ ] 预生成 5000-10000 区块半径的世界
- [ ] 配置定时自动备份
- [ ] 配置防火墙（仅开放必要端口）
- [ ] 设置白名单或正版验证

### 推荐做

- [ ] 安装性能分析工具（Spark）
- [ ] 配置每日重启（降低内存泄漏影响）
- [ ] 限制单个玩家可生成的实体数量
- [ ] 使用 ChunkyBorder 限制世界边界
- [ ] 设置红石 tick 速率限制
- [ ] 配置 Systemd 实现开机自启

### 进阶

- [ ] Docker 容器化 + docker-compose
- [ ] Git 管理配置文件
- [ ] CI/CD 自动部署
- [ ] Prometheus + Grafana 监控
- [ ] 地理 DNS 分流
- [ ] 多线 BGP 接入

---

## 五、常见架构模式

### 单服务器（单体）
```
玩家 → 公网IP:端口 → MC服务端
```
最简单的架构，适合小型服。

### 带代理的跨服
```
玩家 → TCPShield(防护) → Velocity(代理)
                              ├── 生存服 (hub1)
                              ├── 创造服 (hub2)
                              ├── 小游戏服 (hub3)
                              └── 后台管理服 (隐藏)
```
适合中型+服务器，各子服可独立重启。

### Docker 化部署
```
docker-compose.yml
├── velocity (代理)
├── lobby (大厅)
├── survival (生存)
├── creative (创造)
├── db (数据库 - LuckPerms/经济)
└── backup (定时备份容器)
```
适合需要快速部署和迁移的场景。

---

## 六、参考资源

- [PaperMC Docs](https://docs.papermc.io) — Paper 官方文档，包含 Aikar's Flags 和优化指南
- [Minecraft Wiki](https://minecraft.wiki) — MC 官方 Wiki
- [Spark 性能分析](https://spark.lucko.me) — Spark 使用文档
- [Aikar's Flags 原帖](https://mcflags.emc.gs) — JVM 优化参数的原始讨论
- [PaperMC Discord](https://discord.gg/papermc) — Paper 社区
- [Minecraft 服务端软件列表](https://minecraft.wiki/w/Server_software) — 各家核心对比

---

## 相关页面

- [[知识库/概念/MC服务器]] — MC 服务器基本概念
- [[知识库/概念/服务器性能优化]] — 性能优化方法
- [[知识库/来源/MC服务器搭建思路摘要]] — 来源摘要
- [[MC服务器搭建的整体思路]] — 原始搭建指南
