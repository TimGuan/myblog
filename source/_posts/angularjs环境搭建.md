---
title: angularjs环境搭建
date: 2017-03-24 12:06:14
categories: 
- 编程
- angularjs
tags:
- angularjs
---
angularjs干神马的就不赘述了，这里简单介绍介绍怎么样搭建一个angularjs的webapp的基本过程。
# 环境准备:
* 1.nodejs安装
开发前端项目nodejs是必备的，目前推荐安装的是nodejs4.4.7，传送门https://nodejs.org/en/
因为墙的原因安装好nodejs后最好安装一下cnpm，这是淘宝的一个npm镜像源，能更方便的进行插件安装

``` shell
npm install -g cnpm --registry=https://registry.npm.taobao.org
```
<!-- more -->
* 2.grunt安装
grunt是前端项目中的常用项目构建工具，类似我们后端开发常用的maven（除项目包含依赖管理），借助grunt可以完成诸如混淆、打包、测试等过程。

``` shell
cnpm install -g grunt-cli
```
* 3.bower安装
bower的作用类似maven的依赖管理，用来管理项目中的前端组件库依赖

``` shell
cnpm install -g bower
```
* 4.yoeman安装
yoeman是个前端的脚手架工具，能够快速的生成一个前端的项目结构

``` shell
cnpm install -g yo
```
* 5.generator-angular
yoeman搭配generator-angular即可生成一个angularjs的webapp

``` shell
cnpm install -g generator-angular
```

# 使用yo生成angularjs webapp
完成以上的环境准备后，就可以动手搭建第一个angular的webapp了，具体步骤如下

``` shell
mkdir demo 
cd demo
yo angular demo
```
demo即app的名称，执行成功后会生成如下的一个项目结构
{% asset_img img1.png %}
app目录即我们的工程开发目录，scripts用来放controller、service等业务代码，views内放置我们的页面模板，index.html作为项目的访问根，描述了这个项目的资源引用和提供给grunt合并打包的依据。
bower.json类似于我们maven中dependency的作用，管理了诸多项目依赖，在变更了依赖后需要执行bower install来更新依赖，对应代码会下载到bower_components中。
package.json类似于maven中的plugin，管理了一些开发工具的依赖项，在变更后需要执行cnpm install，对应代码会下载到node_modules中，和bower的区别是这个地方的依赖非工程内依赖，也就是说前端的代码内不会引用到node_modules中的js库。
Gruntfile.js这个就描述了压缩、编译、打包、测试等流程，类似maven的生命流。

tips:推荐安装
wnddel因为node的目录路径很长，有时候windows下删除不了，这个node插件能够方便的删除这类目录
https://www.npmjs.com/package/windows-node-deps-deleter#readme
nproxy国人开发的一个代理服务，方便本地调试
https://github.com/goddyZhao/nproxy

简单的介绍了一下angularjs的项目搭建，是不是和后端的开发流程很像？哈哈~~~