---
title: hexo搭建github博客多设备更新
date: 2018-06-02 20:01:32
categories:
- hexo
tags:
- hexo
copyright: true
top:
comments: true
---


前段时间由于电脑重新安装了系统,导致自己的博客文件丢失,而github上面的只有发布后的文件,没有博客的源文件。想要恢复到源文件至今还没有找到解决方法,好在原来的博客内容不多,丢失了也无所谓了。但是以后要是换电脑了,这种问题怎么解决呢?今天介绍一个解决方案。
其实方法很简单,首先应该有两个分支,一个分支用来保存发布后的内容,一个分支用来保存源文件.这里用master分支保存发布后的内容,hexo分支保存源文件内容.

## 第一步
设备A上搭建github博客
## 第二步
在博客根目录下,打开_config.yml,添加如下内容
```yml
deploy:
  type: git
  repository: git@github.com:youname/youname.github.io.git
  branch: master
```
> 可以看到现在已经有一个master分支,但是这时候个人博客还不是一个git目录

提交代码
```
// hexo编译源文件，生成静态文件，也可以分开执行
hexo clean && hexo g && hexo d
```
> hexo clean: 清空博客缓存
> hexo g(hexo generator 的简写):生成静态文件
> hexo d(hexo deploy的简写): 部署文件,这条命令会使用第一步中的配置信息进行部署

现在打开yourname.github.io就能够看到你的博客了
## 第三步
因为我用的是next主题(其他主题做相似处理).删除next文件下的.git文件夹,这是因为我们在要我们的博客下创建.git,如果子目录下也有.git,会有问题.然后执行以下命令
```
// git初始化
git init
// 新建分支并切换到新建的分支
git checkout -b 分支名
// 添加所有本地文件到git
git add .
// git提交
git commit -m "提交说明"
// 文件推送到hexo分支
git push origin hexo
```
> 以后操作都是在hexo分支中,当我们修改了我们的博客内容时,先执行hexo clean && hexo g && hexo d ,这个命令用来将本地博客发布到github上
> 然后在将本地内容提交到hexo分支中

## 第四步
假设我们需要在电脑B上搭建我们之前的博客内容.我们需要拉取我们的hexo分支,因为hexo分支是源文件,master可以不用拉取
```
// 克隆分支到本地
git clone -b hexo https://github.com/用户名/仓库名.git
// 进入博客文件夹
cd youname.github.io
// 安装依赖
npm install
```
## 第五步
电脑B上编辑博客内容,静态文件提交到master分支,源文件提交到hexo分支
```
//博文提交到master上面。
hexo clean && hexo g && hexo d
//源文件提交到hexo分支上面。
// 添加源文件
git add .
// git提交
git commit -m ""
// 先拉原来Github分支上的源文件到本地，进行合并
git pull origin hexo
// 比较解决前后版本冲突后，push源文件到Github的分支
git push origin hexo
```
## 第六步
在电脑A上可以同步hexo分支,开始更新博客

**注意: 以后操作都是在hexo分支中,当我们修改了我们的博客内容时,先执行hexo clean && hexo g && hexo d ,这个命令用来将本地博客发布到github上
然后在将本地内容提交到hexo分支中**
