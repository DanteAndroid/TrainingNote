# TrainingNote
---
##Getting started
###Supporting Different Screens
Android automatically scales your layout in order to properly fit the screen. Thus, your layouts for
different screen sizes don't need to worry about the absolute size of UI elements but instead focus on the 
layout stucture that affects the user experience (such as the size or position of important views relative
to sibling views).

系统会自动缩放你的布局以适应屏幕。也就是说，为了适用不同的屏幕尺寸，你不需要担心布局中控件的绝对尺寸，而要关注会影响用户体验的布局结构
（比如重要控件相对于sibling views的位置，尺寸）。注：sibling views直译为兄弟view。一个包含默认布局和为大屏幕适配的布局示例：
```
    res/
        layout/
            main.xml
        layout-land/      横屏下的布局
            main.xml      
        layout-large/     大屏幕的布局
            main.xml      
        layout-large-land/      大屏幕的横屏布局
            main.xml      注，文件名必须一致
```

为了给不同屏幕适配不同的图像资源，你应该从原图像的矢量格式开始，使用下面的尺寸范围给每种屏幕生成图像：
- xhdpi: 2.0        200*200
- hdpi: 1.5         150*150
- mdpi: 1.0       eg.100*100

Activity can exist in one of only three states for an extended period of time:
Activity只能在三种状态下停留较长时间：
- Resumed ————Activity在前台，用户可以交互，又叫做运行状态。
- Paused ————Activity被另一个在前台的，半透明或者没有覆盖整个屏幕的activity遮挡，暂停状态的activity不接受用户的输入，**不执行任何代码**
- Stoped ————Activity完全隐藏，可以认为进入后台了。停止状态下，activity的实例和状态信息比如成员变量还保留，同Paused一样**不执行任何代码**
其他状态都是非常短暂的，比如`onCreate()`调用后就会立刻`onStart()`,然后快速地跟着`onResume()`.

如果MAIN action或者LAUNCHER category没有定义在你的主activity中，你的app图标就不会在抽屉里显示。

You must implement the `onCreate()` method to perform basic application startup logic that should happen only
once for the entire life of the activity. For example, your implementation of `onCreate()` should define the user interface
and possibly instantiate some class-scope variables.

你必须实现`onCreate()`方法来处理一些应用基本的初始化逻辑，它只在activity的整个生命周期中执行一次。比如，定义UI，初始化一些类级别的变量。`onCreate()`
执行完成后，`onStart()`和`onResume()`会立刻紧接着被执行，然后activity会停留在`onResume`，直到发生改变状态的事件，比如接电话，
或者跳到别的activity、屏幕关闭。

相对于生命周期的第一个回调是`onCreate()`，最后一个被调用的是`onDestroy()`。系统调用它作为你activity实例被完全从内存中移除的标志。大部分app
不需要实现这个方法，因为本地引用（local class references）和activity一同被销毁，而你的activity应当在`onPause()`和`onStop()`中做大部分清理工作。
然而，如果你的activity在`onCreate()`中创建了后台线程，或者其他长时间运行的资源，如果没有恰当关闭会造成内存泄露的话，你应该在`onDestroy()`中干掉他们。

系统会在已经调用了`onPause()`和`onStop()`后才调用`onDestroy()`，只有一个种情况例外：在`onCreate()`中调`finish()`，某些情况，你的activity可能会临时决定启动
另一个activity，那么你也许会在`onCreate()`中调`finish()`，这种情况，系统会立刻调用`onDestroy()`而不调用其他生命周期方法。
