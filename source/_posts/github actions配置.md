---
title: Github Actions Config
date: 2021-1026-14 11:15:36
tags: ["CICD"]
---


1. 生成ssh-key

2. 在 blog 仓库 Settings -> Secrets -> Add a new secret 页面上添加生成的私钥

3. 在 your.github.io 仓库 Settings -> Deploy keys -> Add deploy key 页面上添加生成的公钥, 并勾选 Allow write access 选项

4. 编写 Github Actions
    - 在 blog 仓库根目录下创建 .github/workflows/deploy.yml 文件

```
name: CI
on:
  push:
    branches:
      - hexo

env:
  GIT_USER: Pepsi33
  GIT_EMAIL: xxx@xx.com

jobs:
  build:
    name: Build on node ${{ matrix.node_version }} and ${{ matrix.os }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-latest]
        node_version: [12.x]

    steps:
      - name: Checkout source
        uses: actions/checkout@v1
        with:
          ref: hexo

      - name: Use Node.js ${{ matrix.node_version }}
        uses: actions/setup-node@v1
        with:
          version: ${{ matrix.node_version }}

      - name: Configuration environment
        env:
          ACTION_DEPLOY_KEY: ${{ secrets.HEXO_DEPLOY_KEY }}
        run: |
          sudo timedatectl set-timezone "Asia/Shanghai"
          mkdir -p ~/.ssh/
          echo "$ACTION_DEPLOY_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          git config --global user.name $GIT_USER
          git config --global user.email $GIT_EMAIL

      - name: Install dependencies
        run: |
          npm install hexo-cli -g
          npm install

      - name: Hexo deploy
        run: |
          hexo clean
          hexo d
```
