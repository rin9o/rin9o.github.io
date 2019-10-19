---
title: 从零在 GitHub Pages 搭建一个免费 Hexo 博客并配置自动构建
date: 2019-10-19 18:00:00
tags:
- 博客
- Hexo
- GitHub
---

简单了解了一下现有的自建博客，主流是在自己的服务器或者 PaaS 平台中搭建运行 WordPress、Typecho 等 PHP 博客
系统。前几年前端开发火起来后，又有使用 Node.js 等技术开发各类博客系统，其中静态博客系统因为其不依赖
数据库、执行环境，有响应快、成本低且较难被侵入的优势，还可以在免费的页面托管服务上搭建博客。

最终我选择从零开始在 GitHub Pages 托管服务上搭建一个 Hexo 博客系统，在进行搭建开始前，你需要先准备的工作：

1. 拥有自己的 GitHub 帐号；
2. 安装 Node.js 和 NPM（也可以使用其它包管理，本文以 NPM 为例）；
3. 安装 Git
4. 创建一个名为 `[自己的 GitHub 名字].github.io` 的仓库。

## 安装 Hexo

打开终端，运行下列命令安装 Hexo 命令行工具：

```bash
npm i -g hexo-cli
```

## 新建 Hexo 博客

在自己存放代码的目录中执行新建命令：

```bash
hexo init [博客文件夹名]
```

例如我输入的是 `ringoblog` ，执行完毕后可以 `cd ringoblog` 进入刚刚创建好的博客文件夹，可以看到 Hexo 自动生成了很多文件和文件夹用来存放博客配置和页面数据：

```text
.
├── _config.yml  - 博客配置
├── package.json - Hexo 依赖配置
├── scaffolds    - 模版文件夹
├── source       - 资源文件夹，如文章、媒体等
|   ├── _drafts  - 草稿
|   └── _posts   - 文章
└── themes       - 主题资源
```

> 了解详情可跳转至 Hexo 官方文档查看：[hexo.io/zh-cn/docs/setup](https://hexo.io/zh-cn/docs/setup)

完成新建之后，在 `_config.yml` 修改博客一些必要的基本参数：

1. `title` : 博客标题
2. `author` : 博客作者名称
3. `language` : 语言，支持的语言取决于当前使用的主题
4. `timezone` : 时区，中国可以填 `Asia/Shanghai`
5. `url` : 博客地址，本例使用 GitHub Pages 应填 `[自己的 GitHub 名字].github.io`

要想看到预览效果，可以在博客项目目录内执行下列命令启动 Hexo 演示服务器：

```bash
hexo server
```

也可以生成静态文件然后用浏览器直接打开预览：

```bash
hexo generate
```

## 第一次的手动部署

完成一次 GitHub Pages 页面部署，你需要安装 Git 并掌握一些基本使用方法。

在博客目录下进行初始化：

```bash
git init
```

但不要立即将文件提交到 `master` 分支，因为 GitHub 要求用户页面的仓库（`[用户名].github.io`） 只能以 `master` 分支作为 Pages 页面存放的分支。

我们的代码在生成前只是一些仅对开发者可读的文档、配置，没有 index.html 入口，我们可以命名一个新分支为 `raw` （自由选择），在上面完成初始化后执行下行代码创建新分支：

```bash
git checkout -b raw
```

随后将我们的内容提交到 `raw` 分支并 Push 到在线仓库：

```bash
# Commit all files
git add -A && git commit -m "第一次提交"
# Add remote repo
git remote add origin https://github.com/[用户名]/[用户名].github.io.git
# Push changes to remote
git push origin raw
```

此时因为 `master` 分支还是空的，我推荐使用 Hexo 的 Git 部署插件来完成 “生成-复制-提交” 这一系列部署步骤。

首先通过 NPM 安装插件：

```bash
npm i hexo-deployer-git
```

随后打开项目根目录下的 `_config.yml` 来配置我们的部署选项，默认生成的配置文件有 `deploy` 的键，注意不要写重复了：

```yaml
# Deployment
deploy:
  type: git
  repo: git@github.com:[用户名]/[用户名].github.io.git
  branch: master
  message: Deploy generated pages
```

这一段的意思就是，执行 `hexo deploy` 命令时，将会使用 `hexo-deployer-git` 插件来部署生成好的博客页面到指定的 Git 仓库，并且推送到 `master` 分支中， `repo` 为自己的实际仓库地址。

使用 SSH 方式推送需要事先配置好 SSH 密钥，具体不多介绍。

一切就绪之后，执行下列代码进行一键部署：

```bash
hexo generate && hexo deploy
```

终端提示成功后，查看我们远端的 `master` 分支是否已经有页面文件，然后打开 `[用户名].github.io` 来查看效果。

现在我的这个博客就是通过本文教程步骤完成搭建：[rin9o.github.io](https://rin9o.github.io)

## 配置 GitHub Actions 完成自动部署

GitHub Actions 也是一种免费的 CI/CD 服务，对开源项目是完全免费，测试期间私有项目免费，这篇文章最后一次更新时还在 Beta 阶段，第一次使用要在 [github.com/features/actions](https://github.com/features/actions) 申请加入 Beta 计划。

GitHub Actions 允许开发者将自己写好的脚本发布到公共 Marketplace 中，方便其他人使用，GitHub 官方也提供了一些 Actions，源码存放在：[github.com/actions](https://github.com/actions)

首先，我们在自己的博客目录下创建 `.github/workflow` 目录，然后在该目录下创建一个文件命名为 `deploy.yml` 。

复制以下内容粘贴到里面：

```yaml
name: Deploy GitHub Pages on push event

on:
  push:
    branches:
      - raw

jobs:
  build:

    runs-on: ubuntu-18.04

    steps:
    - uses: actions/checkout@v1
    - name: Set up Node.js
      uses: actions/setup-node@v1
      with:
        node-version: '10.x'
    - name: Set up dependecies
      run: |
        npm i -g hexo-cli
        npm i
    - name: Prepare build environment
      env:
        GH_ACTION_DEPLOY_KEY: ${{secrets.GH_ACTION_DEPLOY_KEY}}
      run: |
        mkdir -p ~/.ssh/
        echo "$GH_ACTION_DEPLOY_KEY" > ~/.ssh/id_rsa
        head ~/.ssh/id_rsa -n 2
        chmod 600 ~/.ssh/id_rsa
        ssh-keyscan github.com >> ~/.ssh/known_hosts
        git config --global user.name 'Ringo (替换成你的 Git 提交名字)'
        git config --global user.email 'ringo@example.com (替换成你的 Git 提交邮箱)'
    - name: Deploy to GitHub Pages
      run: |
        hexo generate && hexo deploy
```

简单解释一下这段 YAML 的意思是：

- 当 `raw` 分支有 Push 操作时，开始执行这个 Action，包含一个 build 工作
  - 这个 build 工作在 `ubuntu-18.04` 系统环境下执行，包含以下步骤
    1. 使用 GitHub 中的 `actions/checkout@v1` 在环境中 Checkout Git 默认分支的代码
    2. 使用 GitHub 中的 `actions/setup-node@v1` 初始化 `10.x` 版本的 Node.js
    3. 从 NPM 安装依赖
    4. 准备构建环境，从仓库 Secret 中获取 `GH_ACTION_DEPLOY_KEY` 作为环境变量，将部署用的 SSH 私钥保存在运行环境中以便后续 Commit 用。
    5. 执行 Hexo 命令行工具生成并部署到 Git，这一步执行等于我们前面的手动部署

检查是否需要根据自己需求修改 Action 内容，提交改动之后，先别急着 Push 到 GitHub 上，我们还有一些工作要做：

上面的 Action 流程依赖了仓库 Secret 中的 `GH_ACTION_DEPLOY_KEY` 键来获取 SSH 密钥内容，我们需要将拥有对博客远端仓库拥有写入权限的 SSH 私钥填到仓库 Settings - Secrets 中，点击“Add a new secret”添加一个名为 `GH_ACTION_DEPLOY_KEY` 的数据，然后将私钥数据粘贴到 Value 中保存。

因为 Action 运行环境是由 GitHub 提供，而且第三方的脚本可能存在风险，这一步我建议新建一个专门用于提交博客仓库的 SSH 密钥而不是使用帐户在用的密钥，创建好新的密钥后，将公钥添加到仓库 Settings - Deploy keys 中（记得勾上 Push 权限）。

添加完密钥后，就可以 Push 这一次在 `raw` 分支上添加 `GitHub Actions Workflow` 的改动了。

完成 Push 后，你应该就能在自己仓库的 Actions 页看到自动执行的 Workflow 记录。

例如我的博客执行记录：[Actions - rin9o.github.io](https://github.com/rin9o/rin9o.github.io/actions)

## 总结

Hexo 博客搭建起来还是比较简单的，编写文章需要学习一下 Markdown 语法，配合一个带预览甚至是可见即可得的编辑器，体验也是非常不错的，只是如果在手机上编写提交文章的话配置 Git 和 Node.js 环境会比较麻烦。

通过 CI 服务自动构建 Hexo 博客的静态页面已经减少了每次提交文章时部署的麻烦，正好 GitHub Actions 语法比较简洁而且也是对开源项目免费，看了一下执行记录平均时长只需 45 秒，提交源码后很快就能查看到在线效果。

另外感谢一位不愿透露姓名的童鞋帮我检查了代码上的笔误，自己花了很多时间没看出来的问题给他一眼就看出来了~
