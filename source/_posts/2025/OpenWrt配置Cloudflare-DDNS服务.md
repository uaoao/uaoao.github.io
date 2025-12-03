---
title: OpenWrt配置Cloudflare-DDNS服务
date: 2025-5-31
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-Cloudflare
  - 主题-OpenWrt
  - 主题-Linux
  - 主题-DDNS
---

一般来说，如果需要对外开放HTTP/HTTPS服务，最合适的方式是[配置Cloudflare隧道](https://uaoao.github.io/2025/5/27/OpenWrt%E9%85%8D%E7%BD%AECloudflare%E9%9A%A7%E9%81%93.html)。因为CF提供的隧道可以提供证书加密且配置简单。但隧道不适合所有场合，比如说使用IPv6 SSH时就不必再套一层CF，而且直连延迟低、丢包率低、带宽取决于双方的上下行带宽。

我家最初还有动态公网IPv4时，一直使用的是DDNS服务，SSH直连到家。后来动态公网IPv4被回收，只能用第三方内网穿透服务（国内IP）。最近这个服务到期需要再次缴费，我决定抛弃这个服务，转而使用纯IPv6连接或CF隧道穿透。

## 准备

- Cloudflare 账号一个，绑定了域名且托管DNS解析，比如 `example.com`
- OpenWrt 可以安装软件包
- 手机或其他与Openwrt不在同一个局域网内的电脑

## 获取 Cloudflare API Token

1. 登陆 [Cloudflare](https://dash.cloudflare.com)，打开 [用户API令牌](https://dash.cloudflare.com/profile/api-tokens)
2. 点击Create Token。不建议使用API keys
3. 点击 Edit zone DNS 右侧的 Use template
4. 编辑令牌名称（可选）为 `example.com DDNS`
5. Permissions栏使用默认设置：Zone、DNS、Edit
6. Zone Resources栏设置为：Include、Specific zone、`example.com`
7. 点击最下面的 Continue to summary 保存，然后点击 Create Token
8. 复制API Token到记事本中，因为只会显示一次。
9. 可以尝试执行测试命令看看能否正常访问，如果响应中包含`"success":true`，则可正常使用

## 编辑 Cloudflare DNS 记录

1. 进入域名的 DNS Records 编辑页面，添加一条IPv6记录
2. 类型为AAAA，名称使用 `openwrt`等不冲突的名称，IPv6地址随机填比如 `2000::1` 或其他符合格式的地址，点击禁用代理模式，然后保存

## 配置 OpenWrt 启用 DDNS 服务

### 安装 DDNS 服务软件

1. 安装 `luci-app-ddns` `ddns-scripts-cloudflare` 和 `bind-host`（这个可以用其他软件包替换）
2. 注销重新登陆 OpenWrt，进入 Services —— Dynamic DNS，确保 State 是 **DDNS Autostart enabled**
3. 点击 Update DDns Services List
4. 删除原有的两个模板，点击 Add new services
5. 名称可以填 `ddns` 或其他，IP地址版本选 IPv6，DDNS服务供应商选择 `cloudflare.com-v4`，点击创建服务

> 注意：如果 State 是 **DDNS Autostart disabled**，可以在 System —— Startup 页面找到名称为 `ddns` 的脚本，点击 Disabled 转为 Enabled 再返回 Service —— Dynamic DNS 中查看是否启用

### 配置 Cloudflare DDNS 服务

1. Lookup Hostname 填写 `openwrt.example.com`，IP地址版本选 IPv6
2. Domain 注意用`@`分割子域名，这里则填入 `openwrt@example.com`
3. Username 填写账号 `Bearer`，这个是API Token的默认账号。Password 填写前面小节复制的令牌
4. 勾选 Use HTTP Secure，在 Path to CA-Certificate 中填入 `/etc/ssl/certs`
5. 点击 Advanced Settings 选项卡。IP address source 保持默认 Network。Network接口选择虚拟接口 `xxxx_6`（如果是Openwrt拨号）或 `lan`
6. 点击保存并应用。等待几分钟用手机或其他与Openwrt不在同一个局域网内的电脑 `ping -6 -c 4 openwrt.example.com` 命令检查是否成功。如果失败，重启 DDNS 服务试试。

## 相关参考链接

- [OpenWrt Cloudflare DDNS](https://tao.zz.ac/unix/openwrt-ddns-cloudflare.html)
- [DDNS Client](https://openwrt.org/docs/guide-user/services/ddns/client)
