---
title: 使用github Action自动化部署 Hugo
date: 2019-12-04T23:58:12+08:00
lastmod: 2019-12-04T23:58:12+08:00
categories: ["部署"]
description:
---
### 前言
最近由于一直在用的`travis-ci`出现了迷之bug,加上想尝试一下`github action`就决定尝试用`github action`替换`travis-ci`

### 选用action
`github`现有的`action`组件可以在[这里](https://github.com/marketplace?type=actions)查看. 如果想要自定义自己仓库的`workflow`,可以选用里面来进行组合.
不过`hugo`在`github page`的`workflow`,已经有人在[这](https://github.com/peaceiris/actions-hugo)弄好了,

### 编写workflow
`workflow`定义是一个位于`.github/workflows/`的`yaml`文件.点开仓库的`action`按钮会出现引导界面
![2019-12-05 00-13-38 的屏幕截图.png](https://i.loli.net/2019/12/05/iIcU3VN4evzZfl6.png)
显示一些常用的`workflow` 但由于我们要用并不在这里面 点击右上角的`skip`就行.

模板文件
```yaml
name: github pages

on:
  push:
    branches:
    # 你的hugo源码分支
    - master

jobs:
  build-deploy:
    runs-on: ubuntu-18.04
    steps:
    - uses: actions/checkout@v1
      # 如果使用了git submodules
      # with:
      #   submodules: true

    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2
      with:
        hugo-version: '0.59.1'
        # extended: true

    - name: Build
      run: hugo --minify

    - name: Deploy
      uses: peaceiris/actions-gh-pages@v2.5.0
      env:
        ACTIONS_DEPLOY_KEY: ${{ secrets.ACTIONS_DEPLOY_KEY}}
        # 你的github page分支
        PUBLISH_BRANCH: gh-pages
        PUBLISH_DIR: ./public
```


### 设置secrets
之后需要设置`secrets`.`secrets.ACTIONS_DEPLOY_KEY`实际上是你的`github Personal access tokens`,申请后点击项目的`setting`,在`secrets`栏添加一个key名是`PERSONAL_TOKEN`的`secrets`即可


