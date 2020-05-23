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
常见的卡通渲染会将阴影、高光和漫反射光线有非常明显的界限，两种颜色之间不会进行平滑过度。
![基于物理的渲染和卡通渲染]({{site.url}}/assets/img/posts/BPRAndNPR.png)
当然，我们还可以增加一个卡通描边：
![描边]({{site.url}}/assets/img/posts/outLine.png)
将这两者结合就可以实现一个比较基础的卡通渲染风格！<br>
接下来进行原理讲解：
## 实现
### 参数设置
首先，我们需要三个色块，分别对应漫反射颜色，高光颜色和阴影颜色，同时我们也需要一个光滑度来控制高光的范围。<br>
各个参数设置如下：
```js
    Properties
    {
        //基础颜色
         _Color("Color", Color) = (1,1,1,1)
	// 自定义贴图
         _MainTex("Albedo (RGB)", 2D) = "white" {}
	// 阴影的颜色
        _ColorShadow("Color Shadow", Color) = (1,1,1,1)
	// 高光度
        _Glossiness("Smoothness", Range(0,128)) = 32
	// 高光颜色
        _GlossinessColor("Glossiness Color",Color) = (1,1,1,1)
        //描边强度（粗细）
        _OutineStrength("Outline strength",Range(0,1)) = 0.1
    }  
    //......
    // 常用的两个结构体
    CGPROGRAM
    // ......
     struct a2v
                {
                    float4 vertex : POSITION;
                    fixed3 normal : NORMAL;
                    float4 texcoord : TEXCOORD0;
                };
                struct v2f
                {
                    float4 pos : SV_POSITION;
                    fixed3 worldNormal:TEXCOORD0;
                    // 存储纹理坐标
                    float2 uv:TEXCOORD2;
                    SHADOW_COORDS(1)
                };
```
### 阴影
如何知道该区域是否为阴影区域？<br>
很简单！只要知道物体的法线方向和光照的方向！<br>
高中数学告诉我们，两个向量的点乘可以反应两个向量的夹角，如果大于0，则夹角小于90度，两者方向越平行，
如果点乘结果等于0，则两个向量互相垂直，如果小于0，则两个夹角大于90度：
通过该性质我们很容易知道法线方向与光线方向的夹角如下图所示：
![描边]({{site.url}}/assets/img/posts/CalShadow.png)
令法向量和光线向量都进行了归一化，可以知道a dot L = 1：
0< b dot L <1; c dot L = 0; d dot L = -1; 由此，可以轻易的实现将每个片元区分出是否在阴影内<br>
在unity中，我们需要引入 `#include "Lighting.cginc"`,然后
调用`_WorldSpaceLightPos0.xyz`便获取到直线光照的方向的反方向<br>
shader中实现如下：
```js
v2f vert(a2v v)
{
    v2f o;
    // ...
    // 得到世界法线方向
     o.worldNormal = UnityObjectToWorldNormal(v.normal);
    //...
    return o;
}

fixed4 frag(v2f v) :SV_Target
{
     // 获取直线光的位置
     fixed3 lightDir = normalize(_WorldSpaceLightPos0.xyz);
    // 获得点乘结果
     float diff = dot(v.worldNormal, lightDir);
    // 如果点乘小于0，则说明是在阴影中
    if(diff<0)
        return _ColorShadow;
    esle
        //否则就使用漫反射
        return _Color;
}     
```
通过上述代码，我们可以对阴影处和被光源照射处分别赋予两种颜色：


