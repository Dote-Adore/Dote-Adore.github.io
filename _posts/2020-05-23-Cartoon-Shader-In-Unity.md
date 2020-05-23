---
date: 2020-05-23 22:50:00 +0800
layout: post
title: 卡通渲染该如何实现~
subtitle: 使用unity制作一个简单的卡通渲染shader~
description: 《动物森友会》中拱形地形实现原理
category: opengl
image: >-
    https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/Zelda_Mifa.png
tags:
  - unity
  - shader
  - 图形学
  - 游戏设计技巧
paginate: true
---
卡通渲染一直收到诸多游戏公司的青睐，
也有非常多质量高的作品选用了卡通渲染风格作为游戏画风，
并及其成功，其中著名的有《女神异闻录》《塞尔达传说》
《崩坏3》等游戏作品；在如今众多的写实风格、堆砌各种光照特效的时代，
使用一个特别的卡通渲染会将你的作品从之间脱颖而出；
下面，跟随我一起使用unity制作一个简单的卡通渲染风格吧~

## 分析
常见的卡通渲染会将阴影、高光和漫反射光线有非常明显的界限，两种颜色之间不会进行平滑过度，
![基于物理的渲染和卡通渲染]({{site.url}}/assets/img/posts/BPRAndNPR.png)



