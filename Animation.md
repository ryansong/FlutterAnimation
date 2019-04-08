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
这部分源码表示的就是在 2000 毫秒内将 logo 图片从 0x0 绘制为 300x300 大小的动画. 类 _LogoAppState 是一个 statusfulWidget，可以调用 setStatus() 方法进行更新 weight. 我们可以看到和普通的 statusfulWidget 相比， 这个例子多了两个成员变量 Animation<double> animation 和 AnimationController controller, 这两个对象是动画能够运行的关键.  
我们看到在 animation 变量的 addListener() 回调方法里面调用了 statusfulWidget 的重绘方法 setState() , 而在 widget 的 build() 方法里使用了 animation 对象的值作为 weight 的宽度和高度使用. 从这里我们能够推测出一个想法：animation 对象通过将监听器注册将 animation 需要更新的变动通知过监听器， 而监听器又调用 setStatus() 方法让 widget 去重新绘制， 而在绘制时，widget 又将 animation 的 value 值当作新绘制图形的参数. 通过这样的机制不断地重绘这个 weight 实现了动画的效果.

## Animation<double>
Animation 是 Flutter 动画库中的核心类，它本身呈现的就是指导动画生成的值(animation.value). Animation 对象是就是会在在一段时间内依次生成一个区间之间值的类, 它的输出可以是线性的、曲线的、一个步进函数或者任何其他可以设计的映射 比如：CurvedAnimation. 在Flutter中，Animation 对象本身和 UI 渲染没有任何关系， UI 会使用 animation.value 的值来确认如何生成对象， 这取决于 UI 自己.  Animation 是一个抽象类，它拥有其当前值和状态（完成或停止）。Animation<double> 是一个比较常用的Animation类, 也可以支持其它的类型，比如： Animation<Color> 或 Animation<Size>. 

## AnimationController
AnimationController 是一个动画控制器




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


#Tween&Animation

Tween 是用来表示变化范围的泛型对象, Animation 则是表示动画所处的状态, Animation 的值一般就在 Tween 给出的范围内. 如果要确定 Animation 的值, 通过 tween 的 evaluate 方法来获得. 根据下面的源码我们可以看到例子里的 Animation 的值是通过 begin + (end - begin) * t 计算的出来的, 这个表达式里面的 t 这是 AnimationController 在 _tick 方法内通过 simulation 计算出来的值. 最终 Animation 的值就是 build 用来更新动画帧的值.

```Dart
  T get value => _evaluatable.evaluate(parent);

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

  _value = _simulation.x(elapsedInSeconds).clamp(lowerBound, upperBound);
```












