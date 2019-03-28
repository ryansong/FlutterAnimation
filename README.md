# FlutterAnimation
Brief Intro of Flutter animation.

Flutter 的动画系统都是基于类型化的 Animation 对象们. Widgets 可以读取值或者监听变化将动画合并到构建函数里, 或者直接将动画传给其他 widgets 来构建更精细的动画.

# Aimation 类

Animation 类是整个 Flutter 动画的砖石，它表示的是动画生命周期中的更改的特殊类型的值。Widgets 展示动画的时候会接受一个 Animation 对象作为参数，从中可以读取动画当前值以及他们监听对值的改动.

addListerner&addStatusListerner 

Animation 提供了两个监听器方法.一般注册了动画监听的 widgets 会在监听回调中调用 setState 来通知 widget 需要用 Animation 的新值重新创建动画. Animation 提供了 AnimationStatus 来标识动画如何随着时间变动. 每当动画状态 (AnimationStatus)发生变化, Animation 都会通知 addStatusListerner 的所有监听器. Animation 的状态: Animation 未开始时处于 dismissed 状态, 比如动画过程是 从 0.0 到 1.0.

AnimationController 

和文本处理一样, Flutter 需要一个 AnimationController 来控制动画, 比如动画播放的前进后退抑或是停止动画. 创建 AnimationController 之后可以创建 ReverseAnimation, 顾名思义就是逆序播放的动画. 同样的还有 CurvedAnimation , 播放时序可以曲线调整.

使用 ticker 来赋予自己生命. 每个 tick 他需要将启动以来的时间传递给 simulator 来获得值. 如果 simulator 报告到那个时间他已经停止了, 那么 controller 自己就停了.

线性控制动画播放方法 forward reverse play resueme. 在播放区间内线性插值.

repeat 函数会让动画不停地执行线性插值.

animationTo

fling 创建特殊控制器, 驱动 controller

animationWith 给定的 simulator 驱动控制器

这些方法都会返回 tikcer 提供的 Feature ,这个也会解决 controller 下次停止或更改 simulator 时将解决的问题.


动画附加到动画

Tweens 

Flutter 提供了 Tweens<T> 泛型的来提供动画插值. 比如 ColorTween & RectTween. 自定义自己的 Tween 子类(需要覆盖 lerp 函数)就可以创建自己的插值了.
  
 不过 Tween 只是定义了两个值之间要插值, 要获得当前动画的帧的具体值还需要 Animation 来确定当前状态.
 
 
 架构
 
 Scheduler
 每一次屏幕上需要显示一个帧时, Flutter 的引擎都会触发一个"begin frame"回调, Scheduler 将调用所有使用 schedulerFrameCallback 注册的监听器.这些回调都会被给定一个相同的时间, 因此从这些回调触发的任何动画都是完全同步的, 即时他们需要几毫秒才能执行.
 
 Tickers
 
Ticker 类会被挂载在 scheduler 的 scheduleFrameCallback 机制中, 以便每次都能被调用.

每次 tick, Ticker 提供一个回调, 这个回调会有一个自从启动后第一次 tick 的时间间隔.

因为 tickers 总是给出他们启动后相对于第一次 tick 的经过的时间, 所以 tickers 都是同步的.????

Simulator 
抽象类, 将相对时间映射到 double 值.

Animatables

将 double 映射到特定类型的值, 这个animatable 类是无状态的和不可变的.

组合动画

