# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**edgetunnel** 是一个基于 Cloudflare Workers/Pages 平台的边缘计算隧道项目，约 5900 行的单文件 JavaScript 实现。核心功能包括 WebSocket/gRPC/XHTTP 代理、订阅生成、多种协议支持（VLESS/Trojan/Shadowsocks）。

## 代码结构

### 主入口 (`_worker.js`)

```javascript
export default {
    async fetch(request, env, ctx) { /* 路由分发 */ }
}
```

所有请求通过 `fetch` 入口处理，根据路径和请求类型分发到不同处理器：

| 路径/条件 | 处理器 | 说明 |
|-----------|--------|------|
| `/version` | 版本接口 | 返回版本号 |
| `Upgrade: websocket` | `处理 WS 请求` | WebSocket 代理 |
| `POST + gRPC content-type` | `处理 gRPC 请求` | gRPC 代理 |
| `POST + XHTTP 特征` | `处理 XHTTP 请求` | XHTTP 代理 |
| `/admin/*` | 管理面板 | 配置管理、日志查看 |
| `/sub` | 订阅生成 | Clash/Singbox/Surge等格式 |
| 其他 | 伪装页 | 反向代理到指定 URL |

### 核心函数模块

#### 协议处理
- `处理 WS 请求` (L1113): WebSocket 连接处理，包含 VLESS/Trojan 解析
- `处理 gRPC 请求` (L823): gRPC 流式传输
- `处理 XHTTP 请求` (L513): XHTTP 协议处理

#### 数据传输
- `forwardataTCP` (L1855): TCP 数据转发核心
- `forwardataudp` (L2084): UDP 数据转发
- `创建上行写入队列` (L2142): 上行数据队列管理
- `创建下行 Grain 发送器` (L2326): 下行分片传输优化

#### 加密与安全
- TLS 客户端实现：`buildClientHello` (L2999)、`parseServerHello` (L2924)
- ChaCha20-Poly1305: `chacha20Poly1305Encrypt` (L2847)
- Shadowsocks AEAD: `SSAEAD 加密` (L1841)、`SSAEAD 解密` (L1848)
- UUID 验证：`UUID 字节匹配` (L1642)

#### 链式代理
- `socks5Connect` (L2487): SOCKS5 代理连接
- `httpConnect` (L2523): HTTP 代理连接
- `httpsConnect` (L2581): HTTPS 代理连接
- `turnConnect` (L3489): TURN 协议连接
- `sstpConnect` (L3676): SSTP 协议连接

#### 订阅生成
- `Clash 订阅配置文件热补丁` (L4155): Clash 格式转换
- `Singbox 订阅配置文件热补丁` (L4372): Singbox 格式转换
- `Surge 订阅配置文件热补丁` (L4652): Surge 格式转换

#### 工具函数
- `MD5MD5` (L4737): 双重 MD5 哈希
- `base64SecretEncode` (L4093): 带密钥的 Base64 编码
- `DoH 查询` (L4769): DNS over HTTPS 解析
- `识别运营商` (L5137): 运营商识别（cmcc/ctcc/unicom）
- `请求优选 API` (L5259): 优选 IP API 请求

## KV 存储结构

配置通过 KV 命名空间持久化：

| Key | 类型 | 说明 |
|-----|------|------|
| `config.json` | JSON | 主配置文件（协议、路径、节点设置） |
| `cf.json` | JSON | Cloudflare API 凭证 |
| `tg.json` | JSON | Telegram 通知配置 |
| `ADD.txt` | Text | 自定义优选 IP 列表 |
| `log.json` | JSON | 请求日志数组 |

## 环境变量

| 变量名 | 必填 | 说明 |
|--------|------|------|
| `ADMIN` | ✅ | 后台管理密码 |
| `KV` | ✅ | KV 命名空间绑定 |
| `KEY` | ❌ | 快速订阅路径密钥 |
| `UUID` | ❌ | 固定 UUID（必须是 UUIDv4 格式） |
| `PROXYIP` | ❌ | 全局反代 IP |
| `URL` | ❌ | 伪装页 URL |
| `GO2SOCKS5` | ❌ | 强制走 SOCKS5 的域名名单 |
| `DEBUG` | ❌ | 开启调试日志 |
| `OFF_LOG` | ❌ | 关闭日志记录 |
| `BEST_SUB` | ❌ | 作为优选订阅生成器 |
| `PRELOAD_RACE_DIAL` | ❌ | 预加载竞速拨号 |

## 开发注意事项

### 代码风格
- **中文变量名**: 项目使用中文命名约定（如 `访问路径`, `管理员密码`）
- **单文件架构**: 所有逻辑在 `_worker.js` 中，约 5900 行
- **无外部依赖**: 纯原生 JavaScript，不依赖 npm 包

### 关键约束
1. **CF Workers 限制**: 注意执行时间、内存限制
2. **async context**: `ctx.waitUntil()` 用于异步后台任务（日志、KV 更新）
3. **版本常量**: 文件开头 `Version` 常量需随更新同步修改

### wrangler.toml 配置
```toml
name = "broken-glade"
main = "_worker.js"
compatibility_date = "2025-11-04"

[[kv_namespaces]]
binding = "KV"
id = "38b34b37260749ff81f7cb1e77c119bc"
```

### 部署方式
1. **Workers 部署**: 直接粘贴 `_worker.js` 内容
2. **Pages 上传**: 下载源码 zip 上传
3. **Pages + GitHub**: Fork 后连接 CF Pages

## API 端点

### 管理接口（需认证）
- `GET /admin/config.json` - 获取配置
- `POST /admin/config.json` - 保存配置
- `GET /admin/log.json` - 获取日志
- `POST /admin/ADD.txt` - 保存优选 IP
- `GET /admin/getCloudflareUsage` - 查询用量
- `GET /admin/check` - 代理检测

### 订阅接口
- `GET /sub?token=xxx` - 生成订阅
- `GET /{KEY}` - 快速订阅（KEY 环境变量值）

### 版本接口
- `GET /version?uuid=xxx` - 返回版本号

## GitHub Actions

- `.github/workflows/sync.yml`: 每日自动同步上游仓库变更
