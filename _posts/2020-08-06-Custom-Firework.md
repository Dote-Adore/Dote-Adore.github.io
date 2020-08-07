---
date: 2020-08-06 19:11:43 +0800
layout: post
title: 自定义烟花效果
subtitle: 夏季的《动物森友会》烟花大会开始啦！自定义的烟花到底该如何实现呢？
description: 《动物森友会》中自定义烟花效果
category: projects
image: >-
    https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/CustomFireworksBG.png
tags:
  - UE4
  - 图形学
  - 游戏设计
paginate: true
---
上个周末看到学校群里的小伙伴们分享了在动物森友会中一起参加烟火大会，不得不说这游戏真的太温馨了，然而没时间的我错过了介么开心的时刻QAQ。
这时候就有小伙伴问了：“这个自定义烟花该怎么实现啊~”;学术不精的我尝试使用UE4来简单复刻一下自定义烟花的实现，先放下最终的效果：
![效果](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/FireworkDemo.gif)
怎么样，是不是有一点动物森友会的味道(●'◡'●)，那下面我就来讲解一下我的实现思路~~；
![2](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/firework_Material_BP.png)


# 简单分析
1. 自定义动画当然是需要一张自定义的图片，作为自定义烟花的采样，
然后该shader可以自定义烟花的横向和竖向的数量，根据其横向和纵向的数量进行采样（像素化处理）
2. 烟花炸开后每一片火花默认情况下是圆形的， 我们可以自定义每一片火花的样式，进行平铺操作
3. 如果是单纯的平铺，会在烟花炸裂时会显得太工整，所以需要对每一个烟花进行随机的一些偏移
4. 当烟花炸开时，每一片火花之间会相互原理，所以需要有一个值控制每一片火花之间的距离

# 详细实现：
1. 首先要解决的是烟花平铺的数量，我这里使用其UV坐标来进行平铺问题；
![4](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl1.png)
当输入目标数量时，TexCoord*PixelNums，将坐标从0~1
映射到0~PixelNums，再进行取整，便将线性空间转换成离散空间，再除以PixelNums，重新将UV范围映射到0~1,
这样，便实现了比较简单、开销较小的采样，每一块区域的采样时都是其最左上角的值；将该结果接入到我们的自定义图片时，效果如下：
![4](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl1-2.png)
2. 其次，进行透明度遮罩：<br>
由于我们只需要中间的圆形部分，我们将第一步得到的结点的0点移动到正中间，再得到每一个像素的length，
通过Radius来控制可以显示的圆形的大小
![4](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl2-1.png)
3. 重点是接下来对于每一个采样后的像素点就行取样，我们需要重复N*N次的UV值<br>
通过将第一步中的两个结点相减，便可以将uv进行重复了，得到N*N个标准的UV平铺<br>
4. 然后再设定每一个烟花之间的间隔（grap），通过条件该数值，可以展现其生命周期，间隔越大说明烟花炸裂的越开：
![4](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl2-3.png)
这里其实我做了一个取巧，直接将grap乘以uv，再次将每一个uv拓展到倍数，到后面只保留在0~1之间的数，
也就是说Grap其实是空闲空间与非空闲空间的比值；
5. 进行随机打乱：这里我使用了一张perlin纹理，按照步骤1的方法取样后，进行一个伪随机偏移，这里输入的是步骤1的结果，
然后输出的是打乱的值(注意，我这里后面又将结果lerp到0~grap中)
![5](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl5-1.png)
之后再与第四步相减，就可以将每一块的uv的(0,0)处重新在每一块的范围内进行随机的分布：
![5](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl5-2.png)
6. 上一步我们输出的是uv坐标的偏移值，但是会有小于0或者大于1的值，我们需要只需要保留0~1之间，并将此之外的值设置为0，
这列很简单，直接放结点：
![6](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl6-1.png)
最后得到如下效果：
![6](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl6-2.png)
7. 然后我们将第6部的输出作为我们每一片火花mask的uv，就可以根据该uv得出一个按照N*N排列的伪随机的uv mask】
![7](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl7-1.png)
8.最后，将将第一步像素化的图像与上一步的结果相乘，得到最终的颜色分布，然后再通过最终颜色得到灰度值，
当灰度值小于0.05时，就设置为透明：
![8](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl8-1.png)
9. 烟花的blinblin效果：在第一步得到的结果输入到第八部之前，我们需要再对其亮度进行处理：
blinblin效果同样使用perlin噪声进行伪随机，效果差不多，只不过使用了time进行跟随时间自动随机，
再使用BlingFreq进行控制：
![9](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl9-1.png)
10. 我同时还对于mask做了一个随机删点，其实也是对像素化的perlin图进行一定范围的删除即可：
![10](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/impl10-1.png)

# 总结
1. 其实这个材质还是有一些问题的，比如每个火花的随机值只能限定在每一个方方正正的方块中，如果没有grap，则无法进行随机
2. 在我实现的样例中，我是使用了3个平面进行随机，达到了比较好的效果，当然，观察动物森友会的实现，其实也最少有两个平面的；

#### 下面是工程链接，有兴趣的小伙伴可以自行下载研究~~
谢谢大家~~

[点击此处下载该demo工程](https://ncutradingplatform.oss-cn-shanghai.aliyuncs.com/posts/custom-firework/ACFirework.zip)
