title: tvOS视差按钮的ObjC实现
date: 2015-11-18 20:31:54
tags:
- tvOS 
- ObjC
categories:
- ObjC

---

 
# 介绍
苹果在最新发布的Apple TV里引入了有趣的[图标设计](https://developer.apple.com/tvos/human-interface-guidelines/icons-and-images/)  
具体说来 图标由2-5个分层图层构成 在图标被选中的时候 图标内每个图层进行不同幅度的位移 从而形成视觉上具有深度距离感的视差效果 图标构成和效果可以见视频:  
  
<video id="video" controls="" preload="none" autoplay loop>
												<source src="https://developer.apple.com/tvos/human-interface-guidelines/icons-and-images/images/icons-and-images-layering.mp4" type="video/mp4">
												<source src="https://developer.apple.com/tvos/human-interface-guidelines/icons-and-images/images/icons-and-images-layering.ogv" type="video/ogg">
												<source src="https://developer.apple.com/tvos/human-interface-guidelines/icons-and-images/images/icons-and-images-layering.webm" type="video/webm">
											</video>  
这种效果很适合用于多媒体类应用 例如图书或者电影封面 让封面变得立体生动 然而这种效果目前只能在Apple TV的tvOS里见到  
所以 如何在iOS上做出同样的效果呢？现在就让我们一起研究下视差按钮的实现原理 并且自己实现一个吧 ^_^  

原理
--
假设我们已经有了以下图片：（你可以从[下载链接](/files/2015/11/tvOS-Gear.zip)下载已经分层的四张图片）  
![四张已经分层的图片](/img/2015/11/tvOSButtonPic_1.jpg)  
基于这四张图片 我们该怎么对其进行变换来达到tvOS的视差效果呢？    
重新观察上文中苹果官方的例子视频 我们可以得出以下结论：  
### 1. 总图层在旋转 但不同于一般在屏幕平面上的旋转 而是相对屏幕具有一定夹角的旋转  
如果不太了解这种旋转是怎么发生的 我们可以看一张有关 CATransform3D 的图：
![CATransform3DMakeRotation的XYZ轴](/img/2015/11/tvOSButtonPic_2.jpg)    
我们常用的 CGRect 有 X 和 Y 两个位置参数 而 CATransform3D 可以理解为在日常的两个轴以外加了 Z 轴 方向为从手机上表面竖直向下 如图   
这么一来 我们日常所见的在屏幕平面上的旋转（比如屏幕旋转）其实是绕 Z 轴旋转   
而绕 X 和 Y 轴的旋转 便是让 CALayer 具有相对屏幕具有一定夹角的旋转 具体表现就是如同tvOS按钮一样有远近效果（其实在透视效果上还是有些不一样 后面会提怎么解决）  
`CATransform3DMakeRotation` 就是这么通过旋转角度定义的：
```
CATransform3D CATransform3DMakeRotation (
   CGFloat angle, //绕着向量(x,y,z)旋转的角度
   CGFloat x, //x轴分量
   CGFloat y, //y轴分量
   CGFloat z //z轴分量
);
```
### 2.除去旋转 每个图层都在进行圆周路径的移动  
为什么有移动？  
透视是创造三维深度感觉的关键 而透视效果最直白的话来说 就是“近大远小”  
让我们来看个例子吧 :-)  
