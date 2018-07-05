
### 前言

性能优化是一个很大的话题，也是一个非常复杂的问题，我们可以通过一些列的工具去查找、分析问题，然后解决。

### UI上可能存在的问题

* 过度绘制
* 丢帧、卡顿
* ANR

### 过度绘制

什么是过度绘制，过度绘制是指在一个像素点上绘制多次。要检测过度绘制也很简单。只需要设置 开发者选项->显示过度绘制区域即可。


![](https://upload-images.jianshu.io/upload_images/2355073-d8db6505d8666f64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/382)


我们优化的目的减少3x、4x区域，尽可能的达到1X区域。要想更好的理解如何去减少过度绘制。我们需要先了解View的绘制步骤是什么样的。

```
1. Draw the background
2. If necessary, save the canvas' layers to prepare for fading
3. Draw view's content
4. Draw children
5. If necessary, draw the fading edges and restore layers
6. Draw decorations (scrollbars for instance)
```

集合源码我们能够知道，减少不必要的的背景色，能有效的减少过度绘制。

默认情况下，每个Window都是有背景色的，可以考虑去掉window的background去减少一定的过度绘制。当然，我们应该严格的检车我们的不居中是否有多余的背景色，如果有，去掉。

### 丢帧、卡顿

```
    private Choreographer.FrameCallback mCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            Choreographer.getInstance().postFrameCallback(mCallback);
            if (time != 0) {
                mTextView.setText(String.valueOf(1000000000 / (frameTimeNanos - time)));
            }
            time = frameTimeNanos;
        }
    };
```

我们能通过上诉方法进行简单的fps计算。不过，更建议大家在开发者模式中设置GPU呈现模式为->在屏幕上显示为条形图。这样更加直观一点。那么，什么样的问题会导致丢帧、卡顿呢。

* 布局复杂，不必要的层级较多
* measure、layout、draw复杂，无法在16ms之内完成
* 主线程阻塞


针对上面的问题，我们有一些比较粗糙的解决办法，如果想要很好的解决。需要借助一些工具并且耐心的去解决，

* 针对布局复杂，我们可以通过merge、viewstub等减少布局层级，不过，这里更推荐使用ConstraintLayout。关于ConstraintLayout的更多知识，可以去逛网学习一哈。
* measure、layout、draw复杂。我们可以使用Hierarchy View去检测，然后根据实际情况去做出优化，比如减少背景，根据层级，选择LinearLayout和RelativeLayout,不过我更建议使用ConstraintLayout。
* 主线程阻塞，大多是因为我们在主线程进行耗时操作，io、频繁创建对象、gc、低效的代码、循环等等。这种情况下，我们可以借助严苛模式、Traceview、Systrace等工具进行分析。


### ANR

当发生ANR的时候，问题就比较严重了，我们需要利用ANR日志去分析问题。这里不细说了。


### Hierarchy View用法

在最新版本的Android Studio中，已经隐藏掉了Android Device Mointor的入口，我们可以通启动sdk/tools/下的monitor去启动Android Device Monitor。Hierarchy View已经集成在ADM里面了，但是我们不一定能使用，有一些设备上是无法获取measure、layout、draw的时间的，这时 我们需要借助 https://github.com/romainguy/ViewServer . 具体的用法大家看文档吧。[官方文档](https://developer.android.com/studio/profile/hierarchy-viewer) 文档上写的很清楚，这里就不多说了。


### TraceView用法

1. 代码调用，我们可以通过Debug类来自己调用，会在Android/data/包名/files下生成.trace文件，
2. 通过ADM工具中集成的TraceView
3. Android Studio中的Android Profiler工具

从这些工具中我们能知道方法的调用次数、执行的耗时时间等，已针对这些进行优化.


### Systrace

关于Systrace的用法建议阅读官方文章。






