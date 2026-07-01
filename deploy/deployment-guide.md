# 个人主页与竞赛系统部署流程

本文档记录当前服务器的部署思路和操作步骤，方便以后重新部署、迁移服务器或排查问题。

目标效果：

```text
http://1.117.230.218/                    -> 个人主页 PersonalSite
http://1.117.230.218/compemanage/#/home  -> 学科竞赛管理系统前端
http://1.117.230.218/api/...             -> 学科竞赛管理系统后端 API
http://1.117.230.218/static/...          -> 后端静态文件
```

整体结构：

```text
浏览器
  |
  v
Nginx :80
  |-- /              -> /var/www/personal-site
  |-- /compemanage/  -> /var/www/compemanage
  |-- /api/          -> 127.0.0.1:8080/api/
  |-- /static/       -> 127.0.0.1:8080/static/
  |-- /health        -> 127.0.0.1:8080/health
  |
  v
Go 后端容器 :8080
  |-- MySQL
  |-- Redis
```

## 一、服务器登录

在本地 PowerShell 连接服务器：

```powershell
ssh -i E:\ywxx.pem ubuntu@1.117.230.218
```

含义：

- `ssh`：远程登录服务器。
- `-i E:\ywxx.pem`：指定腾讯云私钥。
- `ubuntu`：服务器用户名。
- `1.117.230.218`：服务器公网 IP。

登录后建议先确认基础环境：

```bash
cat /etc/os-release
docker --version
docker compose version
sudo systemctl status docker --no-pager
sudo systemctl status nginx --no-pager
```

含义：

- `cat /etc/os-release`：查看系统版本。
- `docker --version`：确认 Docker 是否安装。
- `docker compose version`：确认 Docker Compose 是否安装。
- `systemctl status docker`：确认后端容器运行环境是否正常。
- `systemctl status nginx`：确认 Nginx 是否正在运行。

## 二、目录规划

服务器上推荐使用以下目录：

```text
/opt/PersonalSite                         # 个人主页源码
/opt/CompeManage_frontend                 # 竞赛系统前端源码
/opt/CompeManage_backend                  # 竞赛系统后端源码
/var/www/personal-site                    # 个人主页构建产物
/var/www/compemanage                      # 竞赛系统前端构建产物
/etc/nginx/sites-available/compemanage    # Nginx 站点配置
/etc/nginx/sites-enabled/compemanage      # Nginx 启用配置软链接
```

含义：

- `/opt`：放项目源码，适合后续 `git pull` 更新。
- `/var/www`：放 Nginx 直接读取的静态文件。
- `/etc/nginx/sites-available`：放可用站点配置。
- `/etc/nginx/sites-enabled`：放已启用站点配置。

如果在 `/opt` 下执行 `npm install` 遇到权限错误，先把目录所有权交给当前用户：

```bash
sudo chown -R ubuntu:ubuntu /opt/PersonalSite
sudo chown -R ubuntu:ubuntu /opt/CompeManage_frontend
```

含义：

- `chown`：修改文件/目录所有者。
- `-R`：递归处理目录下所有文件。
- `ubuntu:ubuntu`：把所有者和用户组都改成 `ubuntu`。

不要优先使用 `sudo npm install`。这样会把 `node_modules` 变成 root 创建，后续普通用户更新会更麻烦。

## 三、部署个人主页 PersonalSite

### 1. 拉取源码

首次部署：

```bash
cd /opt
git clone https://github.com/b1each232425/Blogywxx.git PersonalSite
```

以后更新：

```bash
cd /opt/PersonalSite
git pull
```

含义：

- `git clone ... PersonalSite`：把 GitHub 仓库克隆到 `/opt/PersonalSite`。
- `git pull`：拉取远端最新代码。

### 2. 安装依赖

```bash
cd /opt/PersonalSite
npm install
```

含义：

- 根据 `package.json` 安装 Vue、Vite 等依赖。
- 安装结果会放到 `/opt/PersonalSite/node_modules`。

### 3. 构建静态文件

```bash
npm run build
```

含义：

- Vite 会把 Vue 项目构建成纯静态文件。
- 构建产物输出到 `/opt/PersonalSite/dist`。
- Nginx 不直接运行 Vue 源码，而是读取 `dist` 里的 HTML、JS、CSS。

### 4. 发布到 Nginx 静态目录

```bash
sudo mkdir -p /var/www/personal-site
sudo rm -rf /var/www/personal-site/*
sudo cp -r /opt/PersonalSite/dist/* /var/www/personal-site/
```

含义：

- `mkdir -p`：创建目标目录，如果已存在也不报错。
- `rm -rf /var/www/personal-site/*`：清空旧的个人主页构建文件。
- `cp -r dist/* ...`：复制最新构建产物给 Nginx。

注意：

- 这里清理的是 `/var/www/personal-site/*`，不是源码目录。
- 不要写错路径，尤其不要执行 `sudo rm -rf /var/www/*`。

## 四、部署竞赛系统前端到 /compemanage/

竞赛系统原来部署在网站根路径 `/`，现在要放到 `/compemanage/`。

因此构建时必须指定 Vite base：

```text
VITE_BASE_PATH=/compemanage/
```

如果不指定，前端资源可能会生成成：

```text
/assets/index-xxx.js
```

放到子路径后应该生成成：

```text
/compemanage/assets/index-xxx.js
```

否则访问 `/compemanage/#/home` 时，浏览器可能去根路径 `/assets/` 找资源，导致页面空白或 404。

### 1. 拉取 demo 分支

首次部署：

```bash
cd /opt
git clone -b demo-deploy https://github.com/James-FS/CompeManage_frontend.git
```

已有仓库时：

```bash
cd /opt/CompeManage_frontend
git fetch origin
git switch demo-deploy
git pull
```

含义：

- `demo-deploy` 分支包含演示部署需要的 README、截图脚本和 Vite `base` 配置。
- `git fetch origin`：更新远端分支信息。
- `git switch demo-deploy`：切换到演示分支。
- `git pull`：拉取最新提交。

### 2. 安装依赖

```bash
cd /opt/CompeManage_frontend/CompeManage
npm install
```

含义：

- 安装竞赛系统前端依赖。
- 如果已经安装过，并且 `package.json` 没变，也可以跳过。

### 3. 以 /compemanage/ 为 base 构建

```bash
VITE_BASE_PATH=/compemanage/ npm run build
```

含义：

- `VITE_BASE_PATH=/compemanage/`：告诉 Vite 构建资源时带上 `/compemanage/` 前缀。
- `npm run build`：生成生产环境静态文件。
- 构建产物输出到 `/opt/CompeManage_frontend/CompeManage/dist`。

如果在 Windows PowerShell 本地构建，写法是：

```powershell
$env:VITE_BASE_PATH="/compemanage/"
npm run build
```

### 4. 发布到 Nginx 静态目录

```bash
sudo mkdir -p /var/www/compemanage
sudo rm -rf /var/www/compemanage/*
sudo cp -r /opt/CompeManage_frontend/CompeManage/dist/* /var/www/compemanage/
```

含义：

- `/var/www/compemanage` 是竞赛系统前端的静态目录。
- Nginx 的 `/compemanage/` 会读取这个目录。

## 五、确认竞赛系统后端运行

后端通过 Docker Compose 运行，前端的 `/api/` 请求会由 Nginx 转发到本机 `8080` 端口。

进入后端目录：

```bash
cd /opt/CompeManage_backend
```

查看容器状态：

```bash
sudo docker compose -f docker-compose.prod.yml ps
```

期望看到类似：

```text
compemanage_app     Up     0.0.0.0:8080->8080/tcp
compemanage_mysql   Up
compemanage-redis   Up
```

检查健康接口：

```bash
curl http://127.0.0.1:8080/health
```

查看后端日志：

```bash
sudo docker logs --tail=100 compemanage_app
```

含义：

- `docker compose ps`：查看后端、MySQL、Redis 是否运行。
- `curl /health`：确认后端 HTTP 服务可访问。
- `docker logs`：排查后端启动、数据库连接、DataHall 开关等问题。

演示环境应保持：

```text
DATAHALL_ENABLED=false
```

这样不会拉取真实学生或教师数据。

## 六、配置 Nginx

当前推荐配置文件在个人主页仓库：

```text
/opt/PersonalSite/deploy/nginx-personal-site.conf
```

也可以直接覆盖服务器站点配置：

```bash
sudo tee /etc/nginx/sites-available/compemanage > /dev/null <<'EOF'
server {
    listen 80;
    server_name 1.117.230.218;

    root /var/www/personal-site;
    index index.html;

    # 个人主页：公网 IP 根路径优先进入这里
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 学科竞赛管理系统前端
    location /compemanage/ {
        alias /var/www/compemanage/;
        try_files $uri $uri/ /compemanage/index.html;
    }

    # 学科竞赛管理系统后端 API
    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        proxy_pass http://127.0.0.1:8080/health;
    }

    location /static/ {
        proxy_pass http://127.0.0.1:8080/static/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

配置含义：

- `listen 80`：监听 HTTP 80 端口。
- `server_name 1.117.230.218`：当前服务器公网 IP。
- `root /var/www/personal-site`：根路径默认读取个人主页。
- `location /`：个人主页使用 SPA 回退，刷新页面不会 404。
- `location /compemanage/`：竞赛系统前端放在子路径。
- `alias /var/www/compemanage/`：把 `/compemanage/` 映射到竞赛系统静态目录。
- `location /api/`：把前端 API 请求转发到 Go 后端。
- `location /static/`：把后端静态文件请求转发到 Go 后端。
- `location /health`：提供健康检查转发。

确认配置已启用：

```bash
sudo ln -sf /etc/nginx/sites-available/compemanage /etc/nginx/sites-enabled/compemanage
sudo rm -f /etc/nginx/sites-enabled/default
```

含义：

- `ln -sf`：把站点配置软链接到 `sites-enabled`，表示启用。
- `rm -f default`：移除默认 Nginx 站点，避免根路径被默认欢迎页占用。

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

含义：

- `nginx -t`：检查配置语法。
- `systemctl reload nginx`：不停止服务，重新加载新配置。

不要跳过 `nginx -t`。如果配置有错直接 reload，可能导致站点不可用。

## 七、验证部署结果

在服务器本机验证：

```bash
curl -I http://127.0.0.1/
curl -I http://127.0.0.1/compemanage/
curl http://127.0.0.1/health
```

含义：

- 第一个命令检查个人主页是否能返回。
- 第二个命令检查竞赛系统前端是否能返回。
- 第三个命令检查后端健康接口是否能返回。

在浏览器访问：

```text
http://1.117.230.218/
```

期望：

- 首屏显示个人项目展示页。
- 页面上有“学科竞赛管理系统”入口。

点击进入：

```text
http://1.117.230.218/compemanage/#/home
```

期望：

- 能打开竞赛系统。
- 如果未登录，会跳到登录页。
- 登录后能正常访问后台页面。

## 八、查看当前 Nginx 配置

查看启用中的站点：

```bash
ls -l /etc/nginx/sites-enabled/
```

查看当前站点配置：

```bash
sudo cat /etc/nginx/sites-available/compemanage
sudo cat /etc/nginx/sites-enabled/compemanage
```

查看 Nginx 最终加载后的完整配置：

```bash
sudo nginx -T
```

配置很多时可以筛选：

```bash
sudo nginx -T | grep -n "server_name\|root\|location\|proxy_pass"
```

查看日志：

```bash
sudo tail -n 80 /var/log/nginx/access.log
sudo tail -n 80 /var/log/nginx/error.log
```

含义：

- `access.log`：谁访问了什么路径，返回状态码是多少。
- `error.log`：Nginx 找不到文件、代理失败、配置异常等错误。

## 九、日常更新流程

### 只更新个人主页

```bash
cd /opt/PersonalSite
git pull
npm install
npm run build
sudo rm -rf /var/www/personal-site/*
sudo cp -r dist/* /var/www/personal-site/
sudo nginx -t
sudo systemctl reload nginx
```

### 只更新竞赛系统前端

```bash
cd /opt/CompeManage_frontend
git fetch origin
git switch demo-deploy
git pull

cd CompeManage
npm install
VITE_BASE_PATH=/compemanage/ npm run build
sudo rm -rf /var/www/compemanage/*
sudo cp -r dist/* /var/www/compemanage/
sudo nginx -t
sudo systemctl reload nginx
```

### 只更新竞赛系统后端

```bash
cd /opt/CompeManage_backend
git fetch origin
git switch demo-deploy
git pull
sudo docker compose -f docker-compose.prod.yml up -d --build
sudo docker compose -f docker-compose.prod.yml ps
curl http://127.0.0.1:8080/health
```

含义：

- 后端是 Docker 部署，更新后需要重新 build 镜像并启动容器。
- 前端是静态文件部署，更新后只需要重新 build 并复制 `dist`。

## 十、常见问题

### 1. npm install 提示 EACCES

现象：

```text
EACCES: permission denied, mkdir '/opt/PersonalSite/node_modules'
```

原因：

项目目录不是当前 `ubuntu` 用户拥有，普通用户无法创建 `node_modules`。

解决：

```bash
sudo chown -R ubuntu:ubuntu /opt/PersonalSite
sudo chown -R ubuntu:ubuntu /opt/CompeManage_frontend
```

然后重新执行：

```bash
npm install
```

### 2. 访问 /compemanage/ 页面空白

可能原因：

- 竞赛系统没有用 `VITE_BASE_PATH=/compemanage/` 重新构建。
- Nginx 的 `/compemanage/` `alias` 配置不对。
- `/var/www/compemanage` 里没有复制最新 `dist`。

排查：

```bash
ls -l /var/www/compemanage
sudo tail -n 80 /var/log/nginx/error.log
```

重新构建：

```bash
cd /opt/CompeManage_frontend/CompeManage
VITE_BASE_PATH=/compemanage/ npm run build
sudo rm -rf /var/www/compemanage/*
sudo cp -r dist/* /var/www/compemanage/
```

### 3. 竞赛系统能打开但接口报错

可能原因：

- 后端容器没有运行。
- Nginx `/api/` 代理配置错误。
- 后端数据库或 Redis 异常。

排查：

```bash
cd /opt/CompeManage_backend
sudo docker compose -f docker-compose.prod.yml ps
curl http://127.0.0.1:8080/health
sudo docker logs --tail=100 compemanage_app
sudo nginx -T | grep -n "location /api\|proxy_pass"
```

### 4. 修改 Nginx 后没生效

排查：

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx --no-pager
sudo nginx -T | grep -n "personal-site\|compemanage"
```

常见原因：

- 忘记 reload。
- 编辑了 `sites-available`，但 `sites-enabled` 没有软链接到它。
- 还有默认站点抢占了根路径。

### 5. 后端 DataHall 同步不要打开

演示环境保持：

```text
DATAHALL_ENABLED=false
```

原因：

- 演示站点不应该自动拉取真实学生、教师或组织机构数据。
- 如果要演示功能，使用种子数据或手工创建的测试数据即可。

## 十一、当前相关仓库

```text
个人主页：
https://github.com/b1each232425/Blogywxx

竞赛系统前端 demo 分支：
https://github.com/James-FS/CompeManage_frontend/tree/demo-deploy

竞赛系统后端 demo 分支：
https://github.com/James-FS/CompeManage_backend/tree/demo-deploy
```

