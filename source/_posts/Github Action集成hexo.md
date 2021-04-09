---
title: Github Action集成Hexo
date: 2021-04-09 00:00:00
categories: 
- CI
tags:
- CI
---



以前博客的CI是使用travis-ci.org，最近发现需要转移到https://www.travis-ci.com/，并且只有一定的免费限额，后续可能要购买次数，于是准备找一个替代方案。

我的博客主要有两个分支，source和master，source用于维护文章，主题，配置等信息，master用于存储博客的静态页面。只要有source的代码，随时可以生成master页面。

上网搜索发下，Github已经提供了免费的CI工具，Github Action，这里记录下所需的改动。

<!--more-->

## CI流程

在source分支下，创建.github/workflows/deploy.yml文件，内容如下，这里有许多无用的配置，没有删除。

```yml
name: CI

on:
  push:
    branches:
      - source

env:
  GIT_USER: abely
  GIT_EMAIL: abely_liu@sina.com
  THEME_REPO: sanonz/hexo-theme-concise
  THEME_BRANCH: master
  DEPLOY_REPO: sanonz/sanonz.github.io
  DEPLOY_BRANCH: master

jobs:
  build:
    name: Build on node ${{ matrix.node_version }} and ${{ matrix.os }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-latest]
        node_version: [15.x]

    steps:
      - name: Checkout
        uses: actions/checkout@v2


      - name: Use Node.js ${{ matrix.node_version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node_version }}

      - name: cache
        uses: actions/cache@v1
        id: cache-dependencies
        with:
          path: node_modules
          key: ${{runner.OS}}-${{hashFiles('package-lock.json')}}    

      - name: Install Dependencies 
        if: steps.cache.outputs.cache-hit != 'true' 
        run: |
          npm install
          npm install hexo-renderer-scss hexo-renderer-swig --save

      - name: Deploy hexo
        env:
          HEXO_DEPLOY_PRI: ${{ secrets.GH_TOKEN }}
          REPO: github.com/abelyliu/abelyliu.github.io.git
        run: |
          npx hexo g
          cd ./public
          git init
          git config --global  user.name "abely"
          git config --global  user.email "abely_liu@sina.com"
          git add .
          git commit -m "Update docs"
          git push --force --quiet "https://${{ secrets.GH_TOKEN }}@github.com/abelyliu/abelyliu.github.io.git" master:master
```

接下来就是创建配置中需要的GH_TOKEN，在个人设置里的Developer settings。

![image-20210409092750375](http://blog.abely.store/1617931670405-image-20210409092750375.png)

创建完毕，配置到项目中

![image-20210409093020581](http://blog.abely.store/1617931820608-image-20210409093020581.png)

这个时候对source分支做出改动，都会触发CI流程，如下

![image-20210409093135040](http://blog.abely.store/1617931895086-image-20210409093135040.png)

## 自定义域名

自定义域名其实以前配置过，中间博客清理过一次，现在无法绑定原来的域名，说已被占用。这种情况需要邮件联系，手动解除，不过因为域名快到期，续费又很贵，决定买个新的域名。

域名配置也比较简单，在source目录下，创建一个CNAME文件，内容填上域名，我的abely.cn。

在DNS解析的地方配置一个CNAME类型，指向abelyliu.github.io即可。

![image-20210409093724602](http://blog.abely.store/1617932244642-image-20210409093724602.png)

## 图床

图床使用的使阿里云的oss，上传使用picgo，配置了一个压缩插件compression。