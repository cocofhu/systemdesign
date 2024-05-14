---
title: 使用GitHub+Hexo部署博客
date: 2024-05-14 23:49:34
---

> 准备部署一套全新的博客界面来记录一些记录一些有趣解决方案和精妙的系统设计，与之前的的算法博客区分出来，记录一下搭建过程。

### 一、基本原理
编写`Markdown`文件，提交到GitHub，使用GitHub提供的流水线将`Markdown`文件编译成静态`HTML`文件，使用`GitHub`提供的网站部署服务部署这些静态文件。

最终的效果就是，在本地写好文档后，提交到`GitHub`后，自动更新网站数据。

### 二、部署流程
由于`Hexo`是很早以前部署的，我已经忘记了，我也不感兴趣这里的环境搭建，所以这里偷了个懒直接复制了我之前的项目。

#### 1、项目准备
需要完成上述功能，需要在`GitHub`创建如下两个项目(根据情况改名字)：
- 1、cocofhu/systemdesign.github.io 这个项目用来存部署的静态页面
- 2、cocofhu/systemdesign 这个项目用来存原始markdown文件(可以直接复制本项目)

#### 2、创建密钥
```shell
ssh-keygen -f github-deploy-key
```
一直回车直至成功得到公钥和私钥。

#### 3、配置密钥

1、在 [cocofhu/systemdesign.github.io](https://github.com/cocofhu/systemdesign.github.io/settings/keys)的`Deploy keys`配置公钥,`Title`用 `HEXO_DEPLOY_PUB`。

2、在 [cocofhu/systemdesign](https://github.com/cocofhu/systemdesign/settings/secrets/actions) 下`New repository secret`配置私钥,`Title`用 `HEXO_DEPLOY_PRI`。

#### 4、配置CI
更改复制的项目[cocofhu/systemdesign](https://github.com/cocofhu/systemdesign/blob/main/.github/workflows/main.yml)的`.github/workflows/main.yml`:
```yaml
name: HEXO CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [15.x]

    steps:
      - uses: actions/checkout@v1

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node-version }}

      - name: Configuration environment
        env:
          HEXO_DEPLOY_PRI: ${{secrets.HEXO_DEPLOY_PRI}}
        run: |
          mkdir -p ~/.ssh/
          echo "$HEXO_DEPLOY_PRI" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          # 改成自己的
          git config --global user.name "cocofhu"
          git config --global user.email "cocofhu@outlook.com"

      - name: Install dependencies
        run: |
          npm i -g hexo-cli
          npm i

      - name: Deploy hexo
        run: |
          cd blog
          rm -rf node_modules && npm install --force
          npm install --save hexo-deployer-git
          hexo g
          # 自定义域名 配置一个CName指向systemdesign.github.io
          echo "sollution.cocofhu.com" > public/CNAME
          hexo d

```

#### 5、修改配置
更改复制的项目[cocofhu/systemdesign](https://github.com/cocofhu/systemdesign/blob/main/blog/_config.yml)的`blog/_config.yml`:
```yaml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  #改成字节的部署仓库
  repo: git@github.com:cocofhu/systemdesign.github.io
  branch: master
```

最后在[这里](https://github.com/cocofhu/systemdesign.github.io/settings/pages)配置GitHub Pages就大功告成了！








