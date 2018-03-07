### 1. 下载安装hexo
```sh
npm install -g hexo-cli
npm install hexo-server -save
```
### 初始化博客
```sh
// 建立一个博客文件夹，并初始化博客，<folder>为文件夹的名称，可以随便起名字
$ hexo init <folder>
// 进入博客文件夹，<folder>为文件夹的名称
$ cd <folder>
// node.js的命令，根据博客既定的dependencies配置安装所有的依赖包
$ npm install
```

### 配置博客
基于上一步，我们对博客修改相应的配置，我们用到_config.yml文件：
+ 1. 修改网站相关信息
```
title: inerdstack
subtitle: the stack of it nerds
description: start from zero
author: inerdstack
language: zh-CN
timezone: Asia/Shanghai
```
language和timezone都是有输入规范的，详细可参考语言规范和时区规范。

注意：每一项的填写，其:后面都要保留一个空格，下同。

+ 2. 配置统一资源定位符（个人域名）
```
url: http://xxx.com
```
对于root（根目录）、permalink（永久链接）、permalink_defaults（默认永久链接）等其他信息保持默认。

+ 3. 配置部署(建议升级到最新的git客户端，否则会出现无法认证)
```
deploy:
  type: git
  repo: https://github.com/xxx/xxx.github.io.git
  branch: master
```

### 发表一篇文章
在终端输入：
```sh
hexo new "文章标题"
```
### 本地发布
```sh
hexo server
```
### 部署( hexo generate hexo deploy)
```sh
hexo g
hexo d
```