title: SCCatWaitingHUD的Objc代码实现
date: 2015-11-25 18:19:49
tags:
- WaitingHUD 
- ObjC
categories:
- 代码

---

## 介绍
看到图的时候 我先对图大致进行了结构和层次的区分

![](/img/2015/11/SCCatWaitingHUD/pic1.jpg)
 
### 首先 这是一个等待动画 所以应该做成一个HUD类型的控件  
这样可以在当前window的上方出现和消失而不影响当前视图的展示  
所以我们就要在应用当前默认的UIWindow上再添加一个UIWindow 这样可以在全局任何地方调用这个HUD 同时我们的HUD也要实现全局的单例模式  
### 其次是动画 最初的动画其实并不难  
只是眼睛图层和老鼠图层围着各自的中心点旋转 且拥有相同的运动周期 这样能让他们在相同时间内移动的弧度是相同的 产生眼睛跟着老鼠走的感觉

![](/img/2015/11/SCCatWaitingHUD/circle.png)
  
### 最后是眼皮和眼珠的运动协调  
由于眼珠做的是匀速圆周运动 而眼皮做的是直线缩放运动 因此如何协调这两者的运动时间曲线 决定了最后的动画是否流畅和自然  
### 还有一点额外的临时产生的需求就是对横屏的支持  
这是我在写的过程中@sauchye提出的issue，所以我花了一个版本用来加上了对左右横屏的支持


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

Xcode7之后，苹果要求所有的UIWindow在声明的时候都需要有一个rootViewController，即通过代码声明的时候，需要定义一个rootViewController，然后在这个controller之上添加要显示的内容。但是经过验证，在程序运行中创建的非根Window的UIWindow，可以直接当做UIView来使用，仍然不需要强制给一个rootViewController。
self.backgroundWindow = [[UIWindow alloc]initWithFrame:self.frame];
_backgroundWindow.windowLevel = UIWindowLevelStatusBar;
_backgroundWindow.backgroundColor = [UIColor clearColor];
_backgroundWindow.alpha = 0.0f;

那么怎么控制Window的出现和消失呢  

- (void)makeKeyWindow;
- (void)makeKeyAndVisible;                             // convenience. most apps call this to show the main window and also make it key. otherwise use view hidden property




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




所以我们还需要调整眼珠Layer的position，来和之前设置的anchorPoint的位置重合。

```
_rightEye.layer.anchorPoint = CGPointMake(1.5f, 1.5f);
_rightEye.layer.position = CGPointMake(self.faceView.right - 13.5f, self.faceView.top + width/3.0f + 7.5f);
```

这时候，上面的`CAAnimation`就可以正常的添加到Layer的transform上并且执行出来啦。

### 眼皮的运动时间曲线

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

这时候，这个函数图像就是接下来要给眼皮加上的缩放动画的时间曲线了，我没有用专门的画贝塞尔曲线的软件来画，只是根据函数的图像大致画出了一个贝塞尔曲线，作为了下面这个动画的时间曲线

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


