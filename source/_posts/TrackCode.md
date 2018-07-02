---
title: TrackCode 使用教程
date: 2018-6-11 22:45:40
tags:
toc_number: true
---

## 介绍
最近看项目代码，需要画时序图帮助梳理，为了偷懒，顺手撸了一个小插件，大家有兴趣可以试试。
TrackCode可以快速是一个帮你快速生成时序图的工具

支持 IntelliJ 旗下所有IDE软
* IntelliJ IDEA
* PhpStorm  
* WebStorm  
* PyCharm  
* RubyMine  
* CLion  
* GoLand  
* DataGrip  
* Android Studio

理论以上都支持，但是楼主只测了IDEA 和 Studio

简单说就是你按快捷键，TrackCode会帮你记录类和函数名 按照特定格式输出
这种格式可以直接在Markdown中变成时序图
具体请参见如下演示（点击图片可放大观看）

![GIF.gif](https://upload-images.jianshu.io/upload_images/1967257-e651075d2f86006a.gif?imageMogr2/auto-orient/strip)

## 安装
打开插件界面 搜索 TrackCode 安装即可
![image.png](https://upload-images.jianshu.io/upload_images/1967257-160188d605158217.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后重启Android Studio

## 使用
打开任意项目工程
在开始类中鼠标右键 就可以看到TrackCode插件
> * Track Code 用来记录类和函数
> * Write Note 用来写note

![image.png](https://upload-images.jianshu.io/upload_images/1967257-b5683bdd74b4baaf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


比如我们记录A类调用B类test函数的流程，并画出流程图
* 记录时序图起始点：首先我们打开A类 在任意地方 点击右键Track Code 
* 跳转函数：然后我们点击test方法跳转到B类的实现位置
* 记录终止点：**将光标放在ok方法上** 然后点击右键Track Code

就完成了记录

记录完成后 将项目目录方式切换到 project

![image.png](https://upload-images.jianshu.io/upload_images/1967257-c7d08ddb65998f2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就能看到生成了一个 track 的目录 里面有一个 track.md文件
这就是插件生成的文件

## 生成流程图
打开track.md 文件
复制全部
打开 [markdown软件](https://maxiang.io/)
清空所有内容
将内容粘贴进下图所示的光标区域

![image.png](https://upload-images.jianshu.io/upload_images/1967257-074b19f6e9c7cb2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后流程图就生成了

## 导出流程图
最右边有个大象的图标
点击里面有导出
当然直接截图也行

![image.png](https://upload-images.jianshu.io/upload_images/1967257-771bc5f3e16eacae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 添加函数解释 Note
Write Note选项是用来添加Note使用的
note会作为时序图的函数注解出现
具体使用方法是
当我们记录完终止点后，就会发现Write Note功能可以使用了，点击该功能，会出现弹框，输入注解内容点ok即可


有好的建议和反馈请联系：karlpu