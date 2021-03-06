title: Epic-Black-Friday-Deals 效果的前端技术实现
date: 2015-11-28 21:25:40
tags:
- canvas
- web
categories:
- 代码

---
# 介绍

这是一个通过 web 技术实现的效果演示 demo 。

原设计链接： [Epic-Black-Friday-Deals](https://dribbble.com/shots/2372734-Epic-Black-Friday-Deals)

原效果图：

<img width="400px" height="300px" src="https://d13yacurqjgara.cloudfront.net/users/107759/screenshots/2372734/ink2.gif" alt="Ink2">

Demo 链接： https://chemzqm.github.io/dribbble-effects/friday.html

注：使用 safari 保存到桌面浏览效果更佳。

整个效果分为两个部分实现，上半部分通过 canvas 不断绘制实现，下半部分使用了 css 的 transform 和 transition 来实现。 下面是详细介绍。

## 基于 canvas 实现的上半部分动画

### 关于canvas

canvas 是使用 javascript 进行绘图的基本 API, 它本身只提供了基础的画图 API （例如直线、弧线、曲线、矩形、填充等），通过学习[MDN提供的教程](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial), 一个具备 javascript 基本知识的开发者可以很快掌握。

canvas 并不提供动画，事件等更高层的 API，所以需要相应功能都需要自行计算，或者借助其它库来实现。

尽管抽象性很低（或者说非常底层）但是相应的好处是可塑性比较好，因为开发者可以精确的控制每一个细节如何完成。

### canvas 动画实现

canvas 本身是静态的，并不提供动画，所以我们借助 API [requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame)， 通过重复的调用这个 API ， 不断的清空画布，并借助其返回的 timestamp 时间戳计算绘制所需的参数值，就可以实现动画效果了， 代码示例：

``` js
var raf = window.requestAnimationFrame
var start
function animate(timestamp) {
	if(!start) start = timestamp
	//通过持续时间不同重绘
	drawImage(timestamp - start)
	raf(animate)
}
raf(animate)
```

当然你也能用 setInterval 这样的 API 实现，但是这么做性能太差，比如当前页面隐藏的时候 requestAnimationFrame 是不会触发调用的，这样就节省了客户端的资源。

### 实现缓动效果

缓动效果可以让你的动画更柔和自然，[easings.net](http://easings.net/zh-cn)上可以看到已经命名的缓动函数，其实质就是把一个 0 到 1 之间的值进行转化。 [component/ease](https://github.com/component/ease/blob/master/index.js) 是一个实现了缓动函数的 javascript 库，我们拿它的一个函数 `inQuad` 举例：

``` js
function inQuad(n){
  return n * n;
}
var duration = 600 //动画持续 600 ms
var raf = window.requestAnimationFrame
var start
function animate(timestamp) {
	if(!start) start = timestamp
	var percent = (timestamp - start)/duration
	// 转化为缓动后的值
	percent = inQuad(percent)
	drawImage(percent)
	raf(animate)
}
raf(animate)
```
这样我们的 `drawImage`接收到的就是缓动后的百分比了。

尽管 `drawImage` 接收到了百分比，但是具体的值很是需要自己计算，如果你想省事可以借助 [component/tween](https://github.com/component/tween) 之类的库来帮你计算属性值，事实上后面要讲到的日期选择的滑动效果就是借助 tween 来实现的.

## keyframe(关键帧) 的实现

keyframe（关键帧）是一种对动画控制非常有用的抽象，比如说一个画一个圈的起始和终止就各自对应一个 keyframe，css animation 就是通过控制 keyframe 来实现动画的 API。 canvas 也有类似的库来实现 keyframe，例如 [rekapi](https://github.com/jeremyckahn/rekapi)。 因为这次的动画并不需要 playback 支持（或者说我比较懒），所以这次只是简单的通过状态来实现了不同效果的切换，代码大致如下：

``` js
function View() {
	//负责圆圈的绘制
	this.circle = new Circle(this)
	//负责中间图标绘制
	this.icon = new Icon(this)
	//负责时间文本的绘制
	this.time = new Time(this)
	this.stat = 'stopped'
}
//展开状态
View.prototype.pend = function() {
	this.stat = 'pending'
}
//中间对勾效果
View.prototype.check= function() {
	this.stat = 'checking'
}
//重置为展开状态
View.prototype.reset = function() {
	this.stat = 'reseting'
}
//主绘制函数
View.prototype.draw = function(){
	this.circle.draw()
	this.time.draw()
	this.icon.draw()
}

//每个子模块通过判定 view 的 stat 绘制
function Circle(view){
	this.view = view
}
Circle.prototype.draw = function(){
	var stat = this.view.stat
	switch (stat) {
		case 'pending':
			this.pend()
		case 'checking':
			this.check()
		case 'reseting':
			this.reset()
	}
}
// time 模块和 icon 模块同上
```

这种写法比较方便省事，但是抽象性比较差，建议需要灵活控制的话还是找一个适合的库来辅助。

## 动画流程控制

因为没有实现 keyframe ，所以流程控制（例如取消动画和动画连续）也得自己来实现了。 可选的办法有 callback 回调，事件，Promise等，这次使用了 Promise 实现，因为使用起来比较简洁，而且可以很容易实现取消操作。

通过让 Promise reject 来取消原来动画流程：

``` js
View.prototype.cancel = function () {
  this.promise = this.promise || Promise.resolve(null)
  this.canceled = true
  var self = this
  return this.promise.then(function () {
    // 当前 promise 已经执行完毕
    self.canceled = false
  }, function () {
    // reject 表明成功结束
    self.canceled = false
  })
}

View.prototype.animate = function () {
  var duration = this.duration
  var start
  var self = this
  var promise = this.promise = new Promise(function (resolve, reject) {
    // raf 调用的绘制主进程
    function step(timestamp) {
      // transform 重置
      // 因为使用了 https://github.com/component/autoscale-canvas/blob/master/index.js 通过 scale 支持 Retina，
      // 所以这里不能简单的把 scale 设为 1
      self.ctx.setTransform(window.devicePixelRatio || 1 ,0 ,0 ,window.devicePixelRatio || 1 ,0, 0);
      // 清空画布
      self.ctx.clearRect(0, 0, self.width, self.height)
      // 停止动画并且 reject
      if (self.canceled === true) {
        return reject()
      }
      if (!start) start = timestamp
      var d = timestamp - start
      // 回着不同模块
      self.draw()
      // 成功结束
      if (d > duration) {
        return resolve()
      }
      raf(step)
    }
    raf(step)
  })
  return promise
}
```

调用 `view.cancel()` 就可以取消原来的流程了。

[view模块](https://github.com/chemzqm/dribbble-effects/blob/master/lib/friday/index.js) 的 `cancel` `pend` `reset` 和 `check` 方法都返回了 promise 对象，这样我们需要开始新的流程就可以这样写：

``` js

// 先取消当前动画
view.cancel()
.then(function () {
  // 只有当前是 checked 完成状态才走 reset 流程
  if (view.checked) {
    return view.reset().then(function () {
      // 暂停一下再开始展开
      return view.wait(300)
    })
  }
}).then(function () {
  return view.pend()
}).then(function () {
  return view.check()
}).catch(function () {
  // 因为通过 fail 来终止动画，不捕获会报错
  return true
})
```

### 具体效果实现

* 消逝圆圈的残余尾巴：首先是设定了一个区间，例如：`arr = [1, 2, 2, 1, 1]` 表示有 `arr.length` 个区间，每个区间有 `arr[i]` 个尾巴，每次变换区间都重新生成尾巴：

``` js
step_len = arr.length
Circle.prototype.pend = function () {
  var ctx = this.ctx
  var percent = this.view.percent
  var step = Math.floor(percent*step_len)
  // step 转变，生成新的 tail
  if (step > this.step) {
    this.createTails()
  }
  this.step = step
  // 绘制每一个tail
  this.tails.forEach(function (tail) {
    // 计算 tail 的 percent
    var p = percent*step_len - step
    // tail 根据自己 percent (0 为初始 1为消失)进行绘制
    tail.draw(p)
  })
}

Circle.prototype.createTails = function () {
  var n = this.step
  var num = steps[n]
  var tails = []
  for (var i = 0; i < num; i++) {
    var tail = new Tail(this, i)
    tails.push(tail)
  }
  this.tails = tails
}
```

* 时间动画的计算，这里的展开和重置动画是不同的（暂开时分秒的动画要远快与时，重置时时分秒的速率是相同的），所以也就不能简单的 playback 了，重置时只需要 `percent*total` 就能计算对应的十分秒，展开时需要根据一天的十分秒进行一点计算：
```
function pad(n) {
  return ('0' + String(n)).slice(-2)
}

function toHMS(n) {
  var h = Math.floor(n/3600)
  var m = Math.floor((n - h*3600)/60)
  var s = Math.floor(n%60)
  return {
    h: pad(h),
    m: pad(m),
    s: pad(s)
  }
}
// 1天的秒数
var total = 24*60*60
// 获取 hms
toHMS(n)
```

其实这里有个坑，就是因为秒数变化飞快，有可能 1/60 秒就完成了 0~60 的变化, 这样如果浏览器是 60hz 的话就只会显示相对固定的秒数，例如 `
5x` (x为0 ~ 9), 解决办法就是调整下动画持续时间 😅

* 设置动态文本。这次本来打算使用自定义字体 ProximaNova-Light 来显示时间的，但是发现这个字体的数字不是等宽的，结果就是显示动画的时候数字会一直抖动，改为系统自带字体 `sans-serif` `Helvetica` 就没有这种问题。

* 对勾的实现。本次实现最为复杂的部分，因为涉及角度、位置、长度的精确计算，最后使用了黄金分割，也就是右边比左边 1.618: 1, 结果比较满意。代码如下：

``` js
Icon.prototype.drawCorrect = function (p, tx) {
  var l = 14
  var tl = 2.618*l
  var len = tl*(outBack(p))
  var ctx = this.ctx
  var x = this.x
  var y = this.y
  // 起始 x y
  var sx = x - 10
  var sy = y - 3
  var subtense = Math.min(len, l)
  // 中间点 x y
  var mx = sx + subtense*Math.cos(45*PI/180)
  var my = sy + subtense*Math.sin(45*PI/180)
  tx = tx != null ? tx : - 8*(1 - outBack(p))
  // 偏移量
  ctx.translate(tx, 0)
  ctx.beginPath()
  ctx.moveTo(sx, sy)
  ctx.lineTo(mx, my)
  if (len > l) {
    subtense = len - l
    // 终止 x y
    var ex = mx + subtense*Math.cos(50*PI/180)
    var ey = my - subtense*Math.sin(50*PI/180)
    ctx.lineTo(ex, ey)
  }
  ctx.stroke()
  // 重置 transform
  ctx.setTransform(window.devicePixelRatio || 1 ,0 ,0 ,window.devicePixelRatio || 1 ,0, 0);
}

// 出去一点返回
function outBack(n){
  var s = 1.70158;
  return --n * n * ((s + 1) * n + s) + 1;
}
```

这里几个点的坐标不用重复计算的，但是比较偷懒就没抽象出去🙂

* 创建遮罩。为了实现中间图标的进入和退出效果，我在每次动画的时候都在中间图标的左边或者右边创建了一个与背景色相同的矩形区域，这样图标出去的时候就会被遮罩盖住，看上就就好象图标是在下一层退出的一样。(因为canvas 并没有 z-index)

## 基于 Dom 的下半部分日期选择模块

### 日期状态变化时颜色渐变

只需要用 css 一个 transion 就能实现：

``` css
transition: color 0.5s linear;
```

对应状态切换时改变元素的 className 就可以了，（小技巧: classList API）

### 日期列表触摸滚动实现

通过不断调整元素的 translateX 或者有 translate3d 时调整 translate3d 的 x 值实现：

``` js
var s = el.style
if (has3d) {
  s[transform] = 'translate3d(' + x + 'px, 0, 0)'
} else {
  s[transform] = 'translateX(' + x + 'px)'
}
```

这里相对复杂的一步就是 touchend 时计算之后动画的持续时间、最终位置和缓动函数：

``` js
// 减速度
var deceleration = 0.0004
// 当前滑动速度
var speed = this.speed
var x = this.x
// 限定速度
speed = Math.min(speed, 0.6)
var minX = - (this.total - this.count)*width
// 估算终点
var destination = x + ( speed * speed ) / ( 2 * deceleration ) * ( this.distance < 0 ? -1 : 1 )
// 估算时间
var duration = speed / deceleration
var newX
var ease = 'out-cube'
// 终点超出右边界 重算终点和缓动函数
if (destination > 0) {
  newX = 0
  ease = 'out-back'
// 终点小于左边界 重算终点和缓动函数
} else if (destination < minX) {
  newX = minX
  ease = 'out-back'
}
// 超出边界的话重算持续时间
if (typeof newX === 'number') {
  duration = duration*Math.abs((newX - x + 50)/(destination - x))
  //duration = Math.max(200, duration)
  destination = newX
}
// 当前点在边界外，固定持续时间和缓动函数
if (x > 0 || x < minX) {
  duration = 500
  ease = 'out-circ'
}
// 固定终点为子元素宽度的整数倍
destination = Math.round(destination/width)*width
return {
  x: destination,
  duration: duration,
  ease: ease
}
```

计算完成后把结果传递给 `tween` 对象就可以开始滑动了。

### 日期选择

用到了 [events](https://github.com/component/events) 实现事件代理, 以及[tap-event](https://github.com/chemzqm/tap-event)实现正确的 tap 事件 （都是非常常用的移动端组件）

``` js
// 简单的判定，防止 click 和 tap 事件同时触发
var hastouch = 'ontouchstart' in window

function Footer() {
  this.events = events(el, this)
  if (hastouch) {
    this.events.bind('touchstart li', 'ontap')
  } else {
    this.events.bind('click li')
  }
}

Footer.prototype.onclick = function (e) {
  var li = e.delegateTarget
  var children = this.el.children
  for (var i = 0, l = children.length; i < l; i++) {
    if (li === children[i]) {
      // 高亮选中元素
      li.classList.add('active')
    } else {
      // 移除其它元素高亮
      li.classList.remove('active')
    }
  }
}

Footer.prototype.ontap = tap(Footer.prototype.onclick)
```

### 事件传递

使用的 [component/emitter](https://github.com/component/emitter), 它实现了非常简单的观察者模式（订阅发布模式）

``` js
function Footer(el) {
  ...
}

Emitter(el)

Footer.prototype.active = function (index) {
  this.emit('change',  index)
}

function View() {
  var footer = this.footer = new Footer()
  footer.on('change', function(index) {
    // active 变化的事件代理函数
  })
}
```

这样我们就实现了 `Footer` 模块和 `View` 模块的解耦合

## Q and A

Q: 为什么没有注释?

A: 实现比较仓促，而且也不是实际使用项目，所以比较偷懒，其实我的大部分项目都很严肃，也有比较完整的测试和注释。

Q: 为什么 ios 上保存为 app 后状态栏是白色？

A: 因为这是 ios9 的 bug， 原本可以透明的， [了解更多](https://forums.developer.apple.com/thread/9819)

Q: 为什么手机横过来就不能看了？

A: 因为现在safari 还没有实现锁屏的 API !


## 最后

实现 web 动画还有 [css animation](https://developer.mozilla.org/en/docs/Web/CSS/animation), [webgl](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API) [svg smil](https://developer.mozilla.org/en-US/docs/Web/SVG/SVG_animation_with_SMIL) 等方案，这次并没有用到，以后有机会再为大家介绍。

或许是因为 web 存在很多的技术方案以及相应类库，导致很多做 web 的同学觉得搞特效很复杂，但事实并非如此, 只要你了解了它们基本特点并掌握基本方法，就足够做出不错的效果了，再附送一个更简单的例子：[canvas loadings](https://chemzqm.github.io/loadings/)

希望本文能帮你做出更好的 web 动画效果。

## 关于作者

chemzqm@gmail.com 一个以简洁高效为追求的 Javascript 开发者

Github: https://github.com/chemzqm

weibo: http://weibo.com/chemzqm
