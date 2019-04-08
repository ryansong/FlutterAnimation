# Flutter 动画

最近 Flutter 技术以其流畅的交互和跨平台的特性再次在移动开发领域带来震撼, 就此我对于 Flutter 的动画的实现做一下了解.


一个简单的动画效果.
![这是一个简单的 Flutter Logo 动画](https://github.com/ryansong/FlutterAnimation/raw/master/resource/1553845643685.gif)

代码如下
```Dart
class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  Animation<double> animation;
  AnimationController controller;

  initState() {
    super.initState();
    controller = new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this);
    animation = new Tween(begin: 0.0, end: 300.0).animate(controller)
      ..addListener(() {
        setState(() {
          // Animation 对象的值改变了
        });
      });
    controller.forward();
  }
  Widget build(BuildContext context) {
  
        // ....
        height: animation.value,
        width: animation.value,
        // ....
  }
```
这部分源码表示的就是在 2 秒内将 logo 图片从 0x0 绘制为 300x300 大小的动画. 类 _LogoAppState 是一个 statusfulWidget，可以调用 setStatus() 方法进行更新 weight. 我们可以看到和普通的 statusfulWidget 相比， 这个例子多了两个成员变量 Animation<double> animation 和 AnimationController controller, 这两个对象是动画能够运行的关键.  
我们看到在 animation 变量的 addListener() 回调方法里面调用了 statusfulWidget 的重绘方法 setState() , 而在 widget 的 build() 方法里使用了 animation 对象的值作为 weight 的宽度和高度使用. 从这里我们能够推测出：animation 对象通过将监听器注册将 animation 需要更新的变动通知过监听器， 而监听器又调用 setStatus() 方法让 widget 去重新绘制， 而在绘制时，widget 又将 animation 的 value 值当作新绘制图形的参数. 通过这样的机制不断地重绘这个 weight 实现了动画的效果. 

## Animation<double>
Animation 是 Flutter 动画库中的核心类，它会插入指导动画生成的值. Animation 对象知道一个动画当前的状态(例如开始, 停止, 播放, 回放), 但它不知道屏幕上绘制的是什么, 因为 Animation 对象只是提供一个值表示当前需要展示的动画, UI 如何绘制出图形完全取决于 UI 自身如何在渲染和 build() 方法里处理这个值, 当然也可以不做处理. Animation<double> 是一个比较常用的Animation类, 泛型也可以支持其它的类型，比如： Animation<Color> 或 Animation<Size>.  Animation 对象是就是会在在一段时间内依次生成一个区间之间值的类, 它的输出可以是线性的、曲线的、一个步进函数或者任何其他可以设计的映射 比如：CurvedAnimation. 

## AnimationController
AnimationController 是一个动画控制器, 它控制动画的播放状态, 如例子里面的: controller.forward() 就是控制动画"向前"播放. 所以构建 AnimationController 对象之后动画并没有立刻开始执行. 在默认情况下, AnimationController 会在给定的时间内线性地生成从 0.0 到 1.0 之间的数字. AnimationController 是一种特殊的 Animation 对象了, 它父类其实是一个 Animation<double>, 当硬件准备好需要一个新的帧的时候它就会产生一个新的值. 由于 AnimationController 派生自 Animation <double>，因此可以在需要 Animation 对象的任何地方使用它. 但是 AnimationController 还有其他的方法来控制动画的播放, 例如前面提到的 .forward() 方法启动动画. AnimationController 生成的数字(默认是从 0.0 到 1.0) 是和屏幕刷新有关, 前面也提到它会在硬件需要一个新帧的时候产生新值. 因为屏幕一般都是 60 帧/秒, 所以它也通常一秒内生成 60 个数字. 每个数字生成之后, 每个 Animation 对象都会调用绑定的简监听器对象.

#Tween
Tween 本身表示的就是一个 Animation 对象的取值范围, 只需要设置开始和结束的边界值(值也支持泛型). 它唯一的工作就是定义输入范围到输出范围的映射, 输入一般是 AnimationController 给出的值 0.0~1.0.  看下面的例子, 我们就能知道 animation 的 vlaue 是怎么样通过 AnimationController 生成的值映射到 Tween 定义的取值范围里面的.

```Dart
  
   1. Tween.animation 通过传入 aniamtionController 获得一个 _AnimatedEvaluation 类型的 animation 对象(基类为 Animation). 并且将 aniamtionController 和 Tween 对象传入了 _AnimatedEvaluation 对象.
  animation = new Tween(begin: 0.0, end: 300.0).animate(controller)
    ...
  Animation<T> animate(Animation<double> parent) {
    return _AnimatedEvaluation<T>(parent, this);
  }

  2. animation.value 方法即是调用 _evaluatable.evaluate(parent) 方法, 而 _evaluatable 和 parent 分别为 Tween 对象和 AnimationController 对象.
  T get value => _evaluatable.evaluate(parent);
     ....
  class _AnimatedEvaluation<T> extends Animation<T> with AnimationWithParentMixin<double> {
     _AnimatedEvaluation(this.parent, this._evaluatable);
     ....


  3. 这里的 animation 其实就是前面的 AnimationController 对象, transform 方法里面的 animation.value 则就是 AnimationController 线性生成的 0.0~1.0 直接的值. 在 lerp 方法里面我们可以看到这个 0.0~1.0 的值被映射到了 begin 和 end 范围内了.
  T evaluate(Animation<double> animation) => transform(animation.value);

    T transform(double t) {
    if (t == 0.0)
      return begin;
    if (t == 1.0)
      return end;
    return lerp(t);
  }

    T lerp(double t) {
    assert(begin != null);
    assert(end != null);
    return begin + (end - begin) * t;
  }
```


那么 Flutter 是怎么样让这个动画在规定时间不断地绘制的呢?

首先看 Widget 引入的 SingleTickerProviderStateMixin 类. SingleTickerProviderStateMixin 是以 with 关键字引入的, 这是 dart 语言的 mixin 特性, 可以理解成"继承", 所以 widget 相当于是继承了 SingleTickerProviderStateMixin. 所以在 AnimationController 对象的构造方法参数 vsync: this, 我们看到了这个类的使用. 从 "vsync" 参数名意为"垂直帧同步"可以看出, 这个是与绘制动画帧的"节奏器". 

```Dart
  AnimationController({
    double value,
    this.duration,
    this.debugLabel,
    this.lowerBound = 0.0,
    this.upperBound = 1.0,
    this.animationBehavior = AnimationBehavior.normal,
    @required TickerProvider vsync,
  }) : assert(lowerBound != null),
       assert(upperBound != null),
       assert(upperBound >= lowerBound),
       assert(vsync != null),
       _direction = _AnimationDirection.forward {
    _ticker = vsync.createTicker(_tick);
    _internalSetValue(value ?? lowerBound);
  }
 ```
 在 AnimationController 的构造方法中, SingleTickerProviderStateMixin 的父类 TickerProvider 会创建一个 Ticker, 并将 _tick(TickerCallback 类型)回调方法绑定到了 这个 Ticker, 这样 AnimationController 就将回调方法 _tick 和 Ticker 绑定了.

  ```Dart
   @protected
  void scheduleTick({ bool rescheduling = false }) {
    assert(!scheduled);
    assert(shouldScheduleTick);
    _animationId = SchedulerBinding.instance.scheduleFrameCallback(_tick, rescheduling: rescheduling);
  }
 ```
 而 Ticker 会在 start 函数内将 _tick 被绑定到 SchedulerBinding 的帧回调方法内. 返回的 _animationId 是 SchedulerBinding 给定的是下一个动作回调的 ID, 可以根据 _animationId 来取消 SchedulerBinding 上绑定的回调.

 SchedulerBinding 则是在构造方法中将自己的 _handleBeginFrame 函数和 window 的 onBeginFrame 绑定了回调. 这个回调会在屏幕需要准备显示帧之前回调.


再回调 AnimationController 看它是如何控制 Animation 的值的.
```Dart
   void _tick(Duration elapsed) {
    _lastElapsedDuration = elapsed;
    final double elapsedInSeconds = elapsed.inMicroseconds.toDouble() / Duration.microsecondsPerSecond;
    assert(elapsedInSeconds >= 0.0);
    _value = _simulation.x(elapsedInSeconds).clamp(lowerBound, upperBound);
    if (_simulation.isDone(elapsedInSeconds)) {
      _status = (_direction == _AnimationDirection.forward) ?
        AnimationStatus.completed :
        AnimationStatus.dismissed;
      stop(canceled: false);
    }
    notifyListeners();
    _checkStatusChanged();
  }
 ```
在 AnimationController 的回调当中, 会有一个 Simulation 根据动画运行了的时间(elapsed) 来计算当前的的 _value 值, 而且这个值还需要处于 Animation 设置的区间之内. 除了计算 _value 值之外, 该方法还会更新 Animation Status 的状态, 判断是否动画已经结束. 最后通过 notifyListeners 和 _checkStatusChanged 方法通知给监听器 value 和 AnimationStatus 的变化. 监听 AnimationStatus 值的变化有一个专门的注册方法 addStatusListener.

通过监听 AnimationStatus, 在动画开始或者结束的时候反转动画, 就达到了动画循环播放的效果.
```Dart
    // ...
   animation.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        controller.reverse();
      } else if (status == AnimationStatus.dismissed) {
        controller.forward();
      }
    });
    controller.forward();

    // ...
```

 回顾一下这个动画绘制调用的顺序就是, window 调用 SchedulerBinding 的 _handleBeginFrame 方法, SchedulerBinding 调用 Ticker 的 _tick 方法, Ticker 调用 AnimationController 的 _tick 的方法, AnimationContoller 通知监听器, 而监听器调用 widget 的 setStatus 方法来调用 build 更新, 最后 build 使用了 Animation 对象当前的值来绘制动画帧.


看到这里会有一个疑惑, 为什么监听器是注册在 Animation 上的, 监听通知反而由 AnimationController 发送?

还是看源码吧.
```Dart
  Animation<T> animate(Animation<double> parent) {
    return _AnimatedEvaluation<T>(parent, this);
  }

class _AnimatedEvaluation<T> extends Animation<T> with AnimationWithParentMixin<double> {
  _AnimatedEvaluation(this.parent, this._evaluatable);
}

mixin AnimationWithParentMixin<T> {
  Animation<T> get parent;
  /// Listeners can be removed with [removeListener].
  void addListener(VoidCallback listener) => parent.addListener(listener);
}

```
首先 Animation 对象是由 Tween 的 animate 方法生成的, 它传入了 AnimationController(Animation 的子类) 参数 作为 parent 参数. 然后我们发现返回的 _AnimatedEvaluation<T> 泛型对象 使用 mixin "继承" 了 AnimationWithParentMixin<double>, 最后我们看到 Animation 作为 AnimationWithParentMixin 的"子类"实现的 addListener 方法其实是将监听器注册到 parent 对象上了, 也就是 AnimationController.













