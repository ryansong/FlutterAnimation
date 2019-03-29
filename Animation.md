# Flutter 动画原理

最近 Flutter 技术以其流畅的交互和跨平台的特性再次在移动开发领域带来震撼, 就此我们对于 Flutter 的动画架构做一下介绍.


这是一个简单的图标展示代码:
```
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
          // the state that has changed here is the animation object’s value
        });
      });
    controller.forward();
  }
  Widget build(BuildContext context) {
    return new Center(
      child: new Container(
        margin: new EdgeInsets.symmetric(vertical: 10.0),
        height: animation.value,
        width: animation.value,
        child: new FlutterLogo(),
      ),
    );
  }
```
![动画效果](https://github.com/ryansong/FlutterAnimation/raw/master/resource/1553845643685.gif)

AnimationController 设置动画时长, 并控制播放, Animation 添加监听器来调用 setState 方法调用页面更新 build, 使用 Animation.value 的值来绘制帧.

虽然看起来很简单, 但是还是有很多疑惑, 这些类的作用都是什么: SingleTickerProviderStateMixin,Animation,AnimationController,Tween.(排名不分先后顺序)

```
new AnimationController(
        duration: const Duration(milliseconds: 2000), vsync: this)
```
with 是 dart 语言的 mixin 特性, 可以理解成"继承", 所以 widget 相当于是继承了 SingleTickerProviderStateMixin. SingleTickerProviderStateMixin 是 TickerProvider 的子类, 顾名思义它会提供一个 Tiker 对象, 而这个 Tiker 的回调会在每帧绘制的时候调用. 参数名 vsync 的意思就是 垂直帧同步, 和 Ticker 在每帧绘制正好一致. 这样就能理解 Flutter 动画是怎么样不断驱动执行的了.

那这个回调又是怎么传给 AnimationController 的呢?
```
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

 在 AnimationController 的构造方法中, 将 _tick TickerCallback 回调方法绑定到了 Ticker, 这样 AnimationController 也能接收到帧刷新的信号了.


 ```
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
检视 _tick 方法, 发现 AnimationController 会根据 Ticker 的启动时间(elapsed)来更新动画的状态, 并通知给监听器.

看到这里会有一个疑惑, 为什么监听器是注册在 Animation 上的, 监听通知反而由 AnimationController 发送?

还是看源码.
```
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

Animation 是由 Tween 的 animate 方法生成的, 它传入了 AnimationController(也继承 Animation) 对象作为驱动, 而在 AnimationWithParentMixin 的 addListener 函数实现里则把 listener 挂载到了 parent (即AnimationController).
















