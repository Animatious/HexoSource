title: tvOS视差按钮的ObjC实现
date: 2015-11-18 20:31:54
tags:
- tvOS 
- ObjC
categories:
- 代码

---

 
![](/img/2015/11/tvOS/tvOSButtonPic_thumb.jpg)
# 介绍
苹果在最新发布的Apple TV里引入了有趣的[图标设计][1]  
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
假设我们已经有了以下图片：（你可以从[下载链接][2]下载已经分层的四张图片）  
![四张已经分层的图片][image-1]  
基于这四张图片 我们该怎么对其进行变换来达到tvOS的视差效果呢？  
重新观察上文中苹果官方的例子视频 我们可以得出以下结论：  
### 1. 总图层在旋转 但不同于一般在屏幕平面上的旋转 而是相对屏幕具有一定夹角的旋转  
如果不太了解这种旋转是怎么发生的 我们可以看一张有关 CATransform3D 的图：
![CATransform3DMakeRotation的XYZ轴][image-2]  
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
### 2.除去旋转 每个图层都在进行不同半径的圆周移动  
为什么有移动？  
透视是创造三维深度感觉的关键 而透视效果最直白的话来说 就是“近大远小”  
让我们来看个例子吧 :-)  
![近大远小的说明][image-3]  
如果只有总图层自转 分图层不进行移动 那么整个按钮虽然有自转效果 但是看起来还是平的  
如果要保证有三维效果 就要有视差 即近大远小 让**远处的图层移动的距离很小 近处的图层移动距离很大**（大家可以自行想象同样速度远处近处的汽车 看起来移动的距离也不一样）  
因此 就要令分图层进行圆周移动 离我们近的图层 圆周半径要更大些 保证看起来移动的距离更大  
我们简单地用 Principle 做了一个[原型](/files/2015/11/tvOS/tvOS_Principle_1.prd) 大体效果应该是这个样子的 中间的圆点是移动的轴心  
![简陋的圆周移动效果][image-4]
### 3.总的图层也会移动
看到我们刚才那个简陋的效果了没？你有可能会想 为什么看起来和tvOS差别那么大？  
原因是 tvOS实现的效果 整个按钮并没有明显地移动 而是近似于固定在某个位置的 这么一来 就要求我们在分图层移动的时候 总图层叠加一项反方向的移动 保证按钮固定住  
于是我们又用 Principle 做了一个[原型](/files/2015/11/tvOS/tvOS_Principle_2.prd) 大体效果应该是这个样子的  
![还是很简陋的圆周移动效果][image-5]  

### 4.高光的移动方向恰好相反  
高光就是我们在tvOS的图标上看到的白色反光 这个部分其实很简单：  
用PS画一个白色的圆 加上模糊效果 就是一个 高光图层  
让图层在移动的时候于其他图层方向相反 即让图层叠加之后的效果为 高光永远在离我们最近的地方 这里说起来会有点困惑 但是用代码实现的时候就自然明白了 ^_^
# 实现
> 注：实现部分限于篇幅 不可能将所有代码都粘贴出来 只是在几个关键的地方粘贴出来加以说明  
> 完整代码见 https://github.com/JustinFincher/JZtvOSParallaxButton  

### 1.按钮基本  
按照我们的计划 这个按钮默认并没有三维效果 就是很多UIImage叠加起来 只有当我们长按的时候 才会有各种旋转和移动  
这里动画方式分为两种 第一种是自动动画 首先会移动和旋转到一个特定的角度 然后便开始周期移动了 第二种是手动动画 按钮会跟随手指Drag进行旋转和移动  
让我们先新建一个UIButton吧 :-)  
```
//JZParallaxButton.h
# import <UIKit/UIKit.h>

typedef NS_ENUM(NSInteger, RotateMethodType)
{
	RotateMethodTypeAutoRotate = 0, //自动动画
	RotateMethodTypeWithFinger, //跟随手指
	RotateMethodTypeWithFingerReverse, //跟随手指 但反向
};
@interface JZParallaxButton : UIButton
@end
```
```
//JZParallaxButton.m
# define LongPressInterval 0.5 //自动动画状态下的长按判断时间
@interface UIButton ()<UIGestureRecognizerDelegate>
@end
@implementation JZParallaxButton
@end
```
写一个自己的init方法 然后在里面加入长按的手势判断  
```
//JZParallaxButton.m
- (instancetype)initButtonWithCGRect:(CGRect)RectInfo
		              WithLayerArray:(NSMutableArray *)ArrayOfLayer
		      WithRoundCornerEnabled:(BOOL)isRoundCorner
		   WithCornerRadiusifEnabled:(CGFloat)Radius
		          WithRotationFrames:(int)Frames
		        WithRotationInterval:(CGFloat)Interval
{
	self = [super initWithFrame:RectInfo];
	
	            //...省略...
	
	UILongPressGestureRecognizer *longPress = [[UILongPressGestureRecognizer alloc] initWithTarget:self action:@selector(selfLongPressed:)];
	longPress.delegate = self;
	longPress.minimumPressDuration = LongPressInterval;
	//self就是UIButton了 所以可以对self add
	[self addGestureRecognizer:longPress];
	
	return self;
}
```
```
//JZParallaxButton.m
//长按会触发的方法
- (void)selfLongPressed:(UILongPressGestureRecognizer *)sender
{
	CGPoint PressedPoint = [sender locationInView:self];
	NSLog(@"%F , %F",PressedPoint.x,PressedPoint.y);    
	//可以读取当前按下的位置
}
```
### 2.层级关系和逻辑判断
我们的按钮在实现后应有以下层级：
![层级关系](/img/2015/11/tvOS/tvOSButtonPic_4.jpg)
```
ParallaxButton:UIButton  //我们建立的UIButton SubClass
|-BoundsView:UIView   //总视图 
  |--LayerImageView1:UIImageView //分视图 是总视图的SubView
  |--LayerImageView2:UIImageView
  |--LayerImageView3:UIImageView
  |--LayerImageView4:UIImageView
  |--LayerImageView5:UIImageView
  |-- .... 
  |--LayerImageViewX:UIImageView
```
有的同学有可能会觉得 为什么需要总视图这个 `BoundsView` 呢 直接将所有的UIImageView都划归为UIButton的SubView不就好了么？  
实验过直接将UIImageView添加到UIButton为SubView后 我有一个相对合理的解释：  
我们之前分析原理的时候 说明其实是只有总图层（即 `BoundsView` ）在旋转的 分图层只需处理移动问题  
如果去除了总图层 就只能让每个分图层（即 `LayerImageView` ）在移动的同时都旋转 这势必带来一个问题 那就是会有“厚度”的感觉   
让我们实验下 如果层级关系如下图 会是什么结果
![错误的显示效果](/img/2015/11/tvOS/tvOSButtonGif_3.gif)  
可以看到 这里的图层移动方式和原型里的效果已经很接近了 但是因为分图层移动半径不一 并且没有总图层进行约束 导致分图层的显示区域不在一个长方形里 看起来像是有厚度了一样 整个按钮实际看起来并没有tvOS按钮里的那种轻盈感  
因此 需要有总图层进行约束 即将分图层添加为总图层的SubView 并设置总图层Layer的MasksToBounds为YES 这时 所有分图层的可见区域都受限制于总图层 无论怎么旋转和移动都不会出现厚度感了  
我们现在将视图层级需要的一些变量写出来 然后再实现一些逻辑判断的代码 比如长按后需要做什么  
```
//JZParallaxButton.h
# import <UIKit/UIKit.h>

typedef NS_ENUM(NSInteger, ParallaxMethodType)
{
	ParallaxMethodTypeLinear = 0,
	ParallaxMethodTypeEaseIn,
	ParallaxMethodTypeEaseOut,
};

@interface JZParallaxButton : UIButton

//数组用于记录当前Button包含的所有ImageLayer 即分图层
@property (nonatomic,strong) NSMutableArray *LayerArray;

//button圆角
@property (nonatomic,assign) BOOL RoundCornerEnabled;

//button圆角
@property (nonatomic,assign) CGFloat RoundCornerRadius;

//是否在Parallax
@property (nonatomic,assign) BOOL isParallax;

@property (nonatomic,assign) int RotationAllSteps;
@property (nonatomic,assign) CGFloat RotationInterval;

@property (nonatomic,assign) CGFloat ScaleBase;
@property (nonatomic,assign) CGFloat ScaleAddition;

@property (nonatomic,assign) ParallaxMethodType ParallaxMethod;
@property (nonatomic,assign) RotateMethodType RotateMethod;

- (instancetype)initButtonWithCGRect:(CGRect)RectInfo
		              WithLayerArray:(NSMutableArray *)ArrayOfLayer
		      WithRoundCornerEnabled:(BOOL)isRoundCorner
		   WithCornerRadiusifEnabled:(CGFloat)Radius
		          WithRotationFrames:(int)Frames
		        WithRotationInterval:(CGFloat)Interval;

```
```
//JZParallaxButton.m

# define RotateParameter 0.5 //用于调整旋转幅度
# define SpotlightOutRange 20.0f //用于高光距离中心的长度
# define zPositionMax 800 //Core Layer变换时摄像机默认z轴位置

# define BoundsVieTranslation 50 //总图层的移动幅度
# define LayerVieTranslation 20 //分图层的移动幅度
# define LongPressInterval 0.5 //自动动画状态下的长按判断时间  

@interface UIButton ()<UIGestureRecognizerDelegate>

@property (nonatomic,assign) int RotationNowStep; //记录动画的当前状态
@property (nonatomic,weak)NSTimer *RotationTimer; //动画计时器
@property (nonatomic,strong) UIImageView *SpotLightView;  //高光图层
@property (nonatomic,strong) UIView *BoundsView; //总图层
@property (nonatomic,assign) CGPoint TouchPointInSelf; //手指按下的时候在Button内部 的位置 用于Button设为跟随手指的时候
@property (nonatomic,assign) BOOL hasPreformedBeginAnimation; //判断是否在进行动画 防止动画未表演完就触发下一个动作 造成错位
@end

@implementation JZParallaxButton
//省略 @synthesize ...

- (instancetype)initButtonWithCGRect:(CGRect)RectInfo
                      WithLayerArray:(NSMutableArray *)ArrayOfLayer
              WithRoundCornerEnabled:(BOOL)isRoundCorner
           WithCornerRadiusifEnabled:(CGFloat)Radius
                  WithRotationFrames:(int)Frames
                WithRotationInterval:(CGFloat)Interval
{
//省略之前的代码....
	LayerArray = [[NSMutableArray alloc] initWithCapacity:[ArrayOfLayer count]];
    
    BoundsView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, self.frame.size.width, self.frame.size.height)];
    BoundsView.layer.masksToBounds = YES;
    BoundsView.layer.shouldRasterize = TRUE;
    BoundsView.layer.rasterizationScale = [[UIScreen mainScreen] scale];
    if (self.RoundCornerEnabled)
    {
        BoundsView.layer.cornerRadius = self.RoundCornerRadius;
    }
    [self addSubview:BoundsView];
    
    
    for (int i = 0; i < [ArrayOfLayer count]; i++)
    {
        UIImageView *LayerImageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, self.frame.size.width, self.frame.size.height)];
        UIImage *LayerImage = [ArrayOfLayer objectAtIndex:i];
        [LayerImageView setImage:LayerImage];
        LayerImageView.layer.shouldRasterize = TRUE;
        LayerImageView.layer.rasterizationScale = [[UIScreen mainScreen] scale];
        
        //从下往上添加
        [BoundsView addSubview:LayerImageView];
        [LayerArray addObject:LayerImageView];
        
        //如果把所有分图层都加完了
        if (i == [ArrayOfLayer count] - 1)
        {
            //在最上层添加高光分图层
            SpotLightView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, self.frame.size.width,self.frame.size.height)];
            
            NSString *bundlePath = [[NSBundle bundleForClass:[JZParallaxButton class]]
                                    pathForResource:@"JZParallaxButton" ofType:@"bundle"];
            NSBundle *bundle = [NSBundle bundleWithPath:bundlePath];
            
            SpotLightView.image = [UIImage imageNamed:@"Spotlight" inBundle:bundle compatibleWithTraitCollection:nil];
            SpotLightView.contentMode = UIViewContentModeScaleAspectFit;
            SpotLightView.layer.masksToBounds = YES;
            [BoundsView addSubview:SpotLightView];
            SpotLightView.layer.zPosition = zPositionMax;
            [self bringSubviewToFront:SpotLightView];
            SpotLightView.alpha = 0.0;
            [LayerArray addObject:SpotLightView];
        }
}              
- (void)selfLongPressed:(UILongPressGestureRecognizer *)sender
{
	CGPoint PressedPoint = [sender locationInView:self];
	//NSLog(@"%F , %F",PressedPoint.x,PressedPoint.y);
	self.TouchPointInSelf = PressedPoint;
	
	if(sender.state == UIGestureRecognizerStateBegan)
	{
	    //NSLog(@"Long Press");
	    self.hasPreformedBeginAnimation = NO;
	
	    switch (self.RotateMethod)
	    {
	        case RotateMethodTypeAutoRotate:
	        {
	        //长按后 如果在进行视差效果就结束 如果现在是普通状态就开启视差效果
	            if (isParallax)
	            {
	                [self EndParallax];
	            }
	            else
	            {
	                [self BeginParallax];
	            }
	        }
	            break;
	
	        case RotateMethodTypeWithFinger:
	        {
	        //手动动画结束视差效果并不依靠长按 而是通过判断手指是否留在屏幕上
	            if (!isParallax)
	            {
	                [self BeginParallax];
	            }
	        }
	            break;
	    }
	}
	else if (sender.state == UIGestureRecognizerStateEnded)
	{
	//如果长按结束但仍有动画在进行 就等待 LongPressInterval 再执行TouchUp方法 否则立即执行TouchUp
	    if (self.hasPreformedBeginAnimation == NO)
	    {
	        [self performSelector:@selector(TouchUp) withObject:self afterDelay:LongPressInterval ];
	    }
	    else
	    {
	        [self TouchUp];
	    }
	
	}
}
@end

```
这里的 `shouldRasterize` ：
```
@property BOOL shouldRasterize
Description	A Boolean that indicates whether the layer is rendered as a bitmap before compositing. Animatable
```
在某些时候 例如导入的图片分辨率较大 此时对UIViewImage进行CA动画 会出现锯齿 这个时候 可以通过设置 `shouldRasterize` 来解决
```
BoundsView.layer.shouldRasterize = YES;
BoundsView.layer.rasterizationScale = [[UIScreen mainScreen] scale];
```
当然设置 `allowsEdgeAntialiasing` 也是一种方法 这里我们对比下 可以注意看右边墙壁上的树影 通过 `shouldRasterize` 设置后 基本上影子不会出现较大的变动：
![两种抗锯齿的对比](/img/2015/11/tvOS/tvOSButtonGif_4.gif)


### 3.透视效果需要实现的一个方法  
```
//JZParallaxButton.m
//CATransform3DMake 默认给出的CATransform3D是没有透视效果的 需要手动加入这一段 完成从 orthographic 到 perspective 的改变
//这个方法的解析可以看 http://wonderffee.github.io/blog/2013/10/19/an-analysis-for-transform-samples-of-calayer/
CATransform3D CATransform3DMakePerspective(CGPoint center, float disZ)
{
CATransform3D transToCenter = CATransform3DMakeTranslation(-center.x, -center.y, 0);
CATransform3D transBack = CATransform3DMakeTranslation(center.x, center.y, 0);
CATransform3D scale = CATransform3DIdentity;
scale.m34 = -1.0f/disZ;
return CATransform3DConcat(CATransform3DConcat(transToCenter, scale), transBack);
}

CATransform3D CATransform3DPerspect(CATransform3D t, CGPoint center, float disZ)
{
return CATransform3DConcat(t, CATransform3DMakePerspective(center, disZ));
}
```
### 4.实现自动动画  
这部分主要内容就是 在长按后 按钮会先进入某个特定的角度位置 然后进行自转  
这里为了简单 使用了NSTimer进行每一帧的计数 但需要注意的是 NSTimer的精度不足以完成真正流畅的动画  
自动动画里 总图层和分图层的移动 旋转都和两个参数有关：一是当前的计数位置（即Step） 而是图层在总按钮里的层级位置(即LayerArray里的i) 通过这两个参数进行计算CATransform3D
```
//  JZParallaxButton.m

#define OutTranslationParameter  (float)([LayerArray count] + i)/(float)([LayerArray count] * 2)
#define OutScaleParameter  ScaleBase+ScaleAddition/5*((float)i/(float)([LayerArray count]))

@implementation JZParallaxButton

- (void)BeginAutoRotation
{
    __weak JZParallaxButton *weakSelf = self;
    //需要到达动画的起始位置
    
    CGFloat PIE = 0;
    CGFloat Degress = M_PI*2*PIE;
    //NSlog(@"Degress : %f PIE",PIE);
    CGFloat Sin = sin(Degress)/4;
    CGFloat Cos = cos(Degress)/4;
    
    int i =0;
    //计算初始位置的旋转 移动 和缩放
    CATransform3D NewRotate,NewTranslation,NewScale;
    NewRotate = CATransform3DConcat(CATransform3DMakeRotation(-Sin*RotateParameter, 0, 1, 0), CATransform3DMakeRotation(Cos*RotateParameter, 1, 0, 0));
    NewTranslation = CATransform3DMakeTranslation(Sin*BoundsVieTranslation*OutTranslationParameter, Cos*BoundsVieTranslation*OutTranslationParameter, 0);
    NewScale = CATransform3DMakeScale(OutScaleParameter, OutScaleParameter, 1);
    
    //使用CATransform3DConcat将三个CATransform3D结合成一个CATransform3D
    CATransform3D TwoTransform = CATransform3DConcat(NewRotate,NewTranslation);
    CATransform3D AllTransform = CATransform3DConcat(TwoTransform,NewScale);
    //对BoundsLayer 即总图层进行动画
    CABasicAnimation *BoundsLayerCABasicAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
    BoundsLayerCABasicAnimation.duration = 0.4f;
    BoundsLayerCABasicAnimation.autoreverses = NO;
    BoundsLayerCABasicAnimation.toValue = [NSValue valueWithCATransform3D:CATransform3DPerspect(AllTransform, CGPointMake(0, 0), zPositionMax)];
    BoundsLayerCABasicAnimation.fromValue = [NSValue valueWithCATransform3D:BoundsView.layer.transform];
    BoundsLayerCABasicAnimation.fillMode = kCAFillModeBoth;
    BoundsLayerCABasicAnimation.removedOnCompletion = YES;
    [BoundsView.layer addAnimation:BoundsLayerCABasicAnimation forKey:@"BoundsLayerCABasicAnimation"];
    BoundsView.layer.transform = CATransform3DPerspect(AllTransform, CGPointMake(0, 0), zPositionMax);
    
    //对LayerArray内的UIImageView 即分图层进行动画
    for (int i = 0 ; i < [LayerArray count]; i++)
    {
        //对于不同的前后位置 需要移动的半径不一样
        float InTranslationParameter = [self InTranslationParameterWithLayerArray:LayerArray WithIndex:i];
        float InScaleParameter = [self InScaleParameterWithLayerArray:LayerArray WithIndex:i];
        UIImageView *LayerImageView = [LayerArray objectAtIndex:i];

        CATransform3D NewTranslation ;
        CATransform3D NewScale = CATransform3DMakeScale(InScaleParameter, InScaleParameter, 1);
        
        if (i == [LayerArray count] - 1) //是高光所在的View
        {
            NewTranslation = CATransform3DMakeTranslation(Sin*LayerVieTranslation*InTranslationParameter*SpotlightOutRange, Cos*LayerVieTranslation*InTranslationParameter*SpotlightOutRange, 0);
        }
        else //分图层其他的View 注意可以看到这里的 Translation 和高光是相反的
        {
            NewTranslation = CATransform3DMakeTranslation(-Sin*LayerVieTranslation*InTranslationParameter, -Cos*LayerVieTranslation*InTranslationParameter, 0);
        }
        
        CATransform3D NewAllTransform = CATransform3DConcat(NewTranslation,NewScale);
        
        CABasicAnimation *LayerImageViewCABasicAnimation = [CABasicAnimation animationWithKeyPath:@"transform"];
        LayerImageViewCABasicAnimation.duration = 0.4f;
        LayerImageViewCABasicAnimation.autoreverses = NO;
        LayerImageViewCABasicAnimation.toValue = [NSValue valueWithCATransform3D:NewAllTransform];
        LayerImageViewCABasicAnimation.fromValue = [NSValue valueWithCATransform3D:LayerImageView.layer.transform];
        LayerImageViewCABasicAnimation.fillMode = kCAFillModeBoth;
        LayerImageViewCABasicAnimation.removedOnCompletion = YES;
        
        
        CAAnimationGroup *animGroup = [CAAnimationGroup animation];
        animGroup.animations = [NSArray arrayWithObjects:LayerImageViewCABasicAnimation,nil];
        animGroup.duration = 0.4f;
        animGroup.removedOnCompletion = YES;
        animGroup.autoreverses = NO;
        animGroup.fillMode = kCAFillModeRemoved;
        
        [CATransaction begin];
        LayerImageView.layer.transform = CATransform3DPerspect(NewAllTransform, CGPointMake(0, 0), zPositionMax);
        [CATransaction setCompletionBlock:^
         {
             if (i == [LayerArray count] - 1)
             {
                 //简单的周期循环
                 RotationNowStep = 0;
                 RotationTimer =  [NSTimer scheduledTimerWithTimeInterval:RotationInterval/RotationAllSteps target:weakSelf selector:@selector(RotationCreator) userInfo:nil repeats:YES];
                 weakSelf.hasPreformedBeginAnimation = YES;
             }
         }];
        [LayerImageView.layer addAnimation:animGroup forKey:@"LayerImageViewParallaxInitAnimation"];
        [CATransaction commit];
    }

    
}

//计时器每次触发要执行的方法
- (void)RotationCreator
{
    __weak JZParallaxButton *weakSelf = self;
 
    //NSlog(@"RotationNowStep : %d of %d",RotationNowStep,RotationAllSteps);
    if (RotationNowStep == RotationAllSteps)
    {
    //一周完成 计数器置1
        RotationNowStep = 1;
    }
    else
    {
        RotationNowStep ++ ;
    }
    
    CGFloat PIE = (float)RotationNowStep/(float)RotationAllSteps;
    CGFloat Degress = M_PI*2*PIE;
    //NSlog(@"Degress : %f PIE",PIE);
    CGFloat Sin = sin(Degress)/4;
    CGFloat Cos = cos(Degress)/4;
    
    int i = 0;
    CATransform3D NewRotate,NewTranslation,NewScale;
    NewRotate = CATransform3DConcat(CATransform3DMakeRotation(-Sin*RotateParameter, 0, 1, 0), CATransform3DMakeRotation(Cos*RotateParameter, 1, 0, 0));
    NewTranslation = CATransform3DMakeTranslation(Sin*BoundsVieTranslation*OutTranslationParameter, Cos*BoundsVieTranslation*OutTranslationParameter, 0);
    NewScale = CATransform3DMakeScale(OutScaleParameter, OutScaleParameter, 1);
    
    CATransform3D TwoTransform = CATransform3DConcat(NewRotate,NewTranslation);
    CATransform3D AllTransform = CATransform3DConcat(TwoTransform,NewScale);
    BoundsView.layer.transform = CATransform3DPerspect(AllTransform, CGPointMake(0, 0), zPositionMax);
    
    for (int i = 0 ; i < [LayerArray count]; i++)
    {
        float InScaleParameter = [self InScaleParameterWithLayerArray:LayerArray WithIndex:i];
        float InTranslationParameter = [self InTranslationParameterWithLayerArray:LayerArray WithIndex:i];
        
        
        if (i == [LayerArray count] - 1) //is spotlight
        {
            UIImageView *LayerImageView = [weakSelf.LayerArray objectAtIndex:i];
            
            CATransform3D Translation = CATransform3DMakeTranslation(Sin*LayerVieTranslation*InTranslationParameter*SpotlightOutRange, Cos*LayerVieTranslation*InTranslationParameter*SpotlightOutRange,0);
            CATransform3D Scale = CATransform3DMakeScale(InScaleParameter, InScaleParameter, 1);
            CATransform3D AllTransform = CATransform3DConcat(Translation,Scale);
            
            LayerImageView.layer.transform = CATransform3DPerspect(AllTransform, CGPointMake(0, 0), zPositionMax);
        }
        else //is Parallax layer
        {
            UIImageView *LayerImageView = [weakSelf.LayerArray objectAtIndex:i];
            
            CATransform3D Translation = CATransform3DMakeTranslation(-Sin*LayerVieTranslation*InTranslationParameter, -Cos*LayerVieTranslation*InTranslationParameter, 0);
            CATransform3D Scale = CATransform3DMakeScale(InScaleParameter, InScaleParameter, 1);
            CATransform3D AllTransform = CATransform3DConcat(Translation,Scale);
            
            LayerImageView.layer.transform = CATransform3DPerspect(AllTransform, CGPointMake(0, 0), zPositionMax);
        }
        
    }

    
    
}

- (float)InTranslationParameterWithLayerArray:(NSMutableArray *)Array
                                    WithIndex:(int)i
{

    switch (ParallaxMethod)
    {
    	//移动半径和图层层级成线性关系
        case ParallaxMethodTypeLinear:
            return (float)(i)/(float)([Array count]);
            break;
        //移动半径和图层层级成二次关系
        case ParallaxMethodTypeEaseIn:
            return powf((float)(i)/(float)([Array count]), 0.5f);
            break;
    	//移动半径和图层层级成根号关系
        case ParallaxMethodTypeEaseOut:
            return powf((float)(i)/(float)([Array count]), 2.0f);
            break;
            
        default:
            return (float)(i)/(float)([Array count]);
            break;
    }
}
- (float)InScaleParameterWithLayerArray:(NSMutableArray *)Array
                                    WithIndex:(int)i
{
    
    switch (ParallaxMethod)
    {
    //缩放与图层层级的不同关系
        case ParallaxMethodTypeLinear:
            return 1+ScaleAddition/10*((float)i/(float)([LayerArray count]));
            break;
            
        case ParallaxMethodTypeEaseIn:
            return 1+ScaleAddition/10*powf(((float)i/(float)([LayerArray count])), 0.5f);
            break;
            
        case ParallaxMethodTypeEaseOut:
            return 1+ScaleAddition/10*powf(((float)i/(float)([LayerArray count])), 2.0f);
            break;
            
        default:
            return 1+ScaleAddition/10*((float)i/(float)([LayerArray count]));
            break;
    }
}
@end

```

### 4.实现手动动画
```  
//手动动画和自动动画的区别是 移动的角度不再跟进计数器计算 而是直接读取手指的CGPoint
__weak JZParallaxButton *weakSelf = self;
    CGFloat XOffest;
    if (TouchPointInSelf.x < 0)
    {
        XOffest = - weakSelf.frame.size.width / 2;
    }else if (TouchPointInSelf.x > weakSelf.frame.size.width)
    {
        XOffest = weakSelf.frame.size.width / 2;
    }else
    {
        XOffest = TouchPointInSelf.x - weakSelf.frame.size.width / 2;
    }
    
    CGFloat YOffest;
    if (TouchPointInSelf.y < 0)
    {
        YOffest = - weakSelf.frame.size.height / 2;
    }else if (TouchPointInSelf.y > weakSelf.frame.size.height)
    {
        YOffest = weakSelf.frame.size.height / 2;
    }else
    {
        YOffest = TouchPointInSelf.y - weakSelf.frame.size.height / 2;
    }
    
    //NSLog(@"XOffest : %f , YOffest : %f",XOffest,YOffest);
    
    CGFloat XDegress = XOffest / weakSelf.frame.size.width / 2;
    CGFloat YDegress = YOffest / weakSelf.frame.size.height / 2;
    
    //NSLog(@"XDegress : %f , YDegress : %f",XDegress,YDegress);

```
# 效果  
此时还有很多方法没有实现（比如动画结束后需要变回原有的非三维效果） 不过大体上已经可以看到效果了 你也可以直接将完成版的[视差按钮下载下来](https://github.com/JustinFincher/JZtvOSParallaxButton)    
Have Fun :-)
# 其他资源  
如果你对真正三维状态下的按钮还是不太理解 可以点击下面的播放按钮自由查看这个三维模型 点击播放按钮后 通过三维场景右下角的左右切换查看按钮的不同旋转状态 或者点击数字1-5来快速跳转  
<iframe width="640" height="480" src="https://sketchfab.com/models/899bdd009cc64af0b2260fe7570a0510/embed" frameborder="0" allowfullscreen mozallowfullscreen="true" webkitallowfullscreen="true" onmousewheel=""></iframe>

以外 这篇文章里所有文件都是提供公共下载的   
配图所用Sketch文件：[下载链接](/files/2015/11/tvOS/tvOS_Sketch.sketch)
三维模型文件（进入点击Download）：[下载链接](https://sketchfab.com/models/899bdd009cc64af0b2260fe7570a0510)

[1]:	https://developer.apple.com/tvos/human-interface-guidelines/icons-and-images/
[2]:	/files/2015/11/tvOS/tvOS-Gear.zip

[image-1]:	/img/2015/11/tvOS/tvOSButtonPic_1.jpg
[image-2]:	/img/2015/11/tvOS/tvOSButtonPic_2.jpg
[image-3]:	/img/2015/11/tvOS/tvOSButtonPic_3.jpg
[image-4]:	/img/2015/11/tvOS/tvOSButtonGif_1.gif
[image-5]:	/img/2015/11/tvOS/tvOSButtonGif_2.gif