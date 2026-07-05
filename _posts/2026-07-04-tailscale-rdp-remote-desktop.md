---
layout: post
title: "Tailscale + RDP：Ubuntu 远程桌面的最佳方案"
date: 2026-07-04 15:00:00 +0800
categories: Linux 教程
---

## 为什么需要这个方案

Ubuntu 26 默认使用 Wayland 显示协议，导致很多传统远程桌面方案失效：

- **VNC**：Wayland 下权限受限，画面卡顿
- **X11 forwarding**：需要切换回 X11，体验差

而 Ubuntu 自带的 RDP 服务 + Tailscale P2P 组合，完美解决这些问题：

- **快捷**：5 分钟搞定
- **安全**：Tailscale 端到端加密，RDP 走内网
- **免费**：Tailscale 个人版免费，RDP 是系统自带

## 什么是 Tailscale

Tailscale 基于 WireGuard 协议，帮你组建一个私有虚拟网络。所有设备通过 Tailscale 分配的 IP 直连，不经过第三方服务器。

<!-- ![Tailscale 工作原理图](/assets/images/tailscale-architecture.png) -->

## 第一步：安装 Tailscale

### Ubuntu 端

打开终端，执行：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

安装完成后启动：

```bash
sudo tailscale up
```

终端会输出一个登录链接，复制到浏览器打开，用 Google/Microsoft/GitHub 账号登录即可。

<!-- ![Ubuntu 终端执行 tailscale up](/assets/images/ubuntu-tailscale-up.png) -->

登录成功后，查看分配的 IP：

```bash
tailscale ip -4
```

记下这个 IP（例如 `100.x.x.x`），后面手机连接要用。

<!-- ![tailscale ip 输出结果](/assets/images/ubuntu-tailscale-ip.png) -->

### 手机端

在应用商店搜索 Tailscale 安装：

- **Android**：[Google Play](https://play.google.com/store/apps/details?id=com.tailscale.ipn) / [国内下载](https://tailscale.com/download/android)

打开 App，用同一个账号登录。登录后手机会自动获得一个 `100.x.x.x` 的 IP。

![手机端 Tailscale 登录界面](/assets/images/tailscale-login.jpg)

在 Ubuntu 终端确认手机已加入网络：

```bash
tailscale status
```

<!-- ![tailscale status 显示两台设备](/assets/images/tailscale-status.png) -->

## 第二步：启用 Ubuntu RDP 服务

Ubuntu 26 自带 GNOME 远程桌面，支持 RDP 协议。

打开 **设置 → 系统 → 远程桌面**（或搜索 "Remote Desktop"）：

1. 打开 **远程桌面** 开关
2. 打开 **远程控制** 开关
3. 设置用户名和密码（手机连接时要用）

![Ubuntu 远程桌面设置界面](/assets/images/ubuntu-rdp-settings.png)

> **注意**：如果找不到该选项，可能需要安装：
> ```bash
> sudo apt install gnome-remote-desktop
> ```

## 第三步：手机连接

推荐使用 [aRDP Free](https://play.google.com/store/apps/details?id=com.iiordanov.ardp)（免费）。

> 如果 Google Play 无法下载，本仓库提供了安装包（`aRDP Free_v6.4.4(1).apks`），使用 MT 管理器安装即可。

1. 打开 App，点击 **新建连接**
2. **主机** 填 Ubuntu 的 Tailscale IP（`100.x.x.x`）
3. **用户名** 和 **密码** 填刚才设置的
4. 保存后点击连接

![aRDP 连接配置](/assets/images/ardp-config.jpg)

## 连接成功

![手机远程桌面连接成功](/assets/images/remote-desktop-success.jpg)

现在你可以在手机上操控 Ubuntu 桌面了：

- 触屏模拟鼠标操作
- 支持手势缩放
- 延迟极低（局域网级体验）

## 常见问题

### 连不上？

检查两端 Tailscale 是否都在线：

```bash
# Ubuntu 端
tailscale status

# 手机端打开 Tailscale App 查看状态
```

### 画面卡顿？

在 Ubuntu 的远程桌面设置中降低分辨率，或在 RDP 客户端里调整画质。

### Tailscale 免费够用吗？

个人版完全免费，支持最多 100 台设备，对个人使用绰绰有余。

## 总结

| 方案 | Wayland 兼容 | 安全性 | 费用 | 难度 |
|------|-------------|--------|------|------|
| VNC + 内网穿透 | 部分 | 低 | 免费 | 高 |
| X11 forwarding | 需切 X11 | 高 | 免费 | 中 |
| **Tailscale + RDP** | **完美** | **高** | **免费** | **低** |

推荐指数：⭐⭐⭐⭐⭐
