---
title: 漫谈校园网破解
published: 2025-12-17
description: '简单聊聊校园网多设备检测，对比 UA2F、UA3F、UA-Mask 三种方案，提供部分的 OpenWRT 配置教程和硬件选购建议。'
image: ''
tags: [博文, OpenWRT]
category: 技术
draft: false
---

## 前言

校园网多设备检测，估计是每个学生党都遇到过的问题。明明交了网费，结果手机、电脑、平板一起用就被限速，甚至直接断网。这篇文章就来聊聊校园网是怎么检测多设备的，以及怎么在 OpenWRT 路由器上绕过这些检测。

## 校园网检测原理

### 常见检测厂商

国内校园网检测基本这几家：锐捷、深信服、石斧、天翼、Panabit、武汉雨滴。虽然厂商不同，但检测手段其实都差不多。

### 检测手段分析

#### 1. HTTP User-Agent 检测

最简单粗暴的方法。每个设备的浏览器都会在 HTTP 请求里带上自己的"身份证"（User-Agent）：

```
# Windows 电脑
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36

# Android 手机
Mozilla/5.0 (Linux; Android 13; SM-G998B) AppleWebKit/537.36

# iPhone
Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X)
```

校园网一看，同一个 IP 怎么一会儿是 Windows，一会儿是 Android？肯定是多设备共享，直接封了。

**验证工具：**

想看看自己的 UA 是什么，可以访问这两个网站：
- [http://ua-check.stagoh.com/](http://ua-check.stagoh.com/)
- [http://ua.233996.xyz/](http://ua.233996.xyz/)

配置完成后也可以用这些网站验证效果。

#### 2. TTL 值检测

TTL（Time To Live）可以理解为数据包的"寿命"。不同系统的默认值不一样：

- Windows: 128
- Linux/Android/iOS: 64

关键在于，数据包每经过一个路由器，TTL 就会减 1。所以如果你用路由器共享网络，校园网收到的数据包 TTL 就会不一样：电脑直连是 128，但经过路由器的手机就变成 63 了。这一对比，马上就露馅了。

#### 3. DPI 深度包检测

DPI（Deep Packet Inspection）就是深度包检测，比前面两种高级多了。它会扒开数据包看里面的内容：
- HTTP 请求头的各种细节
- TLS/SSL 握手时的指纹特征
- 应用层协议的特征
- 数据包大小和发送时序

这种方法准确率高，但也吃性能，一般只有比较严格的校园网才会用。

#### 4. IPv6 检测

有些校园网开始玩 IPv6 了。因为 IPv6 地址多到用不完，每个设备都能分到独立的公网地址。这样一来，检测多设备就更简单了——看看有多少个不同的 IPv6 地址在用就行。

## 技术方案对比

知道了检测原理，接下来就是怎么绕过了。社区里有几个比较成熟的方案：

### UA2F - User-Agent 早期方案

::github{repo="Zxilly/UA2F"}

**原理：** 用 iptables 的 NFQUEUE 机制拦截 HTTP 请求，把所有设备的 User-Agent 都改成一样的。这样校园网看到的就都是同一个"身份证"了。

```mermaid
graph LR
    A[客户端设备] -->|HTTP 请求| B[UA2F]
    B -->|修改 User-Agent| C[iptables NFQUEUE]
    C -->|统一 UA| D[校园网网关]
    D --> E[互联网]
```

**优点：**
- 简单好用，不怎么吃资源
- HTTP 流量处理没问题

**缺点：**
- 只能改 HTTP，HTTPS 没辙（现在网站基本都是 HTTPS 了）
- 应付不了 DPI 深度检测
- 跟硬路由的硬件加速冲突（比如 MT7621 的 HW NAT、高通的 NSS）
- 跟代理软件一起用容易出问题

### UA3F - User-Agent 过滤（目前最强防检测）

::github{repo="SunBK201/UA3F"}

**原理：** UA2F 的升级版，能处理 HTTPS 流量了。通过分析 TLS 握手时的 SNI 和指纹信息来伪装流量。还自带 TTL 统一功能，在"其他设置"里一键开启就行。支持 iptables 和 nftables。

```mermaid
graph LR
    A[客户端设备] -->|HTTP/HTTPS 请求| B[UA3F]
    B -->|修改 User-Agent| C[TLS 指纹处理]
    C -->|统一 TTL 可选| D[iptables/nftables]
    D -->|伪装流量| E[校园网网关]
    E --> F[互联网]
```

**优点：**
- **目前最强的伪装工具**，对抗能力拉满
- HTTPS 也能处理
- 能改 TLS 握手指纹
- 自带 TTL 统一，不用额外配置
- iptables 和 nftables 都支持

**缺点：**
- **比较吃性能**，CPU 占用高
- 配置相对复杂
- 可能影响某些网站访问
- 低性能路由器跑不动

#### UA3F 使用指南

**核心特性：**

UA3F 功能挺全的，基本该有的都有了：
- 多层级服务模式（应用层、传输层、网络层都能玩）
- 灵活的规则系统，想改啥改啥
- 实时监控面板，能看到流量统计
- TTL、TCP Timestamp、IPID 伪装都支持
- 还有 TCP Desync 分片乱序发射这种高级货

**安装方法：**

1. **IPK 包安装**（推荐）：
   ```bash
   # 从 GitHub Releases 下载对应架构的 ipk 文件
   opkg update && opkg install ua3f_*.ipk
   ```

2. **OpenWrt 编译**：
   ```bash
   # 克隆到 OpenWrt 源码目录
   cd openwrt/package
   git clone https://github.com/SunBK201/UA3F.git

   # 编译
   make package/UA3F/compile
   ```

3. **Docker 部署**：
   ```bash
   docker run -p 1080:1080 sunbk201/ua3f -f FFF
   ```

**配置说明：**

UA3F 有三种重写策略：

- **GLOBAL**：全局重写，所有请求都改
- **DIRECT**：直接转发，啥也不改
- **RULES**：按规则来，想改哪个改哪个（推荐这个）

**服务模式选择：**

- **TPROXY/REDIRECT**：性能好兼容性也好，优先选这个
- **HTTP/SOCKS5**：兼容性最好，但性能一般，有特殊需求时用
- **NFQUEUE**：性能最强，追求极致性能时用

**快速配置（LuCI 界面）：**

1. 打开 Web 界面：**网络 → UA3F**
2. 启用 UA3F 服务
3. 选择重写策略（推荐 **RULES** 规则重写）
4. 在"其他设置"中启用 TTL 修改功能
5. 保存并应用设置

**与代理工具共存：**

如果你还用了 OpenClash/PassWall 这些代理：
- **推荐用 TPROXY 模式**（兼容性最好）
- NFQUEUE 可能跟某些代理冲突
- 具体配置看官方文档
- 记得让 UA3F 先处理流量，再交给代理

> 详细教程请参考：[UA3F 官方文档](https://sunbk201public.notion.site/UA3F-2a21f32cbb4b80669e04ec1f053d0333)

### UA-Mask（性能优先）

::github{repo="Zesuy/UA-Mask"}

**原理：** 把 UA 修改和 TTL 统一结合起来，双管齐下。用 iptables/nftables 把所有数据包的 TTL 改成一样的，再配合 UA 修改。iptables 和 nftables 都支持。

```mermaid
graph LR
    A[客户端设备] -->|多种流量| B[UA-Mask]
    B -->|修改 UA| C[TTL 统一模块]
    C -->|设置 TTL=64| D[iptables/nftables]
    D -->|双重伪装| E[校园网网关]
    E --> F[互联网]
```

**优点：**
- **性能好**，不怎么吃资源
- UA 和 TTL 两个维度都能处理
- 配置简单
- 兼容性不错
- iptables 和 nftables 都能用
- 低性能路由器也能跑

**缺点：**
- **防检测能力一般**，应付不了高强度 DPI
- 改不了 TLS 握手指纹
- TTL 值要配对才行

#### UA-Mask 使用指南

**安装方法：**

1. **预编译包安装**（推荐）：
   ```bash
   # 从 GitHub Releases 下载对应架构的 ipk 文件
   opkg update && opkg install UAmask_*.ipk

   # iptables 用户若需流量卸载功能，需先安装 ipset
   opkg install ipset
   ```

2. **源码编译**：
   ```bash
   # 克隆到 OpenWrt 源码的 package/luci 目录
   cd openwrt/package/luci
   git clone https://github.com/Zesuy/UA-Mask.git

   # 编译生成 ipk
   make package/UA-Mask/compile
   ```

**快速配置**（5分钟搞定）：

1. 打开路由器管理页面：**系统 → 服务 → UA-Mask**，勾上"启用"
2. **常规设置**：性能预设选 Medium 就行，填个通用的浏览器 UA
3. **网络与防火墙**：默认端口 12032 不用改，监听接口选 br-lan
4. 保存应用
5. 打开 `ua.233996.xyz` 看看效果

**使用建议：**

- **追求性能**：开启流量卸载，用关键词匹配（别用复杂正则）
- **追求安全**：关掉流量卸载，把 443 端口从绕过列表里删了
- **跟 OpenClash 一起用**：让 UA-Mask 先处理，再交给 Clash 分流

**注意事项：**

- HTTPS 看不到效果很正常，UA 只在 HTTP 明文里生效
- 被加到卸载 set 的 `ip:port` 会完全绕过 UA-Mask，小心 UA 泄露
- 用关键词模式就行，别整复杂正则，省 CPU
- 流量卸载别乱配，容易误判

> 详细教程请参考：[UA-Mask 官方文档](https://github.com/Zesuy/UA-Mask/blob/main/docs/tutorial.md)

## 实践建议

### TTL 统一方案

#### 方案一：用 UA3F 内置功能（推荐）

UA3F 自带 TTL 统一，最省事：

1. 装好 UA3F，打开路由器管理页面
2. 找到 UA3F 的"其他设置"
3. 把"TTL 修改"打开
4. 保存应用就行

#### 方案二：手动配置防火墙

如果用其他方案，或者想自己配置，可以手动改防火墙规则：

**使用 iptables：**

```bash
# 统一 IPv4 TTL 为 64
iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64

# 统一 IPv6 Hop Limit 为 64
ip6tables -t mangle -A POSTROUTING -j HL --hl-set 64
```

**使用 nftables（推荐）：**

```bash
# 创建 mangle 表和链
nft add table inet mangle
nft add chain inet mangle postrouting { type filter hook postrouting priority mangle \; }

# 统一 IPv4 TTL 为 64
nft add rule inet mangle postrouting ip ttl set 64

# 统一 IPv6 Hop Limit 为 64
nft add rule inet mangle postrouting ip6 hoplimit set 64
```

这样所有数据包的 TTL 都一样了，校园网就看不出来了。

> **注意：** nftables 是新一代防火墙，比 iptables 性能好，语法也简洁。OpenWRT 21.02 以上默认用 nftables。

### 选择合适的硬件方案

根据你的路由器和校园网检测强度，选合适的方案：

1. **检测不严格**：
   - 只改 TTL 就够了
   - 或者用 UA-Mask（性能好，不吃资源）

2. **检测一般**：
   - 路由器性能好：用 **UA3F**（最强）
   - 路由器性能差：用 **UA-Mask**（省资源）

3. **检测很严格**：
   - 必须上 **UA3F**（对抗能力最强）
   - 如果还是不行，试试 UA3F + 代理

**性能参考与硬件建议：**

**MT7621 平台用户**（比如小米 R3G、AC2100）：
- 建议用 **UA-Mask** 并开启流量卸载
- UA3F 在这平台上跑不太动，网速不太行

**路由器升级建议**：

如果预算够的话，换个好点的路由器：

1. **入门推荐**（咸鱼上很多运营商路由器）：
   - **MT7981 芯片**：双核 ARM A53，性能提升明显
   - **IPQ6000 芯片**：四核 ARM A53，性能更强
   - 价格不贵，性价比高，跑 UA3F 没问题

2. **内存建议**：
   - 只用 UA3F/UA-Mask：512MB 够了
   - 如果有跑代理（OpenClash/PassWall）：**起码需要1GB**
   - 多个服务一起跑：1GB 起步

**芯片性能对比：**
- MT7621（双核 MIPS 880MHz）：只能跑 UA-Mask/UA2F/UA3F勉强
- MT7981（双核 A53 1.3GHz）：跑 UA3F 还行
- IPQ6000（四核 A53 1.2GHz）：性能充足，个人推荐首选

## 参考资源

::github{repo="Zxilly/UA2F"}

::github{repo="SunBK201/UA3F"}

::github{repo="Zesuy/UA-Mask"}
