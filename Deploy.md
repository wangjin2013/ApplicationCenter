# 部署文档：阿里云 ECS / 轻量应用服务器（Nginx）

把本站（[index.html](index.html) + [data/apps.json](data/apps.json) + `assets/`）部署到阿里云 ECS 或轻量应用服务器的完整流程。面向维护者。

**前置条件**：一台 ECS / 轻量应用服务器（Ubuntu、Debian 或 Alibaba Cloud Linux）、SSH 登录权限、一个域名。

下文命令中的 `<公网IP>`、`<你的域名>` 需自行替换。

---

## 1. 两个硬约束

**（1）必须有域名 + 有效 HTTPS 证书**

iOS 端的 ATS 默认拒绝明文 HTTP，而裸 IP 拿不到可用证书（Let's Encrypt 的 IP 证书只有 6 天有效期），所以「域名 + 证书」不是可选项——只要 App 要从这台服务器取数据，就绕不开。

**（2）大陆地域的域名必须先 ICP 备案**

- 未备案域名解析到大陆服务器，80/443 的网页访问会被阿里云拦截；直接 IP 访问不被拦，但对 App 无用（见上一条）
- 备案周期约 1–2 周。想立刻上线就用**香港地域**：免备案，大陆访问速度略逊，但仍通常好于 GitHub Pages

---

## 2. 服务器初始化

```sh
ssh root@<公网IP>

# Ubuntu / Debian
apt update && apt install -y nginx rsync
# Alibaba Cloud Linux / CentOS 用：dnf install -y nginx rsync

mkdir -p /var/www/applicationcenter
```

写入站点配置 `/etc/nginx/conf.d/applicationcenter.conf`：

```sh
cat > /etc/nginx/conf.d/applicationcenter.conf <<'EOF'
server {
    listen 80;
    server_name <你的域名> <公网IP>;

    root /var/www/applicationcenter;
    index index.html;

    # 与 GitHub Pages 行为对齐：允许跨域 + 10 分钟缓存
    add_header Access-Control-Allow-Origin "*" always;
    add_header Cache-Control "public, max-age=600" always;

    gzip on;
    gzip_types text/html text/css application/javascript application/json image/svg+xml;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

nginx -t && systemctl enable --now nginx && systemctl reload nginx
```

**缓存为什么是 10 分钟而不是常见的 7 天**：截图是**同名替换**的（`knowcal-1.jpg` 内容会变、文件名不变），长缓存会让改图后用户一周看不到新图。10 分钟与现有 GitHub Pages 的 `max-age=600` 一致。

**放行端口**：在阿里云控制台放行 80 / 443。ECS 是「安全组」，轻量应用服务器是独立的「防火墙」规则，别漏了。

---

## 3. 上传（本地仓库目录执行）

关键一步是**替换 JSON 里的绝对地址**：[data/apps.json](data/apps.json) 里的图片 URL 是硬编码的完整地址（设计决定见 [design.md](design.md) 第 4 节），不替换的话数据在阿里云、图片仍回 GitHub，等于没加速。

```sh
cd /Users/wang/Downloads/code/github_app

# 1) 拷到临时目录并替换域名（仓库本身不动，GitHub Pages 那份仍自洽）
rm -rf /tmp/site && mkdir -p /tmp/site
rsync -a --exclude '.git' --exclude '.DS_Store' --exclude '*.md' ./ /tmp/site/
sed -i '' 's|https://wangjin2013.github.io/ApplicationCenter|https://<你的域名>|g' /tmp/site/data/apps.json
# Linux 上 sed 写法为：sed -i 's|…|…|g' /tmp/site/data/apps.json

# 2) 上传，--delete 使服务器与本地完全一致
rsync -avz --delete /tmp/site/ root@<公网IP>:/var/www/applicationcenter/
```

`--exclude '*.md'` 是不把 README、design、CLAUDE.md 等维护文档发到线上；想让线上与 GitHub Pages 完全一致就去掉它。

**单一数据源不变**：GitHub 仓库仍是唯一数据源，阿里云那份是部署时生成的副本——改 `data/apps.json` → push → 重跑第 3 节的命令，双端同步，与 [design.md](design.md) 第 2 节的原则一致。

---

## 4. HTTPS 证书

```sh
apt install -y certbot python3-certbot-nginx
certbot --nginx -d <你的域名>
```

非 Ubuntu / Debian 系统（Alibaba Cloud Linux 等）建议改用阿里云「数字证书管理服务」的免费证书：控制台下载 Nginx 版证书，手工填 `ssl_certificate` / `ssl_certificate_key` 两行。证书签发要求域名已解析且已备案，否则 certbot 的验证请求会被拦截而失败。

---

## 5. 验证

```sh
curl -sI https://<你的域名>/ | head -1
curl -s https://<你的域名>/data/apps.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(len(d['apps']),'apps'); print(d['apps'][0]['screenshots'][0])"
curl -sI https://<你的域名>/assets/screenshots/thumbs/knowcal-1.jpg | head -1
```

第二条的输出应指向**新域名**而不是 github.io——这是整件事成没成的判据。

---

## 6. 例行更新

首次部署完成后，每次更新只需重跑**第 3 节**的三条命令（`rsync` → `sed` → `rsync`），第 2、4 节不用再做。

---

## 7. App 端接口地址

iOS 侧的接口常量要改为 `https://<你的域名>/data/apps.json`（对应 [ios-integration.md](ios-integration.md) 第 1 节的接口表）。App 未上架时改动代价最小；一旦上架，这个地址就冻结了。

---

## 8. 备注

- **带宽**：截图站点图片占大头。轻量应用服务器一般含 3–5 Mbps 峰值尚可；ECS 若只买 1 Mbps 会被图片拖慢，必要时把 `assets/` 放 OSS + CDN
- **回退**：GitHub Pages 可继续保留作为备用——它的 JSON 仍指向 github.io，自成一套、无需改动
