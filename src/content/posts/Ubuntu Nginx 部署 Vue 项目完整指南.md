---

title: Ubuntu Nginx 部署 Vue 项目完整指南
published: 2026-06-07
description: 从 Nginx 安装与服务管理讲起，涵盖 Vue 3（Vite）项目打包部署、Vue Router 的 history 与 hash 模式、子路径部署及反向代理等常见场景。
image: ./images/ubuntu-nginx-vue-cover.webp
tags: [Vue, 开发]
category: 前端开发
draft: false

---





# Ubuntu Nginx 部署 Vue 项目完整指南

> 环境：Ubuntu（WSL2）+ Nginx + Vue 3（Vite）

------

## 一、Nginx 基础

### 1.1 安装

```bash
sudo apt update
sudo apt install nginx -y
nginx -v    # 确认安装成功
```

### 1.2 服务管理

```bash
# systemctl 方式（普通 Ubuntu）
sudo systemctl start nginx       # 启动
sudo systemctl stop nginx        # 停止
sudo systemctl restart nginx     # 重启
sudo systemctl reload nginx      # 重新加载配置（不中断服务）
sudo systemctl enable nginx      # 开机自启
sudo systemctl disable nginx     # 禁用开机自启

# 直接命令方式（WSL2 不支持 systemctl 时使用）
sudo nginx                       # 启动
sudo nginx -s stop               # 停止
sudo nginx -s reload             # 重新加载配置
sudo nginx -t                    # 测试配置语法
```

### 1.3 重要目录与文件

| 路径                          | 说明                   |
| ----------------------------- | ---------------------- |
| `/etc/nginx/nginx.conf`       | 主配置文件             |
| `/etc/nginx/sites-available/` | 站点配置文件（仓库）   |
| `/etc/nginx/sites-enabled/`   | 已启用的站点（软链接） |
| `/etc/nginx/conf.d/`          | 额外配置片段           |
| `/var/www/html/`              | 默认网站根目录         |
| `/var/log/nginx/access.log`   | 访问日志               |
| `/var/log/nginx/error.log`    | 错误日志               |

### 1.4 站点启用机制

Nginx 通过两个目录管理站点：

```
/etc/nginx/
├── sites-available/     ← 存放所有站点配置文件（仓库）
│   ├── default          ← 默认站点
│   └── my-app           ← 你自己的站点
│
└── sites-enabled/       ← 存放软链接，指向 sites-available/ 中的文件
    └── default → ../sites-available/default
```

- Nginx **只加载** `sites-enabled/` 目录下的配置
- 启用站点：创建软链接到 `sites-enabled/`
- 停用站点：删除 `sites-enabled/` 中的软链接（配置文件本身不丢失）

```bash
# 启用站点
sudo ln -s /etc/nginx/sites-available/站点名 /etc/nginx/sites-enabled/

# 停用站点
sudo rm /etc/nginx/sites-enabled/站点名
```

------

## 二、虚拟主机概念

**虚拟主机**（Virtual Host）就是在一台服务器上同时运行多个网站。

Nginx 通过 `server_name` 区分不同域名，每个 `server { }` 块就是一个虚拟主机。

```
用户访问 a.com ──→ 同一台服务器 ──→ server_name a.com ──→ /var/www/a/
用户访问 b.com ──→ 同一台服务器 ──→ server_name b.com ──→ /var/www/b/
```

### 配置示例

```nginx
# 站点 A
server {
    listen 80;
    server_name a.com;
    root /var/www/a;
}

# 站点 B
server {
    listen 80;
    server_name b.com;
    root /var/www/b;
}
```

### ⚠️ 注意事项

- 当多个站点的 `server_name` 都能匹配同一个请求时（比如都用 `localhost`），Nginx 会选择**最先加载的**那个
- 默认站点 `default` 的配置文件名会导致它排在前面，需要删掉或改名避免冲突
- 本站点之间通过域名（或 IP + 端口）区分，互不干扰

------

## 三、部署 Vue 项目（无路由 / 单页面）

> 适用于：纯单页面应用（如 TodoList），没有使用 Vue Router

### 3.1 打包项目

```bash
# 在项目根目录执行
npm run build
```

执行后生成 `dist/` 文件夹，这是要部署的全部文件。

### 3.2 部署到 Nginx

```bash
# 1. 创建站点目录
sudo mkdir -p /var/www/todolist

# 2. 复制打包文件
sudo cp -r ./dist/* /var/www/todolist/

# 3. 创建 Nginx 配置文件
sudo nano /etc/nginx/sites-available/todolist
```

配置文件内容：

```nginx
server {
    listen 80;
    server_name localhost;     # 或你的域名/IP

    root /var/www/todolist;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # 静态资源缓存（可选）
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public";
    }
}
# 4. 删除默认站点（避免冲突）
sudo rm -f /etc/nginx/sites-enabled/default

# 5. 启用新站点
sudo ln -s /etc/nginx/sites-available/todolist /etc/nginx/sites-enabled/

# 6. 测试配置
sudo nginx -t

# 7. 重载/启动 Nginx
sudo nginx -s reload
# 或
sudo systemctl reload nginx
```

### 3.3 访问

浏览器打开 `http://localhost`（或你的服务器 IP / 域名）。

------

## 四、部署 Vue 项目（有 Vue Router）

> 适用于：使用了 Vue Router 的项目

### 4.1 判断路由模式

在 `src/router/index.js` 中查看：

```js
// history 模式
createRouter({
  history: createWebHistory(),   // ← 就是这个
  routes: [...]
})

// hash 模式
createRouter({
  history: createWebHashHistory(),
  routes: [...]
})
```

### 4.2 两种模式的区别

|            | hash 模式              | history 模式              |
| ---------- | ---------------------- | ------------------------- |
| URL 格式   | `example.com/#/about`  | `example.com/about`       |
| 原理       | `#` 后内容由浏览器处理 | 真实路径，需服务器配合    |
| Nginx 配置 | 不需要特殊处理         | **必须** 配置 `try_files` |
| SEO        | 差                     | 好                        |

### 4.3 history 模式配置

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/my-app;
    index index.html;

    # 关键：所有路径都返回 index.html，由前端路由处理
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 4.4 hash 模式配置

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/my-app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 4.5 配置说明

- `try_files $uri $uri/ /index.html;` — 先找对应文件，找不到就返回 `index.html`，让 Vue Router 接管路由
- `try_files $uri $uri/ =404;` — 找不到就返回 404（适用于没有前端路由的页面）

### ⚠️ history 模式注意事项

- 没有配置 `try_files ... /index.html` 时，用户在浏览器直接访问 `/about` 会返回 404
- 因为这个路径在服务器上没有对应的文件，必须回退到 `index.html`
- 配置了 `try_files` 后，Vue Router 会解析 URL 并显示正确的页面

------

## 五、部署到子路径

如果你的 Vue 项目部署在子路径下（如 `example.com/app/`），需要两步配合：

### 5.1 Vite 配置

```js
// vite.config.js
export default defineConfig({
  base: '/app/',
  plugins: [vue()],
})
```

### 5.2 Nginx 配置

```nginx
server {
    listen 80;
    server_name example.com;

    location /app/ {
        alias /var/www/my-app/;
        try_files $uri $uri/ /app/index.html;
    }
}
```

> ⚠️ 注意：子路径用 `alias` 而不是 `root`。`root` 会把路径拼接成 `/var/www/my-app/app/`，`alias` 才是直接指向指定目录。

------

## 六、常用功能配置

### 6.1 反向代理

将 API 请求代理到后端服务（如 Node.js 3000 端口）：

```nginx
location /api/ {
    proxy_pass http://localhost:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### 6.2 负载均衡

```nginx
upstream backend {
    server 127.0.0.1:3000 weight=3;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002 backup;
}

server {
    location /api/ {
        proxy_pass http://backend;
    }
}
```

### 6.3 Gzip 压缩

在 `server` 块外（http 块中）添加：

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
gzip_min_length 1000;
gzip_comp_level 6;
```

### 6.4 静态资源缓存

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;
    add_header Cache-Control "public";
}
```

- `expires 30d` — 浏览器缓存 30 天
- `immutable` — 告诉浏览器文件不会变，不用重新验证（适合带 hash 的文件名，Vite 打包默认带 hash）

------

## 七、排查问题

### 7.1 常用命令

```bash
# 测试配置语法
sudo nginx -t

# 查看错误日志
sudo tail -f /var/log/nginx/error.log

# 查看访问日志
sudo tail -f /var/log/nginx/access.log

# 检查 Nginx 是否运行
ps aux | grep nginx

# 检查端口占用
sudo ss -tlnp | grep :80
```

### 7.2 常见问题

| 问题                  | 可能原因                     | 解决                                       |
| --------------------- | ---------------------------- | ------------------------------------------ |
| 访问还是默认页面      | 没删 `default` 站点          | `sudo rm /etc/nginx/sites-enabled/default` |
| 刷新页面显示 404      | history 模式没配 `try_files` | 加上 `try_files $uri $uri/ /index.html;`   |
| 403 Forbidden         | 权限问题                     | `sudo chmod -R 755 /var/www/你的目录`      |
| 502 Bad Gateway       | 后端服务没启动               | 检查代理目标端口是否在运行                 |
| Nginx 启动失败        | 配置语法错误                 | `sudo nginx -t` 查看具体报错               |
| 访问 localhost 打不开 | Nginx 没启动或端口被占用     | `ps aux \| grep nginx` 确认进程            |
| 静态资源 404          | Vite 的 `base` 配置不对      | 检查 `vite.config.js` 中的 `base`          |

### 7.3 WSL2 特殊注意

- WSL2 可能不支持 `systemctl`，用 `sudo nginx` 直接启动
- 项目文件建议放在 WSL2 自己的文件系统（`~/projects/`），不要放在 `/mnt/c/`，否则打包很慢
- Windows 浏览器访问 `http://localhost` 即可，网络是互通的
- Windows 可能会弹出防火墙提示，点"允许"

------

## 八、HTTPS 配置（Let's Encrypt）

```bash
# 安装 certbot
sudo apt install certbot python3-certbot-nginx -y

# 自动配置 SSL
sudo certbot --nginx -d example.com -d www.example.com

# 测试自动续期
sudo certbot renew --dry-run
```

------

## 九、完整操作流程速查（无路由单页面）

```bash
# 1. 安装 Nginx
sudo apt update && sudo apt install nginx -y

# 2. 打包项目（在项目目录）
npm run build

# 3. 创建目录并复制文件
sudo mkdir -p /var/www/my-app
sudo cp -r ./dist/* /var/www/my-app/

# 4. 创建配置
sudo nano /etc/nginx/sites-available/my-app

# 5. 启用站点
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/

# 6. 测试并启动
sudo nginx -t
sudo nginx -s reload

# 7. 访问 http://localhost
```

------

*文档更新时间：2026年6月*