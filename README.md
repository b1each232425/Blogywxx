# 个人项目展示页

这是一个最小 Vue/Vite 个人网站，用来部署在服务器根路径。访问公网 IP 时先进入个人主页，再从主页跳转到各个演示系统。

当前项目入口：

- 学科竞赛管理系统：`/compemanage/#/home`

## 本地运行

```bash
npm install
npm run dev
```

默认开发地址：

```text
http://localhost:5220/
```

## 构建

```bash
npm run build
```

构建产物会输出到：

```text
dist/
```

## 服务器部署建议

推荐目录结构：

```text
/var/www/personal-site       # 个人主页 dist
/var/www/compemanage         # 竞赛系统 dist
```

访问关系：

```text
http://1.117.230.218/                    -> 个人主页
http://1.117.230.218/compemanage/#/home  -> 学科竞赛管理系统
http://1.117.230.218/api/...             -> 竞赛系统后端 API
```

Nginx 配置示例见：

```text
deploy/nginx-personal-site.conf
```

完整部署流程说明见：

```text
deploy/deployment-guide.md
```

竞赛系统前端建议用 `/compemanage/` 作为 Vite base 重新构建：

```bash
cd ../CompeManage_frontend/CompeManage
VITE_BASE_PATH=/compemanage/ npm run build
```

Windows PowerShell 写法：

```powershell
cd ..\CompeManage_frontend\CompeManage
$env:VITE_BASE_PATH="/compemanage/"
npm run build
```

然后把竞赛系统 `dist` 发布到 `/var/www/compemanage`。

如果需要统一身份认证跳回竞赛系统，也要把后端的 `CAS_FRONTEND_URL` 调整为：

```text
http://1.117.230.218/compemanage
```
