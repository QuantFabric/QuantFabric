# mdBook 使用指南

## 什么是 mdBook？

mdBook 是一个使用 Markdown 创建在线书籍的工具，由 Rust 编写。它可以将 Markdown 文件转换为美观、可搜索的静态网站。

## 安装 mdBook

### 使用 Cargo（Rust 包管理器）

```bash
cargo install mdbook
```

### 使用预编译二进制

从 [GitHub Releases](https://github.com/rust-lang/mdBook/releases) 下载对应平台的二进制文件。

## 构建文档

### 1. 构建静态 HTML

```bash
cd /home/quantaxis/qapro/QuantFabric/doc
mdbook build
```

生成的文件位于 `doc/book/` 目录。

### 2. 本地预览

```bash
cd /home/quantaxis/qapro/QuantFabric/doc
mdbook serve
```

然后在浏览器访问 http://localhost:3000

### 3. 指定端口

```bash
mdbook serve --port 8080
```

### 4. 监听外网访问

```bash
mdbook serve --hostname 0.0.0.0
```

## 部署文档

### 方式 1: GitHub Pages

1. 将 `doc/book/` 目录内容推送到 `gh-pages` 分支
2. 在 GitHub 仓库设置中启用 GitHub Pages

```bash
# 自动部署脚本
cd /home/quantaxis/qapro/QuantFabric/doc
mdbook build
cd book
git init
git add .
git commit -m "Deploy documentation"
git push -f git@github.com:QuantFabric/QuantFabric.git master:gh-pages
```

### 方式 2: Nginx 静态服务器

```nginx
server {
    listen 80;
    server_name docs.quantfabric.com;

    root /path/to/QuantFabric/doc/book;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 方式 3: Docker

```dockerfile
FROM nginx:alpine
COPY doc/book /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```bash
docker build -t quantfabric-docs .
docker run -d -p 80:80 quantfabric-docs
```

## 文档结构

```
doc/
├── book.toml           # mdBook 配置文件
├── SUMMARY.md          # 文档目录（必需）
├── README.md           # 首页内容
├── architecture/       # 架构文档
├── modules/            # 模块文档
├── deployment/         # 部署文档
├── glossary.md         # 术语表
├── faq.md              # 常见问题
└── book/               # 构建输出（自动生成）
```

## 配置说明 (book.toml)

```toml
[book]
title = "QuantFabric 文档"        # 书籍标题
authors = ["@scorpiostudio"]  # 作者
language = "zh-CN"                # 语言
src = "."                         # 源文件目录

[build]
build-dir = "book"                # 构建输出目录

[output.html]
default-theme = "light"           # 默认主题
git-repository-url = "..."        # GitHub 仓库 URL
edit-url-template = "..."         # 编辑链接模板

[output.html.search]
enable = true                     # 启用搜索
```

## 目录结构 (SUMMARY.md)

SUMMARY.md 定义文档的目录结构：

```markdown
# Summary

[介绍](README.md)

# 章节 1

- [子章节 1.1](chapter1/section1.md)
- [子章节 1.2](chapter1/section2.md)

# 章节 2

- [子章节 2.1](chapter2/section1.md)
```

## 常用命令

### 初始化新书籍

```bash
mdbook init my-book
cd my-book
mdbook serve
```

### 清理构建

```bash
mdbook clean
```

### 测试

```bash
mdbook test
```

### 监听文件变化自动重建

```bash
mdbook watch
```

## 自定义

### 主题

mdBook 支持多种主题：
- light (亮色)
- rust (Rust 风格)
- coal (深色)
- navy (海军蓝)
- ayu (护眼模式)

在浏览器中可以切换主题。

### 自定义 CSS

在 `book.toml` 中添加：

```toml
[output.html]
additional-css = ["custom.css"]
```

### 自定义 JavaScript

```toml
[output.html]
additional-js = ["custom.js"]
```

## 进阶功能

### 1. 代码高亮

支持多种编程语言语法高亮：

````markdown
```cpp
int main() {
    return 0;
}
```
````

### 2. 隐藏代码行

```rust,ignore
// 这行代码不会显示
# println!("隐藏的代码");
println!("显示的代码");
```

### 3. 数学公式（需要插件）

安装 mdbook-katex：
```bash
cargo install mdbook-katex
```

配置：
```toml
[preprocessor.katex]
```

使用：
```markdown
$$ E = mc^2 $$
```

### 4. Mermaid 图表（需要插件）

安装 mdbook-mermaid：
```bash
cargo install mdbook-mermaid
```

## 持续集成

### GitHub Actions

创建 `.github/workflows/docs.yml`:

```yaml
name: Deploy Docs

on:
  push:
    branches: [ master ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Setup mdBook
        uses: peaceiris/actions-mdbook@v1
        with:
          mdbook-version: 'latest'

      - name: Build
        run: cd doc && mdbook build

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./doc/book
```

## 故障排查

### 问题 1: SUMMARY.md 链接失效

**原因**: 文件路径不正确

**解决**: 确保 SUMMARY.md 中的路径相对于 `src` 目录（在我们的配置中是 `doc/` 目录）

### 问题 2: 中文搜索不工作

**原因**: 默认搜索不支持中文分词

**解决**: 使用 mdbook-i18n-helpers 或调整搜索配置

### 问题 3: 构建失败

**原因**: Markdown 语法错误或文件缺失

**解决**: 检查日志，修复错误

## 参考资源

- [mdBook 官方文档](https://rust-lang.github.io/mdBook/)
- [mdBook GitHub](https://github.com/rust-lang/mdBook)
- [Markdown 语法](https://www.markdownguide.org/)

---
作者: @scorpiostudio @yutiansut
