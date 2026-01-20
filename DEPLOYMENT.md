# 博客部署到 GitHub Pages 指南

## 部署步骤

### 1. 在 GitHub 上创建仓库
1. 登录 GitHub
2. 点击右上角的 "+" 号，选择 "New repository"
3. 仓库名称设置为 `myblog`（或你喜欢的名称）
4. 选择 Public（GitHub Pages 需要 public 仓库）
5. 不要初始化 README、.gitignore 或 license
6. 点击 "Create repository"

### 2. 推送代码到 GitHub
```bash
# 添加远程仓库（替换 your-username 为你的 GitHub 用户名）
git remote add origin https://github.com/your-username/myblog.git

# 推送到 GitHub
git push -u origin main
```

### 3. 配置 GitHub Pages
1. 在 GitHub 仓库页面，点击 "Settings" 标签
2. 在左侧菜单中找到 "Pages"
3. 在 "Source" 部分选择 "GitHub Actions"
4. 保存设置

### 4. 触发部署
推送代码后，GitHub Actions 会自动运行部署工作流。你可以在 "Actions" 标签页查看部署状态。

### 5. 访问你的博客
部署完成后，你的博客将在以下地址可用：
```
https://your-username.github.io/myblog/
```

## 本地开发

### 安装依赖
```bash
uv sync
```

### 启动本地服务器
```bash
uv run mkdocs serve
```
然后在浏览器中访问 `http://localhost:8000`

### 构建静态站点
```bash
uv run mkdocs build
```

## 自定义配置

### 修改站点信息
编辑 `mkdocs.yml` 文件：
- `site_name`: 站点名称
- `site_url`: 站点 URL（记得修改为你的用户名）
- `site_author`: 作者姓名

### 添加新页面
1. 在 `docs/` 目录下创建新的 `.md` 文件
2. 在 `mkdocs.yml` 的 `nav` 部分添加导航链接

### 自定义主题
可以在 `mkdocs.yml` 中修改主题配置，参考 [Material for MkDocs 文档](https://squidfunk.github.io/mkdocs-material/)

## 注意事项

1. 确保 `mkdocs.yml` 中的 `site_url` 设置正确
2. 第一次部署可能需要几分钟时间
3. 每次推送到 `main` 分支都会触发自动部署
4. 可以通过 GitHub Actions 的日志查看部署过程中的任何错误