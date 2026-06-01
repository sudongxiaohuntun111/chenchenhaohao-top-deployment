# chenchenhaohao.top 项目部署手册

> **适用对象**：Hermes Agent / Claude Code / 其他 AI 智能体
> **目标**：通过标准化流程将任意 Web 项目部署到 `chenchenhaohao.top` 域名下，实现 HTTPS 访问并绕过阿里云 ICP 限制。

---

## 📐 整体架构

```
用户浏览器
    ↓ HTTPS
Cloudflare CDN (免费 SSL + 代理)
    ↓ Cloudflare Tunnel (加密连接)
VPS (47.94.161.94 阿里云北京)
    ↓ Nginx 反向代理 (按 server_name 路由)
/var/www/<project_name>/ (静态文件或动态服务)
```

**关键设计原则**：
- **完全绕过 ICP**：Cloudflare Tunnel 不依赖 VPS 的 80/443 端口，阿里云无法检测域名访问
- **零成本**：Cloudflare Tunnel 免费，Nginx 免费
- **多项目隔离**：每个子域名独立 Nginx server 块，互不干扰
- **HTTPS 自动生效**：Cloudflare 代理自动提供证书

---

## 🔧 环境信息

| 组件 | 值 | 说明 |
|------|-----|------|
| **VPS IP** | `47.94.161.94` | 阿里云轻量服务器（北京） |
| **主域名** | `chenchenhaohao.top` | Cloudflare 管理 |
| **DNS 服务器** | `dakota.ns.cloudflare.com`, `luciana.ns.cloudflare.com` | 已切换到 Cloudflare |
| **Cloudflare Account** | `2a833819261061a4042ce85ef3704f8e` | |
| **Cloudflare Zone** | `3f24e58e7ac47197e6fd1e304a26fb0e` | |
| **当前 Tunnel** | `6d839fba-a5fe-4af9-a46f-7137ca6259fb` | `english-speaking-trainer` |
| **SSH 用户** | `root` | 密码/密钥已配置 |
| **Nginx** | `1.18.0 (Ubuntu)` | 监听 18080+ 端口 |
| **Cloudflared** | `2026.5.0` | 已安装至 `/usr/local/bin/cloudflared` |

---

## 🚀 部署新项目的标准流程

### 第 1 步：在 VPS 上创建项目目录并上传文件

```bash
# SSH 到 VPS
ssh root@47.94.161.94

# 创建项目目录
mkdir -p /var/www/<project_name>

# 上传文件（从本地或 SCP）
scp -r ./project_files/* root@47.94.161.94:/var/www/<project_name>/
```

### 第 2 步：配置 Nginx

在 VPS 上创建 Nginx 配置文件：

```bash
cat > /etc/nginx/sites-available/<project_name> << 'EOF'
server {
    listen <PORT>;
    server_name <subdomain>.chenchenhaohao.top;

    root /var/www/<project_name>;
    index index.html;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

# 启用站点并重启 Nginx
ln -sf /etc/nginx/sites-available/<project_name> /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

**端口分配规则**：
- `18080`：默认端口（当前用于 `english-speaking-trainer` 和 `kids`）
- `18090`：备用端口（可用于新项目）
- 每个新子域名建议使用 **不同端口**（如 `18091`, `18092`...）

### 第 3 步：配置 Cloudflare Tunnel

编辑 Tunnel 配置文件：

```bash
cat > /etc/cloudflared/config.yml << 'EOF'
tunnel: 6d839fba-a5fe-4af9-a46f-7137ca6259fb
credentials-file: /root/.cloudflared/credentials.json

ingress:
  - hostname: english-speaking-trainer.chenchenhaohao.top
    service: http://localhost:18080
  - hostname: kids.chenchenhaohao.top
    service: http://localhost:18080
  - hostname: <subdomain>.chenchenhaohao.top
    service: http://localhost:<PORT>
  - service: http_status:404
EOF

# 重启 cloudflared 服务
systemctl restart cloudflared-est
```

### 第 4 步：添加 DNS 记录

通过 Cloudflare API 添加 CNAME 记录（需要 API Token）：

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/3f24e58e7ac47197e6fd1e304a26fb0e/dns_records" \
  -H "Authorization: Bearer <API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "CNAME",
    "name": "<subdomain>.chenchenhaohao.top",
    "content": "6d839fba-a5fe-4af9-a46f-7137ca6259fb.cfargotunnel.com",
    "proxied": true
  }'
```

或通过 Dashboard 手动添加：
- Cloudflare → DNS → Records → Add record
- Type: `CNAME`, Name: `<subdomain>`, Target: `6d839fba...cfargotunnel.com`, Proxy: `On`

### 第 5 步：验证

```bash
# 检查 Nginx 配置
curl -I http://localhost:<PORT>/

# 检查 HTTPS 访问
curl -I https://<subdomain>.chenchenhaohao.top/

# 检查 DNS 解析
dig +short <subdomain>.chenchenhaohao.top @8.8.8.8
```

---

## 📁 当前已部署项目

| 子域名 | Nginx 端口 | 目录 | 说明 |
|--------|-----------|------|------|
| `kids.chenchenhaohao.top` | 18080 | `/var/www/kids/` | 儿童小游戏合集 (84MB) |
| `english-speaking-trainer.chenchenhaohao.top` | 18080 | `/var/www/english-speaking-trainer-v2/` | 英语语音训练游戏 (含 300 个 MP3) |
| `chenchenhaohao.top` | 80 | 代理到 `127.0.0.1:19828` | Wiki API 服务 |

---

## ⚠️ 注意事项

1. **Nginx server_name 冲突**：多个项目可以共用同一端口（如 18080），Nginx 通过 `server_name` 区分。但建议新项目使用独立端口避免混淆。
2. **Cloudflare Tunnel Ingress 顺序**：`config.yml` 中的 ingress 规则**从上到下匹配**，第一个匹配的 hostname 生效。必须把具体 hostname 放在前面，`http_status:404` 放最后作为兜底。
3. **音频/大文件**：Cloudflare Tunnel 对大文件传输支持良好，但首次加载可能较慢。MP3/视频建议预生成并放在 `audio/` 子目录。
4. **HTTPS 强制**：Cloudflare 代理开启后（橙色云朵），HTTP 会自动跳转到 HTTPS。麦克风/摄像头等 API 在 HTTPS 下才能正常工作。
5. **缓存问题**：Cloudflare 会缓存静态文件。更新内容后建议在 Dashboard 中 Purge Cache，或使用版本号（`script.js?v=2`）绕过缓存。
6. **凭证安全**：`/root/.cloudflared/credentials.json` 包含 Tunnel 密钥，**不要公开**。API Token 也应妥善保管。

---

## 🛠️ 常用运维命令

```bash
# 查看 Tunnel 状态
systemctl status cloudflared-est
journalctl -u cloudflared-est -n 50

# 查看 Nginx 状态
systemctl status nginx
nginx -t

# 重新加载 Nginx 配置（不停机）
systemctl reload nginx

# 查看当前监听的端口
ss -tlnp | grep nginx

# 手动 Purge Cloudflare 缓存（需要 API Token）
curl -X POST "https://api.cloudflare.com/client/v4/zones/3f24e58e7ac47197e6fd1e304a26fb0e/purge_cache" \
  -H "Authorization: Bearer <API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"purge_everything": true}'
```

---

## 📦 备份与恢复

项目文件位于 `/var/www/`，定期备份：
```bash
tar -czf /backup/www_$(date +%Y%m%d).tar.gz /var/www/
```

Nginx 配置备份：
```bash
tar -czf /backup/nginx_$(date +%Y%m%d).tar.gz /etc/nginx/
```

---

*文档生成时间：2026-06-01*
*维护者：Hermes Agent*
