# TrainingNote
---
##Getting started
###Supporting Different Screens
>Android automatically scales your layout in order to properly fit the screen. Thus, your layouts for
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

###Managing the Activity Lifecycle

>Activity can exist in one of only three states for an extended period of time:

Activity只能在三种状态下停留较长时间：
- Resumed ————Activity在前台，用户可以交互，又叫做运行状态。
- Paused ————Activity被另一个在前台的，半透明或者没有覆盖整个屏幕的activity遮挡，暂停状态的activity不接受用户的输入，**不执行任何代码**
- Stoped ————Activity完全隐藏，可以认为进入后台了。停止状态下，activity的实例和状态信息比如成员变量还保留，同Paused一样**不执行任何代码**
其他状态都是非常短暂的，比如onCreate调用后就会立刻onStart,然后快速地跟着onResume.

如果MAIN action或者LAUNCHER category没有定义在你的主activity中，你的app图标就不会在抽屉里显示。

>You must implement the onCreate method to perform basic application startup logic that should happen only
once for the entire life of the activity. For example, your implementation of `onCreate should define the user interface
and possibly instantiate some class-scope variables.

你必须实现onCreate方法来处理一些应用基本的初始化逻辑，它只在activity的整个生命周期中执行一次。比如，定义UI，初始化一些类级别的变量。onCreate
执行完成后，onStart和onResume会立刻紧接着被执行，然后activity会停留在onResume，直到发生改变状态的事件，比如接电话，
或者跳到别的activity、屏幕关闭。

相对于生命周期的第一个回调是onCreate，最后一个被调用的是onDestroy。系统调用它作为你activity实例被完全从内存中移除的标志。大部分app
不需要实现这个方法，因为本地引用（local class references）和activity一同被销毁，而你的activity应当在onPause和onStop中做大部分清理工作。
然而，如果你的activity在onCreate中创建了后台线程，或者其他长时间运行的资源，如果没有恰当关闭会造成内存泄露的话，你应该在onDestroy中干掉他们。

系统会在已经调用了onPause和onStop后才调用onDestroy，只有一个种情况例外：在onCreate中调finish，某些情况，你的activity可能会临时决定启动
另一个activity，那么你也许会在onCreate中调finish，这种情况，系统会立刻调用onDestroy而不调用其他生命周期方法。

>As your activity enters the paused state, the system calls the onPause() method on your Activity, which allows you to stop ongoing actions that should not continue while paused (such as a video) or persist any information that should be permanently saved
in case the user continues to leave your app. If the user returns to your activity from the paused state, the system resumes
it and calls the onResume() method.

当你activity进入暂停状态时，系统会调用onPause，在这里你可以停止一些在暂停状态不该继续执行的操作，比如视频播放，或者保存应当永久性保存
的数据，防止用户接下来想离开你的app。如果用户从paused状态回到你的activity，系统会恢复并调用onResume()。
注意，通常情况下，进入onPause都是用户要离开你activity的第一个标志(first indication)。
在onPause()中，你通常需要：
- 停止消耗cpu的操作，比如动画
- 保存没保存的改变（仅当这些改变是用户离开时需要保存的东西时，比如编写email）。
- 释放系统资源，比如broadcast receiver，处理GPS之类的传感器，或者其他任何在activity暂停状态下，你的用户不需要的但是可能会耗电的东西
比如如果你的app使用到Camera，onPause()就是很好的释放它的地方。
```
@Override
public void onPause() {
    super.onPause();  // Always call the superclass method first

    // Release the Camera because we don't need it when paused
    // and other activities might need to use it.
    if (mCamera != null) {
        mCamera.release();
        mCamera = null;
    }
}
```
>Generally, you should not use onPause() to store user changes (such as personal information entered into a form) to permanent storage. The only time you should persist user changes to permanent storage within onPause() is when you're certain users expect the changes to be auto-saved (such as when drafting an email). However, you should avoid performing CPU-intensive work during onPause(), such as writing to a database, because it can slow the visible transition to the next activity (you should instead perform heavy-load shutdown operations during onStop()).
You should keep the amount of operations done in the onPause() method relatively simple in order to allow for a speedy transition to the user's next destination if your activity is actually being stopped.

>Note: When your activity is paused, the Activity instance is kept resident in memory and is recalled when the activity resumes. You don’t need to re-initialize components that were created during any of the callback methods leading up to the Resumed state.

通常情况下，你不需要在onPause中永久性存储用户的改变，比如在表格中输入个人信息。唯一你需要永久性保存的情况就是你确定用户离开的时候想要这些东西自动保存（比如写邮件）。但是，你在onPause中应该避免剧烈消耗cpu的操作，比如写入数据库，因为这会影响到转移到下一个activity的效果（你可以在onStop中执行高负载的操作）。为了能流畅的转移到下个目的地，你应该保持onPause中的操作相对简单。注意：你不需要重新初始化你在任何通往Resumed状态的生命周期中创建的控件。

需要留意的是，activity每次进入前台，系统都会调用onResume()方法，包括第一次创建的时候。因此你可以复写它来初始化你在onPause中释放的组件，执行其他每次activity进入Resumed状态时必须的操作，比如开始动画，或者只有有焦点时才需要初始化的控件。
```
@Override
public void onResume() {
    super.onResume();  // Always call the superclass method first
    //这段对应的是刚刚onPause中的代码
    // Get the Camera instance as the activity achieves full user focus
    if (mCamera == null) {
        initializeCamera(); // Local method to handle camera init
    }
}
```

处理好停止和重新开始你的activity是生命周期中很重要的过程，可以确保用户意识到你的app还活着，并不会丢失进度。有一些你activity会stoped或者restarted的场景：
- 用户打开最近任务，切换到其他app。如果用户通过最近任务或者桌面图标回到app，activity就会restarts。
- 用户在app中打开了新的activity。如果用户按下返回，前一个activity就会被restarted。
- 用户接到电话。

因为在stopped状态，系统内存中还保留了activity实例，所以可能你并不需要实现onStop()、onRestart()甚至是onStart()。对于大部分简单的
activity来说，activity会正常stop和restart，你可能只需要在onPause中暂停下正在进行的操作，释放系统资源罢了。
你不需要重新初始化那些在通往Resumed状态中创建的控件。系统始终追踪了layout中每个view的状态，所以如果用户在edittext中输入了内容，那么
内容会被保留，所以你不需要去保存然后恢复。
极端情况下，当你的activity处于stopped状态时，系统会直接销毁activity而不调用onDestroy。尽管如此，系统还是会保留view对象的状态在Bundle中
并在用户导航到该activity时恢复他们。

app使用onRestart来恢复状态并不常见，所以关于这个方法没有什么建议可说的（那你还搞这个方法干嘛- -）。但是捏，因为你app应当在onStop中清理
所有资源，你应该需要在activity重新启动的时候重新初始化这些资源。此外，你的activity第一次创建也需要初始化他们，因此
onStart()就是个初始化他们的好地方了。比如，用户可能会立刻你app很长时间，那么onStart中就可以验证需要的功能是启用状态：
```
@Override
protected void onStart() {
    super.onStart();  // Always call the superclass method first
    
    // The activity is either being restarted or started for the first time
    // so this is where we should make sure that GPS is enabled
    LocationManager locationManager = 
            (LocationManager) getSystemService(Context.LOCATION_SERVICE);
    boolean gpsEnabled = locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER);
    
    if (!gpsEnabled) {
        // Create a dialog here that requests the user to enable GPS, and use an intent
        // with the android.provider.Settings.ACTION_LOCATION_SOURCE_SETTINGS action
        // to take the user to the Settings screen to enable GPS when they click "OK"
    }
}
```
因为通常你在onStop()中就已经清理了大部分资源，所以onDestory()中通常没太多需要处理的。确保额外的线程在这里被销魂
，其他长时间操作比如method tracing也应该停止。

当你的activity是由于用户按下返回键或者调用finish结束的时候，系统会认为这个activity实例永远消失了，因为这个行为暗示它不在被需要。
然而，如果是系统由于内存紧张而把这个activity回收了，系统会记住这个activity，并且通过一组数据来保存它销毁时的状态，这样用户回到这activity时，系统就会恢复它的状态。这个之前存储的状态就叫"instance state"，由一个键值对的集合组成，存放在Bundle对象中。
需要注意，你的activity每次在屏幕旋转时都会被销毁、重建。系统每次在屏幕配置改变时都会这么做是因为，你的activity可能需要
加载相应的资源。（比如不同的layout）
默认情况，系统使用Bundle来保存你activity布局中的每个view，比如edittext中的字符，listview的position。所以你不需要写代码来保存、恢复它之前的状态。(为了恢复activity中的views的状态，每个view必须要有一个id。)但是你的activity可能有更多的状态信息需要恢复，比如追踪用户进度的一个成员变量。这时候你可以复写`onSaveInstanceState()`。

>The system calls this method when the user is leaving your activity and passes it the Bundle object that will be saved in the event that your activity is destroyed unexpectedly. If the system must recreate the activity instance later, it passes the same Bundle object to both the onRestoreInstanceState() and onCreate() methods.

当用户离开你的activity时，系统调用该方法，并传入Bundle，如果你的activity被意外销毁，bundle就被保存。之后**系统若是重建该activity，就会传入这个bundle对象到`onCreate()`和`onRestoreInstanceState()`。**
为了存储额外的信息，你必须复写onSaveInstanceState：
```
static final String STATE_SCORE = "playerScore";
static final String STATE_LEVEL = "playerLevel";
...

@Override
public void onSaveInstanceState(Bundle savedInstanceState) {
    // Save the user's current game state
    savedInstanceState.putInt(STATE_SCORE, mCurrentScore);
    savedInstanceState.putInt(STATE_LEVEL, mCurrentLevel);
    
    // Always call the superclass so it can save the view hierarchy state
    super.onSaveInstanceState(savedInstanceState);
}
```
如果之前被销毁了，重建之后你可以在onCreate中恢复activity的状态：
```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); // Always call the superclass first
   
    // Check whether we're recreating a previously destroyed instance
    if (savedInstanceState != null) {
        // Restore value of members from saved state
        mCurrentScore = savedInstanceState.getInt(STATE_SCORE);
        mCurrentLevel = savedInstanceState.getInt(STATE_LEVEL);
    } else {
        // Probably initialize members with default values for a new instance
    }
    ...
}
```
相比onCreate，你可能会选`onRestoreInstanceState()`，它在`onStart()`后被调用。仅当有保存的state可以恢复的时候，该方法才被调用。所以不用检查Bundle是否为null。

###Building a Dynamic UI with Fragments
