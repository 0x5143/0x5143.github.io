--- 
layout: post
title: "免服务器实现内网穿透：使用 Cloudflare Tunnel 远程访问家里 WSL 服务"
comments: true
categories:
 - 网络技术
 - 内网穿透
tags:
 - Cloudflare
 - WSL
 - 内网穿透
 - Hermes
 - 生产力
description: "介绍如何利用 Cloudflare Tunnel (Zero Trust) 实现免服务器的内网穿透，完美解决公司网络限制且无需安装客户端的远程访问需求。"  
---

本教程介绍如何利用 **Cloudflare Tunnel (Zero Trust)** 实现内网穿透，在公司（或其他外部网络）安全地访问部署在家里 Windows WSL 中的 **Hermes Agent Web UI**。

### 🌟 方案优势

*   **完全免费**：无需购买阿里云等公网 ECS 服务器。
*   **公司端免安装**：公司电脑禁止安装非合规软件？此方案在公司端**只需浏览器**，无需安装任何客户端。
*   **高安全性**：流量全程加密，不暴露家里公网 IP，且支持利用 Cloudflare Access 限制仅允许公司 IP 访问。

* * *

### 🛠️ 准备工作

1.  **Cloudflare 账号**：前往 [Cloudflare 官网](https://dash.cloudflare.com/) 免费注册。
2.  **独立域名**：在阿里云等平台购买一个便宜的域名（一年的费用通常仅需十几元）。
3.  **域名托管**：将该域名的 DNS 服务器（NS 记录）修改并托管至 Cloudflare（Cloudflare 注册后会有保姆级修改引导）。

* * *

### 🚀 第一步：在 Cloudflare 后台创建隧道

1.  登录 Cloudflare 控制台，点击左侧菜单的 **Zero Trust** 进入安全后台。
2.  导航至左侧菜单：**Networks** -> **Tunnels**。
3.  点击 **Create a tunnel** 按钮，选择 **Cloudflared**（默认），点击 **Next**。
4.  为你的隧道起一个辨识度高的名字（例如：`home-wsl-hermes`），点击 **Save tunnel**。

* * *

### 🏠 第二步：在家里 WSL 中安装并运行客户端

完成创建后，网页会跳转到 **Choose an environment**（选择环境）页面。

1.  在环境中选择 **Linux** -> **AMD64**（绝大多数 Windows 电脑的 CPU 架构）。
2.  页面下方会自动生成一段专属于你账号的安装与启动命令，类似如下格式：
    
    bash
    
        curl -L --output cloudflared.deb https://github.com && \
        sudo dpkg -i cloudflared.deb && \
        sudo cloudflared service install eyJhIjoiM...（此处为很长的 Token 密钥）
        
    
    请谨慎使用此类代码。
    
3.  打开家里的 **WSL 终端**，完整粘贴并运行这段命令。
4.  **状态检查**：运行成功后，回到 Cloudflare 网页端。当看到当前 Tunnel 的 Status 变成绿色的 **HEALTHY** 时，说明本地 WSL 已成功与 Cloudflare 建立加密通道。

* * *

### 🌐 第三步：配置域名映射 (Public Hostname)

现在需要将你的公网域名指向家里 WSL 内部的 Hermes 端口。

1.  在 Tunnel 页面点击 **Next**（或点击该隧道右侧的 **Edit**），进入 **Public Hostname** 标签页。
2.  点击 **Add a public hostname**，并填写以下信息：
    *   **Subdomain（子域名）**：输入你想访问的前缀，例如 `hermes`。
    *   **Domain（主域名）**：下拉选择你在 Cloudflare 托管的域名（如 `yourdomain.com`）。
    *   **Type**：选择 **HTTP**。
    *   **URL**：输入 `127.0.0.1:8080`（请根据你 WSL 内 Hermes Agent 实际的 Web UI 端口进行修改）。
3.  点击右下角的 **Save hostname** 保存。

* * *

### 💻 第四步：在公司直接访问

在公司电脑上打开任意浏览器，直接输入刚才配置的完整域名：

text

    http://yourdomain.com
    

请谨慎使用此类代码。

此时你就能直接看到家里 WSL 里的 Hermes Agent 登录界面，且不需要在公司电脑上安装任何特殊网络软件。

* * *

### 🔒 进阶安全防护：配置公司 IP 白名单（强烈建议）

由于域名暴露在公网，为了防止恶意扫描，强烈建议利用 Cloudflare 强大的防火墙拦截外部网络，**仅允许公司网络访问**。

1.  在 Cloudflare Zero Trust 后台，点击左侧的 **Access** -> **Applications**。
2.  点击 **Add an application** -> 选择 **Self-hosted**。
3.  **Application configuration** 配置：
    *   **Application name**：输入 `Hermes-Protect`。
    *   **Application domain**：填写你刚才设置的子域名 `hermes` 和主域名 `yourdomain.com`。
4.  点击 Next 进入 **Rules (策略)** 配置：
    *   **Rule name**：输入 `Allow-Company-IP`。
    *   **Rule action**：选择 **Bypass**（直接放行）或 **Allow**。
    *   **Assign a group** / **Include**：
        *   **Selector** 选择 **IP Ranges**。
        *   **Value** 填入你**公司网络的公网出口 IP/网段**。
5.  一路点击 Next 并保存。

> **效果**：配置完成后，只有当你坐在公司办公室、使用公司网络时才能打开该域名。在家里用手机流量或者外界黑客访问该域名，都会被 Cloudflare 直接拒绝（返回 Blocked 提示），安全性达到企业级。
