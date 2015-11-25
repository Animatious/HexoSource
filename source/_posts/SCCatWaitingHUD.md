title: SCCatWaitingHUD的Objc代码实现
date: 2015-11-25 18:19:49
tags:
- WaitingHUD 
- ObjC
categories:
- 代码

---

# 介绍这个创意其实来源于微博上@画渣程序猿mmoaay转发并艾特我的的一组gif图  
看到图的时候 我先对图大致进行了结构和层次的区分在设计物体动效的时候，首先是要对动画的不同对象进行拆分，在这里，老鼠，猫眼睛，眼皮，以及猫脸，他们各自的行为分别是不同的，因此分为不同的图层 

![](/img/2015/11/SCCatWaitingHUD/pic1.jpg)
 在整体动画进行的时候，他们各自的layer所执行的动画是不同的  看起来这个控件很简单，接下来我们就来分析一下写出这样一个控件需要注意的关键点吧~# 分析先确认我们要做成什么样的控件  
## 首先 这是一个等待动画 所以应该做成一个HUD类型的控件  
这样可以在当前window的上方出现和消失而不影响当前视图的展示  
所以我们就要在应用当前默认的UIWindow上再添加一个UIWindow 这样可以在全局任何地方调用这个HUD 同时我们的HUD也要实现全局的单例模式  
## 其次是动画 最初的动画其实并不难  
只是眼睛图层和老鼠图层围着各自的中心点旋转 且拥有相同的运动周期 这样能让他们在相同时间内移动的弧度是相同的 产生眼睛跟着老鼠走的感觉

![](/img/2015/11/SCCatWaitingHUD/circle.png)
  
## 最后是眼皮和眼珠的运动协调  
由于眼珠做的是匀速圆周运动 而眼皮做的是直线缩放运动 因此如何协调这两者的运动时间曲线 决定了最后的动画是否流畅和自然  
### 还有一点额外的临时产生的需求就是对横屏的支持  
这是我在写的过程中@sauchye提出的issue，所以我花了一个版本用来加上了对左右横屏的支持# 代码实现
## UIWindow

UIWindow 是每一个程序都必须创建的根视图，它是UIView的子类，但是地位却在所有的UIView之上。每一个程序在启动的时候，如果你使用了interface builder作为启动界面，那么Xcode会自动把你的第一个Controller添加到根window上，如果你要通过代码初始化Window，则必须要像系统要求的这样声明：

```
- (BOOL)application:(UIApplication *)application willFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

   UIWindow *window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];

   myViewController = [[MyViewController alloc] init];
   window.rootViewController = myViewController;
   [window makeKeyAndVisible];

   return YES;

}
```


在SCCatWaitingHUD中，由于需要存在一个不干扰原有程序代码逻辑和视图逻辑的View，因此最合适的办法就是声明一个独立的Window。HUD的这种使用模式可以参考MBProgressHUD，开发者可以在全局任意的地方调用HUD的显示，由于HUD所在的Window和系统原本的Window相比level更高，所以可以在不影响原有视图的情况下在任意页面显示。

UIWindow的结构如下，我们可以看到UIWindow本身就是一个UIView，有一个名为rootViewController的属性，而这个属性就是这个window将会呈现的controller对象。

Xcode7之后，苹果要求所有的UIWindow在声明的时候都需要有一个rootViewController，即通过代码声明的时候，需要定义一个rootViewController，然后在这个controller之上添加要显示的内容。但是经过验证，在程序运行中创建的非根Window的UIWindow，可以直接当做UIView来使用，仍然不需要强制给一个rootViewController。```
self.backgroundWindow = [[UIWindow alloc]initWithFrame:self.frame];
_backgroundWindow.windowLevel = UIWindowLevelStatusBar;
_backgroundWindow.backgroundColor = [UIColor clearColor];
_backgroundWindow.alpha = 0.0f;```
所以我们这个HUD的主要显示对象就是名为`backgroundWindow`的UIWindow属性对象  
那么怎么控制Window的出现和消失呢  UIWindow有几个方法:  
```
- (void)makeKeyWindow;
- (void)makeKeyAndVisible;                             // convenience. most apps call this to show the main window and also make it key. otherwise use view hidden property```
一般都是用上述第二个方法来让指定Window成为keyWindow并且出现。至于怎么让指定Window消失，苹果并没有提供一些特别的办法，官方文档中有给出下面这种用法
```_backgroundWindow.hidden = YES;// According to Apple Doc : This is a convenience method to make the receiver the main window and displays it in front of other windows at the same window level or lower. You can also hide and reveal a window using the inherited hidden property of UIView.```## Animation
动画要解决的第一个问题就是，**老鼠的旋转是绕着自身的中心点，然而两个眼珠的旋转并不是**。  首先，旋转动画的实现我们肯定是通过`CATransform3DRotate`，基本的CAAnimation声明如下:

```
double radians(float degrees) {
    return ( degrees * 3.14159265 ) / 180.0;
}
CATransform3D getTransForm3DWithAngle(CGFloat angle)
{
    CATransform3D  transform = CATransform3DIdentity;
    transform  = CATransform3DRotate(transform, angle, 0, 0, 1);
    return transform;
}
- (CABasicAnimation *)rotationAnimation
{
    CABasicAnimation *rotationAnimation;
    rotationAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
    rotationAnimation.fromValue = [NSValue valueWithCATransform3D:getTransForm3DWithAngle(0.0f)];
    rotationAnimation.toValue = [NSValue valueWithCATransform3D:getTransForm3DWithAngle(radians(180.0f))];
    rotationAnimation.duration = self.animationDuration;
    rotationAnimation.cumulative = YES;
    rotationAnimation.repeatCount = HUGE_VALF;
    rotationAnimation.removedOnCompletion=NO;
    rotationAnimation.fillMode=kCAFillModeForwards;
    rotationAnimation.autoreverses = NO;
    rotationAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
    return rotationAnimation;
}
```
我们需要来看看CALayer的几个基本属性：frame、bounds、position、anchorPoint。frame和bounds都可以用UIView中的知识去理解，frame是相对于superLayer的位置和大小，而bounds是origin为(0,0)的frame。  而anchorPoint则是一个Layer的锚点，相信做过一些动画开发或者游戏开发的同学们都知道锚点的作用。锚点作为Layer变换的基准点，它的坐标值为相对于bounds宽高的比例。在iOS中，它的起始点是Layer的左上角，为标准原点(0，0)，右下角为(1,1)，默认值为中心点(0.5,0.5)；在MacOS中，起始点为左下角，(1,1)在右上角。所以通过修改锚点的相对位置，就可以让旋转的轴心变成我们需要的位置。  
这里有一张图来说明锚点是什么。
![](/img/2015/11/SCCatWaitingHUD/anchorpoint2.jpg)
因此我尝试着将眼珠的anchorPoint设置为(1.5,1.5)，旋转中心变成了接近眼睛中心的点。但是这时候我们还需要修改position来配合anchorPoint的改变。我们可以这么理解，position是锚点相对于父layer的坐标值，而anchorPoint则是锚点相对于自身的坐标值，两者共同决定了锚点和整个Layer所在的位置。  
所以我们还需要调整眼珠Layer的position，来和之前设置的anchorPoint的位置重合。

```
_rightEye.layer.anchorPoint = CGPointMake(1.5f, 1.5f);
_rightEye.layer.position = CGPointMake(self.faceView.right - 13.5f, self.faceView.top + width/3.0f + 7.5f);
```

这时候，上面的`CAAnimation`就可以正常的添加到Layer的transform上并且执行出来啦。

## 眼皮的运动时间曲线

这里最重要的一点就是要让眼皮和眼珠的运动达到协调一致，不会出现翻白眼也不会出现斗鸡眼（哈哈哈哈）。所以我们可以先分析出下面两个结论:

* 眼皮的上下运动周期和眼珠的旋转运动周期一致
* 眼皮在竖直方向上的位移和眼珠在竖直方向上的位移一致
* 眼珠在圆周切线方向的线性速率是匀速的

因此我们可以得出结论，眼皮在竖直方向上的速率曲线和眼珠在竖直方向上的速率曲线是一致的。我们来分析眼珠的速度方向。

![](/img/2015/11/SCCatWaitingHUD/v.jpg)

通过将速度分解成水平X轴和竖直Y轴的分量后，我们可以得出竖直方向上的眼珠速度的速度曲线，就是简单的  
![](/img/2015/11/SCCatWaitingHUD/sinx.png)  

函数图像如下
![](/img/2015/11/SCCatWaitingHUD/Vy.jpg)  

这里的y就是竖直速度分量Vy，x是t时刻的圆心角Q，由于这里做的是匀速圆周运动，因此角速度也是匀速的。我们可以将半周的时长定为常量T，半周的弧度是π，因此x和t的关系可以表示为  
![](/img/2015/11/SCCatWaitingHUD/hudu.png)

大家知道位移等于速率乘以时间，因此我们有了速率公式，就可以得出眼皮在竖直方向上的位移公式如下  
![](/img/2015/11/SCCatWaitingHUD/sinxx.jpg)
这里我将常量T的值直接带入了。

这个函数的图像如下
![](/img/2015/11/SCCatWaitingHUD/Sy.jpg)

注意，我们得出这个函数图像的主要目标是确定眼皮的上下缩放动画的时间曲线，这个曲线代表着位移和时间的函数关系。这时候，这个函数图像就是接下来要给眼皮加上的缩放动画的时间曲线了，我没有用专门的画贝塞尔曲线的软件来画，只是根据函数的图像大致画出了一个贝塞尔曲线，作为了下面这个动画的时间曲线

```
- (CAAnimationGroup *)scaleAnimation
{
    // 眼皮和眼珠需要确定一个运动时间曲线
    CABasicAnimation *scaleAnimation;
    scaleAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
    scaleAnimation.fromValue = [NSValue valueWithCATransform3D:CATransform3DIdentity];
    scaleAnimation.toValue = [NSValue valueWithCATransform3D:CATransform3DMakeScale(1.0, 3.0, 1.0)];
    scaleAnimation.duration = self.animationDuration;
    scaleAnimation.cumulative = YES;
    scaleAnimation.repeatCount = 1;
    scaleAnimation.removedOnCompletion= NO;
    scaleAnimation.fillMode=kCAFillModeForwards;
    scaleAnimation.autoreverses = NO;
    scaleAnimation.timingFunction = [CAMediaTimingFunction functionWithControlPoints:0.2:0.0 :0.8 :1.0];
    scaleAnimation.speed = 1.0f;
    scaleAnimation.beginTime = 0.0f;
    
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.duration = self.animationDuration;
    group.repeatCount = HUGE_VALF;
    group.removedOnCompletion= NO;
    group.fillMode=kCAFillModeForwards;
    group.autoreverses = YES;
    group.timingFunction = [CAMediaTimingFunction functionWithControlPoints:0.2:0.0:0.8 :1.0];
    
    group.animations = [NSArray arrayWithObjects:scaleAnimation, nil];
    return group;
}
```

你可以看到`CAMediaTimingFunction`，这就是时间曲线的类，在一般情况下，系统会提供这么几种你常用到的时间曲线

```
CA_EXTERN NSString * const kCAMediaTimingFunctionLinear
    __OSX_AVAILABLE_STARTING (__MAC_10_5, __IPHONE_2_0);
CA_EXTERN NSString * const kCAMediaTimingFunctionEaseIn
    __OSX_AVAILABLE_STARTING (__MAC_10_5, __IPHONE_2_0);
CA_EXTERN NSString * const kCAMediaTimingFunctionEaseOut
    __OSX_AVAILABLE_STARTING (__MAC_10_5, __IPHONE_2_0);
CA_EXTERN NSString * const kCAMediaTimingFunctionEaseInEaseOut
    __OSX_AVAILABLE_STARTING (__MAC_10_5, __IPHONE_2_0);
```

他们分别代表了匀速运动，先快后慢，先慢后快，先快后慢再快这四种运动时间曲线。如果你要自定义时间曲线的话，系统也提供了绘制贝塞尔曲线的方法，你只要确定曲线的两个控制点坐标即可。而坐标的原点为(0，0)，最后收尾的点则在(1,1)，横轴为时间，纵轴为动画的执行程度，坐标值代表着相对比例值，因此我们可以截取上面的函数图像中横轴0到π的部分作为0到1的时间长度，来绘制贝塞尔曲线。

## 最后是一点对横屏的支持

由于我们实现的是一个独立的UIWindow，所以不能通过controller的几个涉及到屏幕旋转的回调来获得屏幕方向的变化。屏幕方向的旋转只会自动这时候我们就要通过监听系统的一个广播消息来获取屏幕状态的变化。

```
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(statusBarOrientationChange:)  name:UIApplicationDidChangeStatusBarOrientationNotification object:nil];

```

在横屏切换的回调里，我记录了前一个屏幕状态，获取到当前屏幕状态，然后做了一个旋转的变换，这样整个HUD就会随着屏幕方向的变化而变化了。详细的实现可以直接看源代码，这里就不再贴出。

# 效果
这个开源控件的基本原理也就这些了，你可以在我的[github](https://github.com/SergioChan/SCCatWaitingHUD)上clone下源代码来阅读 :-) ，如果你有更好的建议或者想法也欢迎随时提PR。