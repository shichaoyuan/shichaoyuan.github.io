---
layout: post
title: 如何快速开始一个小项目 —— playframework
date: 2015-08-06 23:41:21
tags: engineering
---

##1

基本文档 [Play 2.4.x documentation](https://www.playframework.com/documentation/2.4.x/Home)

##2

生成框架 [Creating a new application](https://www.playframework.com/documentation/2.4.x/NewApplication)

导入Eclipse [Setting up your preferred IDE](https://www.playframework.com/documentation/2.4.x/IDE#Eclipse)

**NOTE** 目前`Scala IDE`的4.1.1版本的play plugin有问题，可以暂时自行编译安装[https://github.com/cweinreben/scala-ide-play2](https://github.com/cweinreben/scala-ide-play2)这个fork。

##3

连接数据库

[**Slick**](http://slick.typesafe.com/)

用于连接MySQL，详见文档[PlaySlick](https://www.playframework.com/documentation/2.4.x/PlaySlick)

[**ReactiveMongo**](http://reactivemongo.org/) 

用于连接MongoDB，详见文档[Reactive Scala Driver for MongoDB](http://reactivemongo.org/releases/0.11/documentation/tutorial/play2.html)

##4

Log

基本上就是配置logback

[Configuring logging](https://www.playframework.com/documentation/2.4.x/SettingsLogger)

[The Logging API](https://www.playframework.com/documentation/2.4.x/ScalaLogging)

文档中还很贴心的给出了两种实现accesslog的方法。

##5

部署服务

[Deploying your application](https://www.playframework.com/documentation/2.4.x/Production)
