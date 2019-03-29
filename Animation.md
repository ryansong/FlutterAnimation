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
在这个例子中 Flutter 动画的关键类都已经出场了:SingleTickerProviderStateMixin,Animation,AnimationController,Tween.(排名不分先后顺序)

首先是 SingleTickerProviderStateMixin, 这是一个 TickerProvider 的子类, 顾名思义它会提供一个 Tiker 对象, 而这个 tiker 的回调会在每帧绘制的时候调用,这样 Flutter 动画就能在每帧绘制的时候更新. 关于 With 关键字 涉及到 Dart 的 MixIn "继承"方式就不展开了.
AnimationController 是一个特殊的 Animation<double> 的子类, 使用动画间隔时间和继承自 SingleTickerProviderStateMixin 的 TickerProvider 来进行初始化. 前者指定了完整动画的播放时间, 后者指定了动画更新的节奏(Ticker滴答).
Animation 对象表示的是动画的一个状态, Tween 对象则是动画从开始到结束的取值范围, AnimationController 依据播放状态来决定当前动画是什么值, 这个在 Tween 取值范围内的值就是 Animation 当前的值.
