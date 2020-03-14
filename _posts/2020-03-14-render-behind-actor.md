---
date: 2020-03-14 12:27:40 +0800
layout: post
title: 游戏中主角被遮挡解决方法
subtitle: 玩家被遮挡怎么办？这篇教程帮助你！
description: 游戏中角色被遮挡显示的解决方法
category: opengl
image: >-
    https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/marioOdysseyShotScreen2.png
tags:
  - opengl
  - 图形学
  - 游戏设计技巧
paginate: true
---
在游戏中，经常出现镜头面前被操纵角色被其他物体遮挡而影响操作的问题，一般有以下三种解决方法：
* 将影响视线的物体透明化
* 摄像头进行碰撞判断，使得摄像头永远不会在场景物体后面（防止摄像头穿墙）
* 将主角渲染到整个屏幕的顶部

接下来我将使用opengl，来实现第三中解决方法！

以马里欧奥德赛为例，当马里奥被石柱遮挡时，将显示一个主角的裁影

![马里奥奥德赛]({{site.url}}/assets/img/posts/marioOdysseyShotScreen1.png)

## 分析
要想实现这个效果，需要得到以下信息：

* 主角渲染到屏幕上的像素位置
* 主角是否被遮挡
* 裁剪的材质

## 知识点
* 模板测试： 在着色器处理完一个片段之后，模板测试(Stencil Test)会开始执行，若没有通过模板测试，
则丢弃该片段，不进行渲染。每一个像素都有一个8位的模板值，我们可以对其写入你想要的值，通常我们只要0和
1这两个值就够了。之后我们对此进行判断，来选择保留或求其这个片段。
![模板测试](https://learnopengl-cn.github.io/img/04/02/stencil_buffer.png)
在openGL中模板测试常用函数如下：
>`glEnable(GL_STENCIL_TEST)`  启用模板测试  
>`glStencilFunc(GLenum func, GLint ref, GLuint mask)`  模板测试规则  
>* `func` : 设置测试规则，通过该规则与ref比较，有GL_NEVER、GL_LESS、GL_LEQUAL、GL_GREATER、GL_GEQUAL、GL_EQUAL、GL_NOTEQUAL和GL_ALWAYS。  
>* `ref`: 比较值，模板缓冲的模板值将和ref进行比较  
>* `mask`： 在模板缓冲中的模板值与ref比较时，将会先与这个模板进行AND运算，默认为1
>
>`glStencilOp(GLenum sfail, GLenum dpfail, GLenum dppass)` 写入模板缓冲函数规则  
>*  `sfail`:  模板测试失败时采取的行为。  
>*  `dpfail`: 模板测试通过，但深度测试失败时采取的行为。  
>*  `dppass`: 模板测试和深度测试都通过的行为, 每个选项选择以下一种：  
>* 
|  行为   | 描述  |
|  ----  | ----  |
| GL_KEEP  | 保持当前储存的模板值|
| GL_ZERO  | 将模板值设置为0 |
| GL_REPLACE  | 将模板值设置为`glStencilFunc`函数设置的`ref`值 |
| GL_INCR  | 如果模板值小于最大值则将模板值加1 |
| GL_INCR_WRAP  | 与 GL_INCR 一样，但如果模板值超过了最大值则归零 |
| GL_DECR  | 如果模板值大于最小值则将模板值减1 |
| GL_DECR_WRAP  | 与GL_DECR一样，但如果模板值小于0则将其设置为最大值 |
| GL_INVERT  | 按位翻转当前的模板缓冲值 |
* 深度测试：模板测试之后就是深度测试。同模板测试一样，每一个片元中同样存储了改片元的深度值，通过深度
比较当前片段与深度缓冲中的深度值，从而得出是否该片段位于其他片段后方。

## 实现原理
实现该效果的原理不复杂，我们只需要 **在渲染正常角色时，若没有通过深度测试但通过了模板测试写入一个模板值，
之后再通过之前写入的模板值渲染裁剪效果**，具体做法大致如下：
1. 清除模板缓冲
2. 设置模板写入参数,当模板和深度都通过时，就写入1
3. 启用模板函数，设定为ALWAYS, 将模板值写入到片元中
4. 渲染角色
5. 设置模板写入参数，当模板和深度都通过时，写入0，
6. 渲染场景
7. 禁用深度测试和模板写入，启用模板函数
8. 在一次渲染角色，这次使用裁剪效果
9. 重新启用模板写入和深度测试

## 开始实现吧！

首先搭建一个简单的场景，我们将mur猫作为游戏的主角：
![场景样式]({{site.url}}/assets/img/posts/demoScene.png)
再一个制作裁剪效果的shader，这里比较简单，单纯选择一个突出的颜色：
```js
// vertex Shader
#version 330 core

layout(location = 0) in vec3 aPos;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
	gl_Position = projection *view *vec4(vec3(model *vec4(aPos,1.0)),1.0);
}
```

```js
// fragment Shader
#version 330 core
out vec4 FragColor;



void main()
{
    FragColor = vec4(1,0.714,0.7569,1); // 将向量的四个分量全部设置为1.0
}
```
结果如下：
![shader效果]({{site.url}}/assets/img/posts/clipShader.png)


接下来编写渲染代码：

```c
//...
	// 材质设置 ......
	shader->use();
	shader->setInt("shadowMap", 10);
	shader->setMat4("lightSpaceMatrix", lightProjection * view);
	glActiveTexture(GL_TEXTURE10);
	glBindTexture(GL_TEXTURE_2D, DepthMap);
	//...... 材质设置
	view = camera.GetViewMatrix();


	// 开启模板测试
	glEnable(GL_STENCIL_TEST);
	glStencilMask(0xff);
	// 开启深度测试
	glEnable(GL_DEPTH_TEST);
	// 清除模板缓冲、颜色缓冲和深度缓冲
	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);


	// 设置写入规则，当深度和模板都测试成功则写入1
	glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);
	// 启用渲染测试函数，这里设置为always,表示深度测试永远通过
	glStencilFunc(GL_ALWAYS, 1, 0xFF);
	// 绘制角色
	murCat->Draw(*shader, view, projection);
	
	// 设置写入规则，当深度和模板都测试成功则写入0，其他保持原样，由于上面启用了Always，这里就不再重复
	glStencilOp(GL_KEEP, GL_KEEP, GL_ZERO);
	// 正常渲染场景
	drawScene(shader, view, projection);

	/*接下来渲染角色的剪影*/
	glStencilFunc(GL_EQUAL, 0, 0xFF);
	glStencilMask(0x00);
	// 关闭深度测试，由于剪影一直渲染在屏幕的顶部
	glDisable(GL_DEPTH_TEST);
	// 使用剪影材质渲染角色
	murCat->Draw(*clipShader, view, projection);
	glStencilMask(0xFF);
	glEnable(GL_DEPTH_TEST);
	glDisable(GL_STENCIL_TEST);
//...
```
最终结果如下：
![shader效果](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/Result.gif)

是不是很完美O(∩_∩)O~~当然，您也可以加一些贴图在上面，使得该效果更加炫酷，

加油！
