---
date: 2020-05-23 22:50:00 +0800
layout: post
title: 卡通渲染
subtitle: 使用unity制作一个简单的卡通渲染shader~
description: 《动物森友会》中拱形地形实现原理
category: unity
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
        return _ColorShadow*_LightColor0;
    else
        //否则就使用漫反射
        return _Color*_LightColor0;
}     
```
通过上述代码，我们可以对阴影处和被光源照射处分别赋予两种颜色：
![阴影]({{site.url}}/assets/img/posts/Shadow.png)

但是，这只是通过片元的位置判断是否处于被光面，但是对于其前面的物体挡住了光线
无法判断是否在其阴影的投影中：
![没有投影]({{site.url}}/assets/img/posts/NoShadow.png)
所幸的是，unity可以很快速的判断是否在阴影中：
```js
// 加上这句
#pragma multi_compile_fwdbase
struct v2f
{
    //将阴影信息保存在UV1中
   SHADOW_COORDS(1)
};
// ...
v2f vert(a2v v)
{
   //...
    // 根据变换求解上面结构体中的float4坐标，unity5中采用的是屏幕空间阴影贴图
    TRANSFER_SHADOW(o);
    //...
};
fixed4 frag(v2f v) :SV_Target
{
    // ...
    // 获取shadow
    float shadow = SHADOW_ATTENUATION(v);
    //再增加一个判断，如果在阴影中则渲染为阴影颜色
    if(diff<0|| shadow<0.95)
        return _ColorShadow*_LightColor0;
    else
        //否则就使用漫反射
        return _Color_LightColor0;
}
```
结果如下：
![阴影]({{site.url}}/assets/img/posts/CastShadow.png)

### 高光
接着，我们需要为mesh打上高光。
高光的范围与物体的平滑度密切相关，物体越平滑，光线打在物体上
散射越少，反射区域越集中；当光线到达物体表面后，经由法线向量反射，
如果与视线的方向越平行，则反射强度越大，相反，则反射强度越小：
![阴影]({{site.url}}/assets/img/posts/Reflection.png)
在unity中，使用`_WorldSpaceCameraPos`可以获取相机的世界位置，
`mul(unity_ObjectToWorld, pos)`可以将pos从模型位置转换成世界位置
我们补充以下上述的代码： 
```js
v2f vert(a2v v)
{
    // ...
    o.worldSpacePos = mul(unity_ObjectToWorld, v.vertex);
    // ...
}

 fixed4 frag(v2f v) :SV_Target
 {
     // 获取视线方向(相机位置减去顶点位置)
     float3 viewDir = normalize(_WorldSpaceCameraPos - v.worldSpacePos.xyz);
     // 计算光照的反射方向
     fixed3 reflectDir = normalize(reflect(-lightDir, v.worldNormal));
     // 反射方向与视角方向点乘
     float reflectDotView = dot(viewDir, reflectDir);
    //使用公式计算高光区域
     float spec = pow(max(reflectDotView, 0), _Glossiness);
     float3 specular = 0;
     if(spec>0.7) 
        specular =  _GlossinessColor.rgb;
    // 这里就加一个高光颜色
     if (diff < 0||shadow<0.95)
         return (_ColorShadow)*_LightColor0;
     else
         return (_Color + float4(specular, 1))* _LightColor0;
 }
```

加入高光后，我们可以得到如下的效果：

![高光]({{site.url}}/assets/img/posts/Glossiness.png)
### 给你的反射润润色！
这么看来，似乎已经能够已经有比较好的效果，但是总觉得缺少的什么，
现在，我们请出大名鼎鼎的[菲涅尔反射](https://www.zhihu.com/question/53022233/answer/401412842)！,
以塞尔达传说为例：
![塞尔达传说的菲涅尔]({{site.url}}/assets/img/posts/FresnelInZelda.png)
可以看到，模型的向光明的边缘处，有一条比较细的白边，这就是菲涅尔造成的结果，
我们使用菲涅尔的简化公式：
<br>$$fresnel = baseFresnel+fresnelScale*pow(1-dot(N,V),indensity)$$
其中N表示视角方向，V表示顶点的法线方向。
在得到fresnel系数后,通过该系数进行对原来的颜色和菲涅尔反射颜色进行fresnel插值，
便可以得出比较自然的菲涅尔效果：
```js

Properties
{
    //...
    // 菲涅尔反射系数
    fresnelBase("fresnelBase", Range(0, 10)) = 1
    fresnelIndensity("fresnelIndensity", Range(0, 10)) = 5
    fresnelScale("fresnelScale", Range(0, 10)) = 1
    fresnelColor("fresnel Color", Color) = (1,1,1,1)
}

//...

fixed4 frag(v2f v):SV_Target
{
    //...
    //计算菲涅尔值
    float fresnel = _fresnelBase + _fresnelScale * pow(1 - dot(viewDir,v.worldNormal), _fresnelIndensity);
    //阴影处不考虑菲涅尔效果
    if (diff < 0 || shadow < 0.95)
    {
        diffuseColor = (_ColorShadow * textureColor) * _LightColor0;
        return diffuseColor;
    }
    else
    {
        diffuseColor = (_Color * textureColor + specular) * _LightColor0;
        return lerp(diffuseColor, diffuseColor * (1 - _fresnelColor.a) + _fresnelColor * _fresnelColor.a, fresnel);
    }
}
```

至此，我们已经将菲涅尔反射，高光反射，漫反射和阴影颜色都加进去了，最终成果如下：
![塞尔达传说的菲涅尔]({{site.url}}/assets/img/posts/cartoonDemo.gif)

以下是该pass的完整源码：
```js
        Pass
        {
            Tags{"LightMode" = "ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
        	#pragma multi_compile_fog
            #pragma multi_compile_fwdbase
            #include "UnityCG.cginc"
            #include "Lighting.cginc"
           #include "AutoLight.cginc"

            //要想有正确的衰减内置变量等，必须要有这句

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
                //宏表示为定义一个float4的采样坐标，放到编号为1的寄存器中
                float4 worldSpacePos:COLOR;
                // 存储纹理坐标
                float2 uv:TEXCOORD2;
                SHADOW_COORDS(1)
            };
            float4 _Color;
            float4 _ColorShadow;
            float4 _GlossinessColor;
            float _Glossiness;
            //float _OutineStrength;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldSpacePos = mul(unity_ObjectToWorld, v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                // //根据变换求解上面结构体中的float4坐标，unity5中采用的是屏幕空间阴影贴图
                TRANSFER_SHADOW(o)
                return o;
            }

            fixed4 frag(v2f v) :SV_Target
            {
                // 将阴影部分和接受光源的部分分开
                // 获取直线光的位置
                fixed3 lightDir = normalize(_WorldSpaceLightPos0.xyz);
                float diff = dot(v.worldNormal, lightDir);
                // 纹理贴图
                fixed4 textureColor = tex2D(_MainTex, v.uv);
                // 获取视角方向
                // 有问题
               // float3 viewDir = UNITY_MATRIX_V[2].xyz;
                float3 viewDir = normalize(_WorldSpaceCameraPos - v.worldSpacePos.xyz);
                // 计算反射方向
                fixed3 reflectDir = normalize(reflect(-lightDir, v.worldNormal));
                // 反射方向与视角方向点乘
                float reflectDotView = dot(viewDir, reflectDir);
                float spec = pow(max(reflectDotView, 0), _Glossiness);
                float3 specular = 0;
                if(spec>0.7)
                specular =  _GlossinessColor.rgb;
                // 获取shadow
                float shadow = SHADOW_ATTENUATION(v);

                // 描边，如果视角方向与法线方向越垂直，点乘越接近0，则越为边缘
                //if (dot(viewDir, v.worldNormal) < _OutineStrength /2)
                //{
               //     return fixed4(0,0,0,1);
               // }
                if (diff < 0||shadow<0.95)
                    return (_ColorShadow* textureColor +float4(specular,1))*_LightColor0;
                else
                    return (_Color* textureColor + float4(specular, 1))* _LightColor0;
            }
            ENDCG
        }
```
### 最后一步，描一下边！
卡通卡通，当然少不了描边啦，描边分为两种，一种是只描述其轮廓，
一种是对于模型内部的边缘也进行勾勒，
第一种一般作为对象选中的效果，如unity编辑器中选中一个模型
第二种才是我们需要做的事情；
可以利用后处理通过其GBUffer的深度值进行拉普拉斯算子来检测边缘，
当然，我们使用另外一种方法：将模型依照法线方向往外扩一圈，然后剔除正面
在描边厚度不大的情况下，会有很好的效果：也就是说我们需要使用第二个pass来渲染这个模型：
话不多说，直接贴源码！
```js
        Pass
        {
           // 描边Pass
           Name"OUTLINE"
           // 剔除正面
           Cull Front
           CGPROGRAM
            #pragma vertex vert 
            #pragma fragment frag 

            #include "UnityCG.cginc"
           struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };
            float _OutineStrength;
            struct v2f

            {
                float4 pos :POSITION; 
            };
            v2f vert(a2v v)
            {
                // 模型空间转法线
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
		// 将法线转成从模型空间转成观察空间
                float3 norm= normalize(mul((float3x3)UNITY_MATRIX_IT_MV, v.normal));
		//计算出模型的偏移值，将观察空间转换成透视空间，与o.pos处于同一空间下
                float2 offset = TransformViewToProjection(norm.xy);
		// 移动模型的顶点位置
                o.pos.xy += offset* _OutineStrength;
                return o;
            }
            fixed4 frag(v2f i) :COLOR
            {
		//返回描边颜色
                return fixed4(0,0,0,1);
            }
                ENDCG
        }
```
将所有的效果开启之后，我们可以得到以下的效果：
![开启描边]({{site.url}}/assets/img/posts/OutLineOn.png)

当然，这个直接向外扩展法线存在一些问题，如果该模型是一个cube，使用描边后：
![描边的问题]({{site.url}}/assets/img/posts/BadOutline.png)
描边可以很清楚的看见他与模型是分开了，应用这种方式描边，一定要注意该模型是否适合；
<br>
<br><br>好了，本次教程结束，如果你有什么更好的描边方式和卡通渲染方式，欢迎在下面留言讨论噢~
