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
	AutoRotate = 0, //自动动画
	WithFinger, //跟随手指
	WithFingerReverse, //跟随手指 但反向
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
### 2.逻辑判断

```
//JZParallaxButton.h
# import <UIKit/UIKit.h>

typedef NS_ENUM(NSInteger, ParallaxMethodType)
{
	Linear = 0,
	EaseIn,
	EaseOut,
};

@interface JZParallaxButton : UIButton

//当前Button包含的所有ImageLayer
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
	        case AutoRotate:
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
	
	        case WithFinger:
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
    	//半径和图层层级成线性关系
        case Linear:
            return (float)(i)/(float)([Array count]);
            break;
        //半径和图层层级成二次关系
        case EaseIn:
            return powf((float)(i)/(float)([Array count]), 0.5f);
            break;
    	//半径和图层层级成根号关系
        case EaseOut:
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
        case Linear:
            return 1+ScaleAddition/10*((float)i/(float)([LayerArray count]));
            break;
            
        case EaseIn:
            return 1+ScaleAddition/10*powf(((float)i/(float)([LayerArray count])), 0.5f);
            break;
            
        case EaseOut:
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
# 资源  
配图所用Sketch文件：[下载链接](/files/2015/11/tvOS/tvOS_Sketch.sketch)


[1]:	https://developer.apple.com/tvos/human-interface-guidelines/icons-and-images/
[2]:	/files/2015/11/tvOS/tvOS-Gear.zip

[image-1]:	/img/2015/11/tvOS/tvOSButtonPic_1.jpg
[image-2]:	/img/2015/11/tvOS/tvOSButtonPic_2.jpg
[image-3]:	/img/2015/11/tvOS/tvOSButtonPic_3.jpg
[image-4]:	/img/2015/11/tvOS/tvOSButtonGif_1.gif
[image-5]:	/img/2015/11/tvOS/tvOSButtonGif_2.gif