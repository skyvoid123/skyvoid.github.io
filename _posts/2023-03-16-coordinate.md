---
layout: post
title: 坐标系差异（OpenGL/Metal/Vulkan/D3D）
category: Graphics-API
---

**原创内容，转载请注明**

---

## 概述

|         | Metal/D3D | Vulkan | OpenGL/ES |
|:-------:|:---------:|:------:|:---------:|
| 世界坐标 |     左手   |   右手  |   右手    |
| 相机坐标 |     左手   |   左手  |   左手    |
| NDC | 左手<br>(-1,-1)在左下角 ↑ | 右手<br>(-1,-1)在左上角 ↓ | 左手<br>(-1,-1)在左下角 ↑ |
| Framebuffer坐标 | (0,0)在左上角 ↓ | (0,0)在左上角 ↓ | (0,0)在左下角 ↑ |
| 纹理坐标 | (0,0)在左上角 ↓ | (0,0)在左上角 ↓ | (0,0)在左下角 ↑ |

## 世界坐标系（3D）

- OpenGL/OpenGL ES、Vulkan： 右手坐标系
- Metal、D3D： 左手坐标系

## 相机坐标系（3D）

- OpenGL/OpenGL ES、Metal、D3D、Vulkan：左手坐标系

其中，OpenGL/OpenGL ES、Vulkan 在相机变换过程中交换了左右手，相机变换之前是右手坐标系。

## NDC（3D）

投影变换+透视除法之后的坐标空间

- OpenGL/OpenGL ES、Metal、D3D：Y轴向上为正，(-1,-1)在左下角，左手坐标系
- Vulkan： Y轴向下为正，(-1,-1)在左上角，右手坐标系

## Vulkan NDC 差异解决方案

解决Vulkan的NDC手性不一致带来的问题，分为下面三个方案：

- 在Shader中翻转
- 翻转投影矩阵
- 使用Vulkan的扩展，可以设置Viewport为负数，并且设置Viewport的Y值，便可完成翻转。

**方案一**
第一个方案在shader中翻转，我们可以考虑在顶点Shader里面进行翻转，在顶点Shader里面，我们只需要对顶点输入的时候做一次转换，将顶点坐标以及附带的矢量(法线之类)一起翻转，然后将包装之后的属性提供出来使用。如下所示：
{% highlight C++ %}
gl_Position.y = -gl_Position.y;
{% endhighlight %}
该方案缺点:
- 在集成不同的图形API中比较麻烦，并且不了解的人会一头雾水。
- 在做一些屏幕空间算法的时候，比如从屏幕空间转换为视线空间，还需要再翻转一次才能够计算正确。

**方案二**
翻转投影矩阵，我们可以让我们的投影矩阵Projection Matrix进行垂直翻转，处理也很简单，Y轴缩放乘以-1即可。
{% highlight C++ %}
pro[1][1] *= -1;
{% endhighlight %}
该方案缺点:
- 三角形剔除问题，因为我们进行了垂直翻转，那么我们的三角形缠绕顺序就变了，以前是顺时针的现在就变成了逆时针。然而背面剔除需要根据此为判断，因为相应的背面剔除设置也需要修改。
- 顶点属性问题，投影矩阵的翻转是只针对顶点坐标，但是对于法线等顶点属性是没有翻转的。那么跟顶点相关的向量属性也要一并翻转。这种方式处理起来就异常麻烦了，可能整个引擎架构都需要随之调整。

**方案三**
就是使用Vulkan提供的VK\_KHR\_maintenance1扩展。在这个在Vulkan 1.1就已经纳入核心规范中了，如果是Vulkan1.0则需要显式启用。需要做的修改如下所示:

{% highlight C++ %}
forwardPipeLine.viewport.x = 0.0f;
forwardPipeLine.viewport.y = swapchainExtent.height;
forwardPipeLine.viewport.width = swapchainExtent.width;
forwardPipeLine.viewport.height = -swapchainExtent.height;
{% endhighlight %}

这个方案也是在上面正常渲染使用的方案。可以使用Vulkan和OpenGL、Metal、DirectX保持一致。在处理跨平台的问题上更加得心应手。并且如果你也有一些常用的库，比如模型解析库，解析出来的模型坐标系已经是跟左手细对应上了，结果现在解析出来放到Vulkan里面却发现模型翻转了。只是静态模型其实还好处理，关键麻烦的是我们还有骨骼动画，还涉及到骨骼是不是需要翻转，翻转了之后能不能用等等问题。所以使用这个扩展也可以减少各种库迁移的问题。

## 窗口/像素/FrameBuffer 坐标系（2D）
视口变换（viewport，光栅化阶段）之后的坐标空间，FragmentShader所在的坐标系，亦即FrameBuffer的坐标系。

- OpenGL/OpenGL ES： Y轴向上为正，(0,0)在左下角
- Metal、D3D、Vulkan： Y轴向下为正，(0,0)在左上角  

> Note: pixel centers are offset by (0.5, 0.5). For example, the pixel at the origin has its center at (0.5, 0.5); the center of the adjacent pixel to its right is (1.5, 0.5). This is also true for textures.

## 纹理坐标系（2D）
- OpenGL/OpenGL ES： Y轴向上为正，(0,0)在左下角。glTexImage2D等传入的data数据从图像的左下角开始
- Metal、D3D、Vulkan： Y轴向下为正，(0,0)在左上角。[replaceRegion:mipmapLevel:withBytes:bytesPerRow:](doc://com.apple.documentation/documentation/metal/mtltexture/1515464-replaceregion?language=swift)等传入的data数据从图像的左上角开始

> 注：加载纹理数据时，需保证纹理数据的存储顺序与纹理坐标系一致。