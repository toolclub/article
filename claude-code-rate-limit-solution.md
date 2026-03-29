# 突破 Claude Code 多账户 Rate Limit 限制：一次踩坑实录

## 背景

我是 Claude Pro 订阅用户，昨天一个账户被Rate Limit 限制。一个账号的额度根本不够用，于是我想到：搞两个账号，一个跑在 Windows 本机，一个跑在 Docker 容器里，双开干活。

听起来很完美，对吧？

## 第一步：Docker 双开，翻车了

我在 Windows 上装了 Docker Desktop，拉了一个 Ubuntu 开发镜像，里面装好 Claude Code，用第二个账号登录。

本机一个 Claude Code，Docker 里一个 Claude Code，两个不同的 Pro 账号，互不干扰——我是这么想的。

结果两边同时用了一会儿，Docker 里先报了：

```
API Error: Rate limit reached
```

紧接着，Windows 本机也跟着报了同样的错误。

两个完全不同的账号，怎么会互相影响？

## 第二步：定位问题——IP 锁定

我开始排查。先看了两边的出口 IP：

```bash
# Docker 容器内
curl ifconfig.me
# 结果：87.124.109.176

# Windows 本机
curl.exe ifconfig.me
# 结果：87.124.109.185
```

虽然最后一位不同，但都是 87.124.109.x——同一个 VPN 出口 IP 段。

真相大白：Anthropic 的 rate limit 不只是按账号算，还按 IP（或 IP 段）算。 Docker 容器默认走宿主机的网络，两个账号从同一个 IP 段发请求，被合并计算额度。

## 第三步：解题思路

问题的本质是两个 Claude Code 实例共享了同一个出口 IP。解决方案很简单：让它们走不同的出口 IP。

我在中国大陆，访问 Anthropic API 必须走代理。本机已经在用一个付费 VPN（LetsTAP），所以我需要给 Docker 找一条不同的出口线路。

最终选择了 Cloudflare WARP——免费、稳定、配置简单。

## 第四步：完整配置过程

### 4.1 安装 Cloudflare WARP

去 https://1.1.1.1/ 下载 Windows 客户端，安装一路默认。

### 4.2 设置为本地代理模式

WARP 默认是全局模式，会和已有的 VPN 冲突。需要改成代理模式，只开一个本地 SOCKS5 端口：

```powershell
# 切换到代理模式（不接管全局流量）
& "C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe" mode proxy

# 连接
& "C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe" connect
```

连接成功后，WARP 会在 127.0.0.1:40000 开一个 SOCKS5 代理。

验证一下：

```powershell
# 确认端口在监听
netstat -ano | findstr "40000"

# 测试 WARP 出口 IP
curl.exe -x socks5://127.0.0.1:40000 ifconfig.me
# 结果：104.28.246.148（WARP 的出口 IP）

# 对比本机 VPN 出口 IP
curl.exe ifconfig.me
# 结果：87.124.109.180（LetsTAP 的出口 IP）
```

两个 IP 完全不同。

### 4.3 验证 Docker 能访问 WARP 代理

Docker Desktop for Windows 支持用 host.docker.internal 访问宿主机：

```powershell
docker run --rm curlimages/curl -x socks5://host.docker.internal:40000 ifconfig.me
# 结果：104.28.246.148 ✅
```

### 4.4 启动 Docker 容器

启动时传入代理环境变量：

```powershell
docker run -it \
  -e ALL_PROXY=http://host.docker.internal:40000 \
  -e HTTPS_PROXY=socks5://host.docker.internal:40000 \
  -e HTTP_PROXY=socks5://host.docker.internal:40000 \
  fullstack-dev:latest bash
```

> 关键： Claude Code 实际识别的是 ALL_PROXY 环境变量，而且需要用 http:// 协议头而不是 socks5://。HTTPS_PROXY 和 HTTP_PROXY 是给 npm、curl 等其他工具用的。

### 4.5 容器内持久化配置

进容器后把代理写进 .bashrc，以后不用每次手动设：

```bash
echo 'export ALL_PROXY=http://host.docker.internal:40000' >> ~/.bashrc
```

## 日常使用：开机后要做的事

1. 确保主 VPN（LetsTAP）已连接——否则本机没外网
2. 启动 WARP 代理：

```powershell
& "C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe" mode proxy
& "C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe" connect
```

1. 启动 Docker 容器时带代理参数（或者用已经写好 .bashrc 的容器）

我写了一个 .bat 脚本放在桌面，双击自动完成前两步。

## 最终效果

| 实例 | 出口 IP | 账号 |
| --- | --- | --- |
| Windows 本机 Claude Code | 87.124.109.x（LetsTAP） | 账号 A |
| Docker 容器 Claude Code | 104.28.246.x（WARP） | 账号 B |

两个 Claude Code 同时运行，各自独立的 IP 和账号，rate limit 各算各的，再也没有互相干扰。

## 总结

- 问题根因： Anthropic 按 IP（段）做 rate limit，Docker 默认共享宿主机网络出口
- 解决方案： 用免费的 Cloudflare WARP 作为第二条出口线路
- 核心配置： WARP 代理模式 → 127.0.0.1:40000 → Docker 通过 host.docker.internal:40000 访问
- 关键踩坑： Claude Code 认的是 ALL_PROXY 环境变量，协议头用 http:// 而不是 socks5://

