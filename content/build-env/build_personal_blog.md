+++
date = '2026-04-23T08:56:18+08:00'
draft = false 
title = 'Build_personal_blog'
+++



# Hugo + Github Pages搭建个人博客

## 创建Github pages repo
- 进入到github个人主页，在右上角点击“+”，选择New repository，如下图所示
![create github pages repo 1](/images/build_personal_blog/create_github_pages_repo_1.png "新建repo")

- 在General配置栏中填写Repository name，格式是username.github.io，***其中username必须是自己的github用户名***，例如我的github用户名是phoon0416，则需要填phoon0416.github.io，如下图所示
![create github pages repo 2](/images/build_personal_blog/create_github_pages_repo_2.png "配置repo")

- 把页面拉到最底部，点击Create repository完成github pages repo的创建，如下图所示
![create github pages repo 3](/images/build_personal_blog/create_github_pages_repo_3.png "完成创建repo")

- 配置github pages为Actions方案
    - 进入到该repo的Settings菜单，如下图所示
      ![create_github_pages_repo_4](/images/build_personal_blog/create_github_pages_repo_4.png)
    - 在Code and automation一栏，选择Pages，如下图所示
      ![create_github_pages_repo_5](/images/build_personal_blog/create_github_pages_repo_5.png)
    - Build and deployment配置栏中的source设置项设置为GitHub Actions，如下图所示
      ![create_github_pages_repo_6](/images/build_personal_blog/create_github_pages_repo_6.png)

## 安装hugo并创建文档
- 安装hugo

***如果你的系统是ubuntu22.04，并且使用apt安装默认包，那么很多新的主题并不能支持，在这里我们选择hugo_extended_0.160.0_linux-amd64.deb版本***

```bash
sudo dpkg -i hugo_extended_0.160.0_linux-amd64.deb
```

- 创建新站点

```bash
hugo new site my-blog
cd my-blog
```

- 添加主题

Hugo本身不带主题，你需要去[Hugo Themes](https://themes.gohugo.io/)选一个，这里以PaperMod为例:

```bash
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

- 修改配置文件

```bash
vi hugo.toml ---> baseURL修改成baseURL = 'https://username.github.io/'（username是你的github用户名）
             ---> locale修改成locale = 'zh-cn'
             ---> 增加主题配置theme = 'PaperMod'（PaperMod是你安装的主题名）
```

- 创建文章

```bash
hugo new build-env/build_personal_blog.md
cd content/build-env
vi build_personal_blog.md ---> draft修改成draft = false
```

## 部署到Github（Actions方案）

- 创建Github Actions自动化脚本

在my-blog根目录下，创建目录结构.github/workflows/hugo.yml，在hugo.yml中写入以下配置

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true
      - name: Build
        run: hugo --minify
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- 推送代码到github

```bash
cd my-blog
git add .
git commit -m "Initial Commit"
git branch -M main
git remote add origin https://github.com/username/username.github.io.git（username是你的github用户名）
git push -u origin main
```

## 测试站点
部署到github之后，***等待两分钟让github编译生成静态网页***，打开chrome浏览器，***为了防止浏览器缓存的干扰，用Ctrl+Shift+N打开无痕浏览模式***，输入https://username.github.io（username是你github用户名）回车，能正常显示网页说明部署成功，如下图所示
![test_site_1](/images/build_personal_blog/test_site_1.png)
