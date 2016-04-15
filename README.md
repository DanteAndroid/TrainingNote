
```
---Update 4.14---
Building apps部分翻译完成，下一步：润色
```

本文用尽可能通俗且准确的语言翻译了我认为[Training文档](http://developer.android.com/intl/en/training)中较重要的部分。
其他不常用或者依据业务需求而不同的部分（如蓝牙，高级动画）暂不做翻译。
注意，本人无论是英语和技术水平都很有限，如有瑕疵请不要犹豫，告诉我你的想法。

# Menu
1. [开始吧](#getting-started)
    1. [屏幕适配](#supporting-different-screens)
    2. [管理Activity生命周期](#managing-the-activity-lifecycle)
    3. [用Fragments构建动态UI](#building-a-dynamic-ui-with-fragments)
    4. [与其他app交♂互](#interacting-with-other-apps)
    5. [搞好系统权限](#working-with-system-permissions)
2. [分享内容](#content-sharing)
2. [多媒体](#multimedia)
    1. [管理音频播放](#manage-audio-playback)
    2. [拍照](#capturing-photos)
3. [图像 & 动画](#graphics-&-animation)
    1. [高效显示bitmap](#display-bitmaps-efficiently)
    2. [用Scenes和Transitions来使view动起来](#animating-views–using–scenes–and-transitions)
    3. [添加动画](#adding-animations)
4. [网络连接](#connectivity-&-the-cloud)

<!--1. [Transferring Data Without Draining the Battery](transferring–data–without–draining-the-battery)-->
    
---
##Getting started

###Supporting Different Screens

系统会自动缩放你的布局以适应屏幕。也就是说，为了支持不同的屏幕(以下都称为适配)的时候，你不需要担心布局中控件的绝对尺寸，而要关注会影响用户体验的布局结构
，比如重要控件相对于sibling views(兄弟view)的位置，尺寸。一个包含默认布局和为大屏幕适配的布局示例：
```
    res/
        layout/
            main.xml
        layout-land/      横屏下的布局
            main.xml      
        layout-large/     大屏幕的布局
            main.xml      
        layout-large-land/      大屏幕的横屏布局
            main.xml      注，**文件名必须一致**
```

为了适配不同的图像资源，你应该从原图像的矢量格式开始，用下面的尺寸为每种屏幕生成图像：
- xhdpi: 2.0        eg.200*200
- hdpi: 1.5         150*150
- mdpi: 1.0         100*100

***

###Managing the Activity Lifecycle

Activity只能在三种状态下停留较长(extended period)时间：
- Resumed ————Activity在前台，用户可以交互，又叫做运行状态。
- Paused ————Activity被另一个在前台的，半透明或者没有覆盖整个屏幕的activity遮挡，暂停状态的activity不接受用户的输入，**不执行任何代码**
- Stoped ————Activity完全隐藏，可以认为进入后台了。停止状态下，activity的实例和状态信息比如成员变量还保留，同Paused一样**不执行任何代码**
其他状态都是非常短暂的，比如onCreate调用后就会立刻onStart,然后快速跟来onResume.

如果MAIN action或者LAUNCHER category没有定义在你的主activity中，你的app图标就不会在抽屉里显示。

你必须实现onCreate方法来处理应用的一些基本启动逻辑，它只在activity的整个生命周期中执行一次。比如，定义UI，初始化一些类级别的变量。onCreate执行后，onStart和onResume会立刻紧接着被执行，然后activity会停留在onResume，直到发生改变状态的事件，比如接电话，或者跳到别的activity、屏幕关闭。

相比于生命周期的第一个回调onCreate，最后一个被调用的是onDestroy。系统调用它作为你activity实例被完全从内存中移除的标志。大部分app
不需要实现这个方法，因为本地类引用(local class references)和activity一同被销毁，而你的activity应当在onPause和onStop中做大部分清理工作。
然而，如果你的activity在onCreate中创建了后台线程，或者其他长时间运行的资源，如果没有恰当关闭会造成内存泄露的话，你应该在onDestroy中干掉他们。

系统会在已经调用了onPause和onStop后才调用onDestroy，只有一个种情况例外：在onCreate中调finish，某些情况，你的activity可能会临时决定启动
另一个activity，那么你也许会在onCreate中调finish，这种情况，系统会立刻调用onDestroy而不调用其他生命周期方法。

当你activity进入暂停状态时，系统会调用onPause，在这里你可以停止一些在暂停状态不该继续执行的操作，比如视频播放，或者保存应当永久性保存
的数据，防止用户接下来想离开你的app。如果用户从暂停状态回到你的activity，系统会恢复并调用onResume()。
注意，通常情况下，进入onPause都是用户要离开你activity的第一个标志(first indication)。

在onPause()中，你通常需要：
- 停止消耗cpu的操作，如动画
- 保存没保存的改变（仅当**这些改变是用户离开时需要保存的东西**时，比如编写email）。
- 释放系统资源，比如broadcast receiver，处理GPS之类的传感器，或者其他任何在activity暂停状态下，你的用户不需要的但可能耗电的东西
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

通常情况下，你不需要在onPause中永久性存储用户的改变，比如表格中输入的个人信息。唯一你需要永久性保存的情况就是你确定用户离开的时候想要这些东西自动保存（如写邮件）。但是，你在onPause中应该避免剧特别消耗cpu的操作，如写入数据库，因为这会影响到转移到下一个activity的效果（你可以在onStop中执行高负载的操作）。为了能流畅的转移到下个目的地，你应该保持onPause中的操作相对简单。注意：你不需要重新初始化你在任何通往Resumed状态的生命周期中(onCreate, onStart)创建的控件。

需要留意的是，activity每次进入前台系统都会调用onResume()方法，包括第一次创建的时候。因此你可以复写它来初始化你在onPause中释放的组件，执行其他每次activity进入Resumed状态时必须的操作，比如开始动画，或者只有获得焦点时才需要初始化的控件。

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

处理好停止和重新开始你的activity是生命周期中很重要的过程，可以确保用户意识到你的app还活着，且不会丢失进度。下面是一些你activity会stoped或者restarted的场景：
- 用户打开最近任务，切换到其他app。如果用户通过最近任务或者桌面图标回到app，activity就会restarts。
- 用户在app中打开了新的activity。如果用户按下返回，前一个activity就会被restarted。
- 用户接到电话。

因为在stopped状态，系统内存中还保留了activity实例，所以可能你并不需要实现onStop()、onRestart()甚至是onStart()。对于大部分简单的
activity来说，activity会正常stop和restart，你可能只需要在onPause中暂停下正在进行的操作，释放系统资源罢了。
你不需要重新初始化那些在通往Resumed状态中创建的控件。系统始终追踪了layout中每个view的状态，所以如果用户在edittext中输入了内容，那么
内容会被保留，所以你不需要去保存然后恢复。极端情况下，当你的activity处于stopped状态时，系统会直接销毁activity而不调用onDestroy。尽管如此，系统还是会保留view对象的状态在Bundle中并在用户导航到该activity时恢复他们。

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
因为通常你在onStop()中就已经清理了大部分资源，所以onDestory()中通常没太多需要处理的。确保额外的线程在这里被销毁了，其他长时间操作比如method tracing也应该停止。

当你的activity是由于用户按下返回键或者调用finish结束的时候，系统会认为这个activity实例永远消失了，因为这个行为暗示它不再被需♂要。
然而，如果是系统**由于内存紧张而把这个activity回收**了，系统会记住这个activity，并且通过一组数据来保存它销毁时的状态，这样用户回到这activity时，系统就会恢复它的状态。这个之前存储的状态就叫"instance state"，由一个键值对的集合组成，存放在Bundle对象中。
需要注意，你的activity每次在屏幕旋转时都会被销毁、重建。系统每次在屏幕配置改变时都会这么做是因为，你的activity可能需要
加载相应的资源。（比如不同的layout）
默认情况，系统使用Bundle来保存你activity布局中的每个view，比如edittext中的字符，listview的position。所以你不需要写代码来保存、恢复它之前的状态。(为了恢复activity中的views的状态，每个view必须要有一个id。)但是你的activity可能有更多的状态信息需要恢复，比如追踪用户进度的一个成员变量。这时候你可以复写`onSaveInstanceState()`。
 
当用户离开你的activity时，系统调用该方法，并传入Bundle，如果你的**activity被意外销毁，Bundle就被保存。之后系统若是重建该activity，就会传入这个bundle对象到`onCreate()`和`onRestoreInstanceState()`。**
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

***

###Building a Dynamic UI with Fragments

为了创建动态的、多面板的UI，你需要封装UI控件和activity行为成模块，这样可以包进你的activity(swap into and out of your activities)。
你可以用Fragment类来创建这些模块，它就像一个嵌套的activity一样。你可以把Fragment看做模块化的activity。有自己的布局、生命周期、输入事件，还能在activity运行时动态添加、移除。（有点像你可以在不同activity中复用的子activity）

由于fragment是可复用的、模块化的UI组件，每个实例都必须与父Activity相关联。你可以通过在布局文件中定义fragment来实现这种关联。示例：
```
res/layout-large/news_articles.xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent">

    <fragment android:name="com.example.android.fragments.HeadlinesFragment"
              android:id="@+id/headlines_fragment"
              android:layout_weight="1"
              android:layout_width="0dp"
              android:layout_height="match_parent" />

    <fragment android:name="com.example.android.fragments.ArticleFragment"
              android:id="@+id/article_fragment"
              android:layout_weight="2"
              android:layout_width="0dp"
              android:layout_height="match_parent" />

</LinearLayout>
```
注意，当你在activity的布局文件中定义fragment时，是无法运行时移除的。为了能动态添加或移除Fragment，你得用FragmentManager来创建FragmentTransaction。动态处理Fragment时有一点很重要————你的activity布局必须包含一个Container View作为你插入fragment的容器。
```
res/layout/news_articles.xml:

<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

注意相比上节中的布局，这个layout没有-large的后缀，也就是用在默认的小尺寸的屏幕上。下面是activity中动态添加fragment的代码(※重要※)：
```
@Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.news_articles);

        // Check that the activity is using the layout version with
        // the fragment_container FrameLayout
        //检查activity是否是含有container（如果是，说明需要动态添加）
        if (findViewById(R.id.fragment_container) != null) {

            // However, if we're being restored from a previous state,
            // then we don't need to do anything and should return or else
            // we could end up with overlapping fragments.
            //**如果是从之前状态恢复的，并不需要做什么，否则可能会出现fragment重叠**
            if (savedInstanceState != null) {
                return;
            }

            // Create a new Fragment to be placed in the activity layout
            HeadlinesFragment firstFragment = new HeadlinesFragment();
            
            // In case this activity was started with special instructions from an
            // Intent, pass the Intent's extras to the fragment as arguments
            //**以防activity被含有特别指示的Intent启动的，把activity的Intent中的extras设为Fragment的arguments**
            firstFragment.setArguments(getIntent().getExtras());
            
            // Add the fragment to the 'fragment_container' FrameLayout
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.fragment_container, firstFragment).commit();
        }
    }
```

请记住，当你执行fragment事务比如replace或者remove时，允许用户能回到之前的状态并"撤销"该改变通常是合适的。当你执行这些操作，并`addToBackStack()`时，被移除的fragment实际上是处于stopped状态而非destroyed。如果用户返回并恢复之前的fragment，它会重新启动(restarts)。**如果你不加到back stack，那么被remove或replace的fragment就会被销毁**。addToBackStack()有一个可选的String参数，给这个transaction指定了独一无二的名字。这个名字没什么卵用，除非你打算用fragment操作中的FragmentManager.BackStackEntry里的API。

所有的fragment之间的通讯都应该通过相连接的activity来完成，俩fragment绝不应该直接交流。Fragment和activity通讯示例：
```
public class HeadlinesFragment extends ListFragment {
    OnHeadlineSelectedListener mCallback;

    // Container Activity must implement this interface
    public interface OnHeadlineSelectedListener {
        public void onArticleSelected(int position);
    }

    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        
        // This makes sure that the container activity has implemented
        // the callback interface. If not, it throws an exception
        try {
            mCallback = (OnHeadlineSelectedListener) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString()
                    + " must implement OnHeadlineSelectedListener");
        }
    }
    
    ...
}
//Use the callback interface to deliver the event to the parent activity.
    @Override
    public void onListItemClick(ListView l, View v, int position, long id) {
        // Send the event to the host activity
        mCallback.onArticleSelected(position);
    }

```
```
public static class MainActivity extends Activity
        implements HeadlinesFragment.OnHeadlineSelectedListener{
    ...
    
    public void onArticleSelected(int position) {
        // The user selected the headline of an article from the HeadlinesFragment
        // Do something here to display that article
    }
}
```

宿主activity(host activity)可以通过`findFragmentById()`来找到fragment实例，然后直接调用其public方法，以发消息给fragment。
例如，上面例子中activity可能包含另外一个fragment，通过回调中返回的数据来让该fragment显示相应的item：
```
public static class MainActivity extends Activity
        implements HeadlinesFragment.OnHeadlineSelectedListener{
    ...

    public void onArticleSelected(int position) {
        // The user selected the headline of an article from the HeadlinesFragment
        // Do something here to display that article

        ArticleFragment articleFrag = (ArticleFragment)
                getSupportFragmentManager().findFragmentById(R.id.article_fragment);

        if (articleFrag != null) {
            // If article frag is available, we're in two-pane layout...
            // Call a method in the ArticleFragment to update its content
            //如果能找到详情fragment，说明在双面板的布局中，直接调用其更新方法
            articleFrag.updateArticleView(position);
        } else {
            // Otherwise, we're in the one-pane layout and must swap frags...
            // Create fragment and give it an argument for the selected article
            //否则，就是在单面板布局中，需要replace相应的fragment。
            ArticleFragment newFragment = new ArticleFragment();
            Bundle args = new Bundle();
            args.putInt(ArticleFragment.ARG_POSITION, position);
            newFragment.setArguments(args);
        
            FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();

            // Replace whatever is in the fragment_container view with this fragment,
            // and add the transaction to the back stack so the user can navigate back
            //无论container中是啥，都应该replace掉，并且加到back stack这样用户可以返回。
            transaction.replace(R.id.fragment_container, newFragment);
            transaction.addToBackStack(null);

            // Commit the transaction
            transaction.commit();
        }
    }
}
```S

你可以用下面的方法，创建一个SharedPreference文件或者获得已经存在的sp：
- `getSharedPreferences()` 通过指定文件名，来获得相应的sp。你可以在app中的任何Context下使用该方法
- `getPreferences()` 在Activity中调用该方法，可以获得此activity专属的sp。不需要指定文件名。
- `PreferenceManager.getDefaultSharedPreferences()` 包级别的sp(注：这个实际上返回的就是以`context.getPackageName() + "_preferences"`作为文件名的sp)

目前，所有app不需要指定特殊权限就可以读取外部存储。但是这一点将来会改变。所以为了确保你的app能像预期一样工作，你需要在manifest中添加
` <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />`但是如果你的app使用了WRITE_EXTERNAL_STORAGE权限，那么就默认也具有读取权限了。

内部存储写入文件示例：
```
String filename = "myfile";
String string = "Hello world!";
FileOutputStream outputStream;

try {
  outputStream = openFileOutput(filename, Context.MODE_PRIVATE);
  outputStream.write(string.getBytes());
  outputStream.close();
} catch (Exception e) {
  e.printStackTrace();
}
```
或者你只需要缓存一些文件，可以用`createTempFile()`，下面的示例演示了**从url中提取文件名**，并用它在app的内部cache文件夹中创建文件：
```
public File getTempFile(Context context, String url) {
    File file;
    try {
        String fileName = Uri.parse(url).getLastPathSegment();
        file = File.createTempFile(fileName, null, context.getCacheDir());
    catch (IOException e) {
        // Error while creating file
    }
    return file;
}
```
你app的内部存储文件夹放在在android文件系统中的一个特殊位置，它是由你的包名决定的。严格来说，只要你把文件模式设为可读，其他app就可以读取你app的内部文件。但是，他还得知道你app的包名、文件名。只要你把内部存储文件设为`MODE_PRIVATE`，其他app就绝对没法访问。

在访问外部存储之前，你总是应该先判断是否可用（因为可能用户连接存储到PC，或者移除SD卡）：
```
/* Checks if external storage is available for read and write */
public boolean isExternalStorageWritable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state)) {
        return true;
    }
    return false;
}

/* Checks if external storage is available to at least read */
public boolean isExternalStorageReadable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state) ||
        Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
        return true;
    }
    return false;
}
```
尽管外部存储对于用户和其他app都是可修改的，你还是可以有俩类别的文件可以保存：
- 公共文件：
应该让用户和其他app自由访问的文件。应用被卸载时，这些文件应该保留。例如由你app拍摄或者下载的文件。
- 私有文件：
你app正当拥有的，卸载时应当被删除的文件。尽管技术上来说，用户和其他app可以访问这些文件，但是他们在脱离你app之后就没什么意义（价值）。当用户卸载app时，系统会删掉你外部私人目录中的文件。例如，你app下载的扩充资源或者缓存的媒体文件。

如果要保存到外部存储的公共文件，用`getExternalStoragePublicDirectory()`（记得先判断外部存储可用性）：
```
    // Get the directory for the user's public pictures directory. 
    File file = new File(Environment.getExternalStoragePublicDirectory(
            Environment.DIRECTORY_PICTURES), albumName);
    if (!file.mkdirs()) {
        Log.e(LOG_TAG, "Directory not created");
    }
    return file;
}
```
如果要保存私有文件，用`getExternalFilesDir()`（这个方法创建的每个目录都会被加到一个父目录，它包含了了你所有外部存储文件的）：
```
public File getAlbumStorageDir(Context context, String albumName) {
    // Get the directory for the app's private pictures directory. 
    File file = new File(context.getExternalFilesDir(
            Environment.DIRECTORY_PICTURES), albumName);
    if (!file.mkdirs()) {
        Log.e(LOG_TAG, "Directory not created");
    }
    return file;
}
```
如果预定义的子文件夹名称不能满♂足你，你可以调用`getExternalFilesDir()`并传入null。这样会返回你app在外部存储的私有文件夹的根目录（以下简称为根目录）。记住，`getExternalFilesDir()`会在根目录下创建一个目录，而这个根目录会在app卸载的时候也被删除（因为它是私有文件夹呀）。

无论你用getExternalStoragePublicDirectory()还是getExternalFilesDir()保存文件，有一件事很重要：使用系统提供的文件夹名的API比如DIRECTORY_PICUTRES。这些文件夹名可以确保系统能正确处理里面的文件。比如保存在`DIRECTORY_RINGTONES`里的文件会被系统媒体扫描器分类到铃声而不是音乐。

如果你知道你要保存的数据有多大，你可以直接使用`getFreeSpace()`或`getTotalSpace()`查询可用空间而不需要抛出`IOException`。这俩方法分别fa返回现在可用的空间，存储器的总空间。但系统并不保证你获得的可用空间就是你能写的空间（字节），如果返回的空间比你想写入的多几mb，或者文件系统占用了90%以下是没问题的。否则你可能不该写入。不一定非得在保存文件之前查询可用空间，你可以直接写入，然后catch到`IOException`。比如你想把PNG图片转成JPEG，就不知道文件会有多大。

你可以直接调用文件的delete方法来删除。如果文件保存在内部存储，可以用`mContext.deleteFile(fileName);`来定位并删除文件。当app被卸载时，系统会删掉内部存储的所有文件，外部存储中，使用getExternalFilesDir()创建的文件。但是呢，你应该定期清理所有用`getCacheDir()`创建的缓存文件，定期删除其他你不需要的文件。

---

###Interacting with Other Apps

通常你会用显示intent来在activity间切换，它定义了你想启动的组件的类名。但是，如果你
需要进行独立于app之外的操作，比如查看地图，你得用隐式intent。它不指定类名而是声明了一个action。
Intent通常会与data联系在一起，比如你要查看的地址，或者你想发送的邮件信息，取决于你要创建的意图。data数据可能是Uri，
可能是其他数据类型，也可能不需要data。

如果你的data是Uri，Intent有个构造方法可以让你方便地指定action和data：
```
//initiate a call
Uri number = Uri.parse("tel:5551234");
Intent callIntent = new Intent(Intent.ACTION_DIAL, number);
//view a web page
Uri webpage = Uri.parse("http://www.android.com");
Intent webIntent = new Intent(Intent.ACTION_VIEW, webpage);
```

其他类型的隐式intent需要extra data来提供不同的数据类型，比如String。你可以使用putExtra()添加一或多个extra数据。默认情况下系统会基于
Uri数据来判断合适的MIME类型。如果你intent里没有包含Uri，你得用setType()来指定与intent关联的数据类型：
```
//Create a calendar event
Intent calendarIntent = new Intent(Intent.ACTION_INSERT, Events.CONTENT_URI);
Calendar beginTime = Calendar.getInstance().set(2012, 0, 19, 7, 30);
Calendar endTime = Calendar.getInstance().set(2012, 0, 19, 10, 30);
calendarIntent.putExtra(CalendarContract.EXTRA_EVENT_BEGIN_TIME, beginTime.getTimeInMillis());
calendarIntent.putExtra(CalendarContract.EXTRA_EVENT_END_TIME, endTime.getTimeInMillis());
calendarIntent.putExtra(Events.TITLE, "Ninja class");
calendarIntent.putExtra(Events.EVENT_LOCATION, "Secret dojo");
```

注意，尽可能详细地定义你的Intent很重要。比如你想用ACTION_VIEW来显示图片，你应该指定MIME type为`image/*`
这样可以防止触发可以查看其它类型数据（比如地图）的app。

尽管android平台保证每个特定的intent都可以被内置的app来解析，在调用intent之前，你还是应该验证一下。因为如果你调用intent没有app能打开的话，你的app就会直接崩掉。你应该在activity刚启动的时候就检查intent，这样你可以在用户使用它之前就禁用掉这个功能。如果你知道有特定app可以处理这个intent，你可以给用户提供一个下载链接：
```
PackageManager packageManager = getPackageManager();
List activities = packageManager.queryIntentActivities(intent,
        PackageManager.MATCH_DEFAULT_ONLY);
boolean isIntentSafe = activities.size() > 0;
```

创建好intent，设置好extra后，调用startActivity()给系统。系统如果识别到有多个activity可以处理，就会显示一个对话框让用户选择，如果只有一个，系统就会立刻打开。默认情况对话框底部都会有一个checkbox可以让用户选择默认打开的应用。这很方便，比如用户倾向于使用同一个浏览器或是相机app。但用户可能每次都想选不同的app，比如分享操作，这种情况你得明确地创建一个选择框：
```
Intent intent = new Intent(Intent.ACTION_SEND);
...

// Always use string resources for UI text.
// This says something like "Share this photo with"
String title = getResources().getString(R.string.chooser_title);
// Create intent to show chooser
Intent chooser = Intent.createChooser(intent, title);

// Verify the intent will resolve to at least one activity
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```

启动Activity不必是单向的，你还可以启动其他activity并接受结果。比如你可以打开联系人让用户选择一个联系方式然后你接受这个联系人的详情作为一个result。当然，前提是该activity必须设计为可以返回一个result。你可以在`onActivityResult()`来接受回调。用`startActivityForResult()`既可以用显式也可用隐式intent。如果启动的是你自己的activity，你应该用显示intent来确保接受的result是正确的。
用来startActivityForResult的和普通的intent没什么大区别，唯一不同就是你得传一个额外的int值作为请求码(request code)。当你收到result Intent时，回调会提供同一个request code这样你可以验证结果并决定如何处理。例如选择一个联系人：
```
static final int PICK_CONTACT_REQUEST = 1;  // The request code
...
private void pickContact() {
    Intent pickContactIntent = new Intent(Intent.ACTION_PICK, Uri.parse("content://contacts"));
    pickContactIntent.setType(Phone.CONTENT_TYPE); // Show user only contacts w/ phone numbers
    startActivityForResult(pickContactIntent, PICK_CONTACT_REQUEST);
}
```
当用户从接下来的activity返回时，系统会调用你activity的onActivityResult()。它包括3个参数：
- 你传入startActivityForResult的request code
- 由第二个activity决定的result code。要么是RESULT_OK（表示操作完成）要么是RESULT_CANCEL(用户退出或者因为某些原因操作失败)
- 带着返回数据的Intent。
```
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    // Check which request we're responding to
    if (requestCode == PICK_CONTACT_REQUEST) {
        // Make sure the request was successful
        if (resultCode == RESULT_OK) {
            // The user picked a contact.
            // The Intent's data Uri identifies which contact was selected.
            // Get the URI that points to the selected contact
            Uri contactUri = data.getData();
            // We only need the NUMBER column, because there will be only one row in the result
            String[] projection = {Phone.NUMBER};

            // Perform the query on the contact to get the NUMBER column
            // We don't need a selection or sort order (there's only one result for the given URI)
            // CAUTION: The query() method should be called from a separate thread to avoid blocking
            // your app's UI thread. (For simplicity of the sample, this code doesn't do that.)
            // Consider using CursorLoader to perform the query.
            Cursor cursor = getContentResolver()
                    .query(contactUri, projection, null, null, null);
            cursor.moveToFirst();

            // Retrieve the phone number from the NUMBER column
            int column = cursor.getColumnIndex(Phone.NUMBER);
            String number = cursor.getString(column);

            // Do something with the phone number...
        }
    }
}
```

为了正确处理结果，你必须明白结果Intent是什么格式的。如果是你自己的activity返回的结果当然很简单。android平台提供了自己的API，你可以通过他们获得明确的result data。比如联系人总是返回一个带有被选择的联系人的Uri，相机总是返回一个在"data"的extra里面包含Bitmap的数据。
另外，联系人app可以授予你的app一个临时权限来从ContactsProvider读取结果。但是只允许你读取特定的需求的联系人，所以你没法去查询一个联系人，除非你声明READ_CONTACTS权限。

如果你的app能处理可能对其他app有用的操作，你应该准备好从其他app接受action请求。比如你做了一个可以分享信息给朋友的app，你最好能支持ACTION_SEND的intent。为了让其他app能启动你的activity，你需要在manifest里面添加相应activity的<intent-filter>元素。
当你的app被安装后，系统会识别你的intent filter并把信息添加到一个被所有app支持的intents的内部目录。当app用隐式intent调用startActivity或者startActivityForResult，系统会找到能响应它的activity。仅当activity的intent filter满足下面的标准，系统才可能会把intent发给它：
- <action> 描述需要执行的动作的字符串。通常是平台定义好的比如ACTION_SEND, ACTION_VIEW。如果不是平台的常量是你自己定义的，必须使用全名。
- <data> 使用一个或者多个属性来指定与intent相关联的数据类型。你可以仅指定MIME type，URI前缀，URI scheme，或者它们的组合。如果你不需要指定data的Uri，比如你的activity使用了extra data而不是Uri，你应该指定`android:mimeType`属性，如`text/plain`, `image/jpeg`
- <category> 提供额外的方式来个性化处理intent的activity。通常使用默认的CATEGORY_DEFAULT。如果没添加这个category，那你的activity就不能处理隐式的intent。

你可以在activity任何生命周期调用`getIntent()`来解析启动这个activity的intent，但通常会在onCreate或者onStart里面：
```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);

    // Get the intent that started this activity
    Intent intent = getIntent();
    Uri data = intent.getData();

    // Figure out what to do based on the intent type
    if (intent.getType().indexOf("image/") != -1) {
        // Handle intents with image data ...
    } else if (intent.getType().equals("text/plain")) {
        // Handle intents with text ...
    }
}
```

如果需要给调用它的activity返回数据，只要`setResult()`指定结果码和结果intent就行。当你操作完成而用户应该返回到原activity时，调用finish来关闭你的activity：
```
// Create intent to deliver some kind of result data
Intent result = new Intent("com.example.RESULT_ACTION", Uri.parse("content://result_uri");
setResult(Activity.RESULT_OK, result);
finish();
```

你必须指定一个result code给这个result。然后你可以用Intent提供额外的数据。这个结果默认是RESULT_CANCELED，所以如果用户在操作完成之前点了返回，原activity就能收到一个取消的result。如果你只是需要返回一个代表几种可能的结果之一的数值，你可以直接把result code设为大于0的任意值。如果你不需要包含intent，你可以只传一个result code: `setResult(RESULT_COLOR_RED);     finish();` 这对于返回结果到你自己的app时很好用。因为这个result可以指向决定返回值的常量。不需要检查你的activity是startActivity还是startActivityForResult调用的。只要写上setResult以防启动你的activity需要一个结果。如果不需要，结果会被忽略。

***

### Working with System Permissions

为了保护系统完整性和用户隐私，android在一个有限权限的沙箱里运行每个app。如果这个app像用沙箱之外的资源、信息，就需要明确地申请权限。

取决于权限有多敏感，系统可能会自动授予权限或让用户来选择授权与否。比如如果你的app要求打开设备的闪光灯，系统就自动授权；但是如果你需要读取联系人，系统就会询问用户。这个行为取决于平台的版本，如果是Android5.1和之前的版本，会在安装的时候让用户授权；如果是6.0或者更高版本，就在运行时询问用户。通常来说，只要app**想用到这个app本身不创造的资源、信息，或者进行影响到设备或者其他app的操作，都需要权限**。例如访问网络、使用相机、开关wifi，都需要相应权限。你的app只需要它亲自操作的权限，并不需要其他app来执行或者提供信息的权限。比如如果你的app需要从联系人app中读取信息，就不需要READ_CONTACTS，而那个联系人app才需要。（但是如果你app直接读取用户的联系人时就需要）
申请权限时，需要在manifest的顶级（top-level）元素下加入例如`<uses-permission android:name="android.permission.SEND_SMS"/>`，这样就可以有发信息的权限。

从android6.0开始，用户可以随时撤回app的权限，**哪怕app适配的是低版本的API**。因此你应该测试你的app在缺失需要的权限时，能否正常运行。
如果你的app需要一个风险性权限，你必须在执行相关操作时每次都检查有没有这个权限。用户可能昨天用了相机，但并不能假设今天还有那个权限。
例如检查写入日历的权限：`int permissionCheck = ContextCompat.checkSelfPermission(thisActivity,Manifest.permission.WRITE_CALENDAR);`如果app拥有这权限，就会返回`PackageManager.PERMISSION_GRANTED`；如果返回PERMISSION_DENIED，app就得向用户申请权限。如果你的app在manifest列出了风险性权限，就必须询问用户来授权。android提供了一些方法来申请权限，这些方法会显示标准的dialog，而且你没法去自定义。
有时候可能想帮助用户理解你为啥需要某权限。比如用户打开一个相册软件，就不会对它要用相机感到奇怪。但用户可能想知道为什么它要获得用户位置或联系人。在你申请权限之前，你应该考虑需不需要给用户一个解释。请记住，你并不想用铺天盖地的解释来压垮用户；如果给太多解释，用户可能很烦，然后直接卸载掉了。
一种可行的方法是：**仅当用户关掉那个需要的权限时才提供解释**。如果用户一直关掉权限，但还想用功能说明他可能不了解为啥需要这个权限。此时给他一个解释就很合适。为了查询啥时候需要显示解释，android提供一个方法`shouldShowRequestPermissionRationale()`，如果请求过该权限而用户拒绝了，它就返回true。如果用户关掉权限而且选了不要再询问，这方法就永远返回false。如果设备禁止app访问该权限它也返回false。
用 `requstPermissions()`申请权限，需要传入权限，和一个int请求码来识别你需要的权限。这方法是异步的，他会迅速执行完，然后在用户响应请求对话框之后，系统会根据结果来调用回调，传入同一个请求码：
```
if (ContextCompat.checkSelfPermission(thisActivity,
                Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {
    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
            Manifest.permission.READ_CONTACTS)) {

        // Show an expanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.

    } else {
        // No explanation needed, we can request the permission.

        ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
}


//override this method to find out whether the permission was granted
@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
            // If request is cancelled, the result arrays are empty.
            if (grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                // permission was granted, yay! Do the
                // contacts-related task you need to do.

            } else {

                // permission denied, boo! Disable the
                // functionality that depends on this permission.
            }
            return;
        }
        // other 'case' lines to check for other
        // permissions this app might request
    }
}
```
这个对话框描述了你需要访问的权限组而非具体权限，对于每个permission group，用户每次只需要授权一次。如果你app请求同一个group的其他权限，系统会自动授权（调用你的onRequestPermissionsResult回调，并传入
PERMISSION_GRANTED）。即使用户已经授予同一个group的其他权限，你的app还是得每次需要的时候请求权限。此外，将来group可能会发生变化，你并不能依赖于同组的其他权限。当你在onRequestPermissionsResult中收到PERMISSION_DENIED时，并不能假设用户采取了操作（可能勾选了不在询问）。

使用权限和Intent的区别：
- 用权限：对于操作时用户的体验有完全的控制权，但是可能增加你任务的复杂性（要设计合适的UI）；如果不授权，就完全不可用。
- Intent：不用设计UI，这意味着用户可能在一个你从来没见过的app上交互；如果用户没指定默认app，用户每次执行操作时都要经历一个额外的选择dialog。（看到这里不得不说Google实在太特么细心了）

---

## Content Sharing

如果把Intent传入`Intent.createChooser()`中，就可以每次都显示选择对话框。这有一些额外的好处：
- 如果没有匹配的app，android会显示提示信息
- 你可以为选择框指定一个标题

分享文本的示例：
```
Intent sendIntent = new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT, "This is my text to send.");
sendIntent.setType("text/plain");
startActivity(Intent.createChooser(sendIntent, getResources().getText(R.string.send_to)));
```

分享二进制数据通常使用ACTION_SEND加上恰当的MIME type的设置，然后把URI数据放在一个叫`EXTRA_STREAM`附加值里面。通常用来分享图片，但是也可以用来分享任意类型的二进制内容：
```
Intent shareIntent = new Intent();
shareIntent.setAction(Intent.ACTION_SEND);
shareIntent.putExtra(Intent.EXTRA_STREAM, uriToImage);
shareIntent.setType("image/jpeg");
startActivity(Intent.createChooser(shareIntent, getResources().getText(R.string.send_to)));
```
你可以用`*/*`的MIME type，但是这样的话只有能处理泛型数据的activity才会被匹配。

接受分享的app需要对Uri指向的数据的访问权限。推荐做法：
> Store the data in your own ContentProvider, making sure that other apps have the correct permission to access your provider. The preferred mechanism for providing access is to use per-URI permissions which are temporary and only grant access to the receiving application. An easy way to create a ContentProvider like this is to use the FileProvider helper class.

- 把数据存储到你自己的ContentProvider，确保其他app有正确权限能访问你的Provider。提供权限的一个极好的技巧是用per-URI permissions，它是临时性的权限，而且只授予给接受的app。一个简单的方法是创造一个像这样的ContentProvider来用FileProvider帮助类（这里翻译不太清楚，因为没用过）。

>Use the system MediaStore. The MediaStore is primarily aimed at video, audio and image MIME types, however beginning with Android 3.0 (API level 11) it can also store non-media types (see MediaStore.Files for more info). Files can be inserted into the MediaStore using scanFile() after which a content:// style Uri suitable for sharing is passed to the provided onScanCompleted() callback. Note that once added to the system MediaStore the content is accessible to any app on the device.

- 用系统的MediaStore（媒体库）。媒体库主要用于视频、音频和图片的MIME类型，但是从3.0开始，就可以存储非媒体类型了。文件可以用`scanFile()`来插入媒体库，可以把符合`content://`格式的Uri传入提供的`onScanComplted()`回调方法。注意，只要加到MediaStore，就能被其他app访问了。

分享多条内容：
```
ArrayList<Uri> imageUris = new ArrayList<Uri>();
imageUris.add(imageUri1); // Add your image URIs here
imageUris.add(imageUri2);

Intent shareIntent = new Intent();
shareIntent.setAction(Intent.ACTION_SEND_MULTIPLE);
shareIntent.putParcelableArrayListExtra(Intent.EXTRA_STREAM, imageUris);
shareIntent.setType("image/*");
startActivity(Intent.createChooser(shareIntent, "Share images to.."));
```

接受其他app的数据，你的activity的manifest可能长这个样子：
```
<activity android:name=".ui.MyActivity" >
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND_MULTIPLE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
</activity>
```

处理接受的Intent：
```
void onCreate (Bundle savedInstanceState) {
    ...
    // Get intent, action and MIME type
    Intent intent = getIntent();
    String action = intent.getAction();
    String type = intent.getType();

    if (Intent.ACTION_SEND.equals(action) && type != null) {
        if ("text/plain".equals(type)) {
            handleSendText(intent); // Handle text being sent
        } else if (type.startsWith("image/")) {
            handleSendImage(intent); // Handle single image being sent
        }
    } else if (Intent.ACTION_SEND_MULTIPLE.equals(action) && type != null) {
        if (type.startsWith("image/")) {
            handleSendMultipleImages(intent); // Handle multiple images being sent
        }
    } else {
        // Handle other intents, such as being started from the home screen
    }
    ...
}

void handleSendText(Intent intent) {
    String sharedText = intent.getStringExtra(Intent.EXTRA_TEXT);
    if (sharedText != null) {
        // Update UI to reflect text being shared
    }
}

void handleSendImage(Intent intent) {
    Uri imageUri = (Uri) intent.getParcelableExtra(Intent.EXTRA_STREAM);
    if (imageUri != null) {
        // Update UI to reflect image being shared
    }
}

void handleSendMultipleImages(Intent intent) {
    ArrayList<Uri> imageUris = intent.getParcelableArrayListExtra(Intent.EXTRA_STREAM);
    if (imageUris != null) {
        // Update UI to reflect multiple images being shared
    }
}
```
检查进来的数据时，一定要额外小心，你永远不知道其他app会发送**什♂么♀数据**给你。比如可能会有错的MIME type被设置，或者发送的图片超级超级大，另外，记得在子线程不要在主线程处理二进制数据。

从4.0开始，实现一个高效的、用户友好的分享操作变得更加简单。使用ActionProvider，只要依附到actionBar的一个菜单项，就可以处理外观和点击的行为。至于ShareActionProvider，你只要提供一个分享的intent它就会帮你处理好。要用SAP的话，只要在菜单项里面加一个`actionProviderClass`属性就行了：

> //注意，这段是官方提供的代码，但是亲测不能正常显示
    <menu xmlns:android="http://schemas.android.com/apk/res/android">
        <item
            android:id="@+id/menu_item_share"
            android:showAsAction="ifRoom"
            android:title="Share"
            android:actionProviderClass=
                "android.widget.ShareActionProvider" />
        ...
    </menu>
    
    //Activity 代码
    private ShareActionProvider mShareActionProvider;
    ...
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate menu resource file.
        getMenuInflater().inflate(R.menu.share_menu, menu);
        // Locate MenuItem with ShareActionProvider
        MenuItem item = menu.findItem(R.id.menu_item_share);
        // Fetch and store ShareActionProvider
        mShareActionProvider = (ShareActionProvider) item.getActionProvider();
        // Return true to display menu
        return true;
    }
    // Call to update the share intent
    private void setShareIntent(Intent shareIntent) {
        if (mShareActionProvider != null) {
            mShareActionProvider.setShareIntent(shareIntent);
        }
    }

```
//这是stackoverflow的代码，其中dante可以是你自定义的名字（"android"、"app"除外），亲测可用（不保证未来版本仍然可用）
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:dante="http://schemas.android.com/apk/res-auto" >
    <item
        android:id="@+id/menu_item_share"
        android:title="@string/menu_share"
        dante:actionProviderClass="android.support.v7.widget.ShareActionProvider"
        dante:showAsAction="ifRoom" />
</menu>
 //Activity或Fragment的代码（这里是fragment）
    @Override
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {
        getActivity().getMenuInflater().inflate(R.menu.share_menu, menu);
        MenuItem item = menu.findItem(R.id.menu_item_share);
        mShareActionProvider = (ShareActionProvider) MenuItemCompat.getActionProvider(item);
        setShareIntent();//这里同官方代码，每次更新intent的时候都要setShareIntent一下。
    }
```

###Sharing Files：

为了安全地从分享文件给其他app，你得给文件配置一个安全的处理方法，以一个内容URI的形式。android的FileProvicer组件为文件生成了内容URI，基于你在XML文件中的详细定义。
为你app定义FileProvider需要在manifest中定义一个入口(entry)，这个入口包含生成内容URI的授权信息，和XML的文件名（这个XML指定了你app能分享的目录）：
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <application
        ...>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.myapp.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
        ...
    </application>
</manifest>
```
在这里，android:authorities指定了你想用于由FileProvider生成的内容URI的URI授权。对于你自己的app来说，指定一个authority，由包名+"fileprovider"组成比较好。Provider的子元素<meta-data>指向了记录你想分享的文件夹路径的XML文件。
怎么指定你分享的文件夹呢？在你项目的`res/xml/`下创建文件`filepaths.xml`，下面的例子说明了如何分享你app的内部存储下的fiels/文件的子目录：
```
<paths>
    <files-path path="images/" name="myimages" />
</paths>
```
这里，files-path指定了分享files/images/文件夹。
>The name attribute tells the FileProvider to add the path segment myimages to content URIs for files in the files/images/ subdirectory. 至于name属性，说明了FileProvider给内容URI添加路径块，在files/images/文件下。（看不懂。。）

<paths>元素可以有多个子元素，比如<external-path>，<cache-path>，看名字就知道其作用了。那么你现在就完成了FileProvider的定义，它能为你app的内部存储中的files/目录或者其子目录来生成内容URI给文件用。当你ap为文件生成一个内容URI时，它包含了<provider>属性中指定的授权(com.example.myapp.fileprovider), 路径，(myimages/)，和文件名。比如在上面的示例中你定义的FileProvider，你对文件defaul_image.jpg获取一个内容URI，那么FileProvider就会返回这样的URI：`content://com.example.myapp.fileprovider/myimages/default_image.jpg`。（终于有点头绪了，累死了）
接下来还有几节[分享文件](http://developer.android.com/training/secure-file-sharing/share-file.html)的内容，不翻了，用到的时候再看吧。

---

[回到顶部](#menu)

## Multimedia

### Manage Audio Playback

> A good user experience is a predictable one

好的用户体验是可控的（可预测的）。如果你的app播放音视频，那么能让用户通过软件或者硬件的方式来调节设备、蓝牙、耳机的音量就很重要。（同样地，播放、暂停、停止、跳过、前一首如果需要的话，也应该执行各自的行为）。创造一个可控的音频体验的第一步，是搞清楚你app用的是哪个音频流。
系统为播放音乐，闹铃，电话铃，系统提示音，电话音量，和DTMF tones（什么鬼）都维持了一个独立的音频流。这样主要为了让用户能独立控制每个流的音量。大部分流都受限于系统事件，所以除非你app是第三方闹钟，播放音频基本上一定用的是`STREAM_MUSIC`流。大部分情况，按下音量键都会改变正在活跃的音频流，如果你app啥都没放，就只会改变手机铃声音量。如果你搞的是音乐或者游戏app，那么很可能当用户按下音量键的时候，他们想控制游戏或音乐的音量，即便可能歌曲正在切换，或者当前游戏场景没音乐。这种情况下，可能想监听音量键并修改你音频流的音量。请你忍住这个欲♂望♀。android提供了现成的`setVolumeControlStream()`方法来联系(direct)音量键和你指定的stream。 
确认了你app会用的音频流后，你应该把他设为目标声音流。你应该在app生命周期中尽早调用（onCreate），因为只需要调用一次。这方法确保无论你app是否可见，音量键总是能像用户预期的一样工作。`setVolumeControlStream(AudioManager.STREAM_MUSIC);`从这时开始，按下音量键就会影响到你指定的audio stream，无论目标activity或fragment是不是可见的。

播放、暂停、前一首等等媒体播放键在一些耳机和许多手机上都有。当用户按下这些实体键时，系统会发一个带`AXCCTION_MEDIA_BUTTON`的action的Intent的广播。为了响应它，你得注册一个BroadcastReceiver：
```
<receiver android:name=".RemoteControlReceiver">
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_BUTTON" />
    </intent-filter>
</receiver>
```
这广播的intent里包含一个EXTRA_KEY_EVENT，里面有个KeyEvent对象，你可以取出Key code来判断是哪个按键触发的广播：
```
public class RemoteControlReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (Intent.ACTION_MEDIA_BUTTON.equals(intent.getAction())) {
            KeyEvent event = (KeyEvent)intent.getParcelableExtra(Intent.EXTRA_KEY_EVENT);
            if (KeyEvent.KEYCODE_MEDIA_PLAY == event.getKeyCode()) {
                // Handle key press.
            }
        }
    }
}
```
因为可能有多个app想监听媒体键，你必须得在需要的时候来注册媒体键事件的receiver。注册后你的broadcast receiver就是所有媒体键广播的唯一的监听者。
```
AudioManager am = mContext.getSystemService(Context.AUDIO_SERVICE);
...
// Start listening for button presses
am.registerMediaButtonEventReceiver(RemoteControlReceiver);
...
// Stop listening for button presses
am.unregisterMediaButtonEventReceiver(RemoteControlReceiver);
```
一般来说，当app不活跃或者不可见的时候就该取消注册，比如在onStop里。但是，对于媒体播放的app没那么简单——实际上，反而当你app不可见的时候监听媒体键才是最重要的时候，因为它没法用屏幕上的UI来控制。所以较好的处理方法是你app获得/失去音频焦点的时候来注册/解除receiver。
为了避免多个音乐app同时播放，android使用音频焦点来使音频播放符合标准。只有持有焦点的app才应该播放音乐。在你app开始播放之前，它应该申请并接受焦点。同样滴，它应该监听啥时候失去焦点然后做出相应的反应。

对app正在使用的流用`requestAudioFocus()`，如果返回`AUDIOFOCUS_REQUEST_GRANTED`就可以获得音频焦点了。无论你想获得的是短暂还是永久的焦点，你都必须指定你在用的流。如果播放短时间的音频，比如一个说明，就用临时焦点；如果在可预见的未来范围内播放音乐就申请永久焦点，比如播放音乐。你应该在准备播放前立刻申请焦点，比如用户按下播放或者游戏下一关开始的背景音乐：
```
udioManager am = mContext.getSystemService(Context.AUDIO_SERVICE);
...

// Request audio focus for playback
int result = am.requestAudioFocus(afChangeListener,
                                 // Use the music stream.
                                 AudioManager.STREAM_MUSIC,
                                 // Request permanent focus.
                                 AudioManager.AUDIOFOCUS_GAIN);

if (result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    am.registerMediaButtonEventReceiver(RemoteControlReceiver);
    // Start playback.
}
```
当你播放结束后，一定要`abandonAudioFocus()`。这会通知系统你不再需要焦点，并解除AudioManager.OnAudioFocusChangeListener的关联。临时焦点的弃用，会让任何被中断的app继续播放。`// Abandon audio focus when playback complete    am.abandonAudioFocus(afChangeListener);`
当你申请临时焦点时，还可以额外选择是否开启"回避模式"(ducking)。通常，一个守规矩的(well-behaved)音频软件会在失去焦点时立刻静音他的播放。要求一个带回避模式的临时焦点，就意味着你告诉其他app，如果他们在焦点返回之前降低声音，还是可以继续播放滴。
```
// Request audio focus for playback
int result = am.requestAudioFocus(afChangeListener,
                             // Use the music stream.
                             AudioManager.STREAM_MUSIC,
                             // Request permanent focus.这里官方注释有错，应该是要求duck模式的临时焦点
                             AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK);

if (result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // Start playback.
}
```
duck模式断断续续地播放音频的软件尤其适合，比如外放的驾驶导航。当有其他app像上面说的一样要求音频焦点时，它是永久的还是临时的（支持或者不支持duck）焦点，都会通知到你注册焦点时的listener。`onAudioFocusChange`回调就是用来接收这个事件的。

通常来说，失去临时的焦点应该让你的app保持无声，但是其他方面应该维持状态。你应该继续监听较焦点的变化并准备在重获焦点时，恢复播放。
如果是永久性缺失焦点，你的app就该尽快结束。具体的说，就是停止播放，移除媒体键监听——让新的播放器独享这些事件——然后抛弃你的音频焦点。
在那时候，你只能期待用户操作来恢复音频播放了（按下你app的播放键）：
```
AudioManager.OnAudioFocusChangeListener afChangeListener =
    new AudioManager.OnAudioFocusChangeListener() {
        public void onAudioFocusChange(int focusChange) {
            if (focusChange == AUDIOFOCUS_LOSS_TRANSIENT) {
                // Pause playback
            } else if (focusChange == AudioManager.AUDIOFOCUS_GAIN) {
                // Resume playback
            } else if (focusChange == AudioManager.AUDIOFOCUS_LOSS) {
                am.unregisterMediaButtonEventReceiver(RemoteControlReceiver);
                am.abandonAudioFocus(afChangeListener);
                // Stop playback
            } else if (focusChange == AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK) {
            // Lower the volume
        } else if (focusChange == AudioManager.AUDIOFOCUS_GAIN) {
            // Raise it back to normal
        }
    }
        }
    };
```
你可以查询`AudioManager`来判断音频是由扬声器、有线耳机，还是蓝牙输出的：
```
if (isBluetoothA2dpOn()) {
    // Adjust output for Bluetooth.
} else if (isSpeakerphoneOn()) {
    // Adjust output for Speakerphone.
} else if (isWiredHeadsetOn()) {
    // Adjust output for headsets
} else { 
    // If audio plays and noone can hear it, is it still playing?
}
```
当耳机被拔出，或者蓝牙断开连接，音频流就能自动重新输出到内置的扬声器。如果你像我一样（谷歌工程师你好，工程师再见）听音乐声音开的很大，那样就可能会被吓一跳。幸运的是，这种情况下系统会发一个ACTION_AUDIO_BECOMING_NOISY的广播。每当你放音频的时候，注册一个BroadcastReceiver来监听这个Intent都是好习惯。
```
private class NoisyAudioStreamReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (AudioManager.ACTION_AUDIO_BECOMING_NOISY.equals(intent.getAction())) {
            // Pause the playback
        }
    }
}

private IntentFilter intentFilter = new IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY);

private void startPlayback() {
    registerReceiver(myNoisyAudioStreamReceiver(), intentFilter);
}

private void stopPlayback() {
    unregisterReceiver(myNoisyAudioStreamReceiver);
}
```

### Capturing Photos

>The world was a dismal and featureless place before rich media became prevalent. Remember Gopher? We don't, either. For your app to become part of your users' lives, give them a way to put their lives into it. Using the on-board cameras, your application can enable users to augment what they see around them, make unique avatars, look for zombies around the corner, or simply share their experiences.

在富媒体普及之前，这个世界是黯淡无光、毫无特色的。记得Gopher（WWW普及前一个信息检索网站）么？我们也忘了。为了让你的app成为用户生活的一部分，提供一种方式把他们的生活放进app中。使用自带的相机，你的app可以让用户讨论他们生活中有啥，制作自己的专属头像，寻找角落的僵尸，或者就是纯粹地分享他们的经历。

如果你app的基本功能之一是拍照，那么可以让你的app在Google Play上仅对有相机的设备可见，添加uses-feature：
```
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```
如果用，但是并不是必须的，可以设为false，那么没相机的设备也能下载到。然后在运行时检查相机可用性就是你的责任了。`hasSystemFeature(PackageManager.FEATURE_CAMERA)`。（译者注：uses-feature这个属性对于国内的store应该是没意义的）
开启拍照的代码：
```
static final int REQUEST_IMAGE_CAPTURE = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
    }
}
```
android相机会把照片编码之后作为一个小的Bitmap（只是个缩略图，适合作为icon，如果要大图还需要一些代码），放在Intent的extras里面，key是"data"：
```
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        mImageView.setImageBitmap(imageBitmap);
    }
}
```

android相机可以保存全尺寸的照片，但是你必须提供一个有效的、相机能保存的文件全名。一般来说，用户拍的照片都该保存到外部存储的公共文件夹中，这样所有app都能访问。（用getExternalStoragePublicDirectory()和DIRECTORY_PICTURES参数）读取和写入这个文件夹分别需要READ_EXTERNAL_STORAGE和 WRITE_EXTERNAL_STORAGE权限。但是如果你想让照片仅用于你的app，你可以用getExternalFilesDir（注：这个文件夹在你app卸载时也会被删掉）。用这个方法在4.4以上是不需要写入权限的。所以你可以这么玩：
```
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                     android:maxSdkVersion="18" />
    ...
</manifest>
```
决定了文件夹之后，你得创建一个防止冲突的文件名，为了方便后续的使用，你可能还需要创建成员变量。
```
String mCurrentPhotoPath;

private File createImageFile() throws IOException {
    // Create an image file name
    String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss").format(new Date());
    String imageFileName = "JPEG_" + timeStamp + "_";
    File storageDir = Environment.getExternalStoragePublicDirectory(
            Environment.DIRECTORY_PICTURES);
    File image = File.createTempFile(
        imageFileName,  /* prefix */
        ".jpg",         /* suffix */
        storageDir      /* directory */
    );

    // Save a file: path for use with ACTION_VIEW intents
    mCurrentPhotoPath = "file:" + image.getAbsolutePath();
    return image;
}
```
这方法可以为照片创建一个文件。然后你可以这么调用Intent：
```
static final int REQUEST_TAKE_PHOTO = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    // Ensure that there's a camera activity to handle the intent
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        // Create the File where the photo should go
        File photoFile = null;
        try {
            photoFile = createImageFile();
        } catch (IOException ex) {
            // Error occurred while creating the File
            ...
        }
        // Continue only if the File was successfully created
        if (photoFile != null) {
        //这里跟使用缩略图的区别就是多一个EXTRA_OUTPUT
            takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT,
                    Uri.fromFile(photoFile));
            startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO);
        }
    }
}
```
当你用这样的intent创造了一个照片时，你就知道你的图片在哪了。但是对其他人来说，访问你照片的最简单方法，就是让它能从媒体库中访问（Media Provider）。（如果你用的是getExternalFilesDir，媒体扫描器就不能访问这些文件）。
```
private void galleryAddPic() {
    Intent mediaScanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
    File f = new File(mCurrentPhotoPath);
    Uri contentUri = Uri.fromFile(f);
    mediaScanIntent.setData(contentUri);
    this.sendBroadcast(mediaScanIntent);
}
```

管理多个全尺寸的图片，对于有限的内存来说很棘手。如果你发现app显示几张图之后就OOM了，你可以通过扩展JPEG到一个已经匹配目标view的尺寸的内存数组里面，来减少动态堆的使用(you can dramatically reduce the amount of dynamic heap used by expanding the JPEG into a memory array that's already scaled to match the size of the destination view.)。
```
private void setPic() {
    // Get the dimensions of the View
    int targetW = mImageView.getWidth();
    int targetH = mImageView.getHeight();

    // Get the dimensions of the bitmap
    BitmapFactory.Options bmOptions = new BitmapFactory.Options();
    bmOptions.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    int photoW = bmOptions.outWidth;
    int photoH = bmOptions.outHeight;

    // Determine how much to scale down the image
    int scaleFactor = Math.min(photoW/targetW, photoH/targetH);

    // Decode the image file into a Bitmap sized to fill the View
    bmOptions.inJustDecodeBounds = false;
    bmOptions.inSampleSize = scaleFactor;
    bmOptions.inPurgeable = true;

    Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    mImageView.setImageBitmap(bitmap);
}
```
然后是视频的录制和获取：
```
static final int REQUEST_VIDEO_CAPTURE = 1;

private void dispatchTakeVideoIntent() {
    Intent takeVideoIntent = new Intent(MediaStore.ACTION_VIDEO_CAPTURE);
    if (takeVideoIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takeVideoIntent, REQUEST_VIDEO_CAPTURE);
    }
}

...

//retrieve this video and displays it in a VideoView
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_VIDEO_CAPTURE && resultCode == RESULT_OK) {
        Uri videoUri = intent.getData();
        mVideoView.setVideoURI(videoUri);
    }
}
```
接下来一节是[Controlling the camera](http://developer.android.com/intl/zh-cn/training/camera/cameradirect.html)，考虑到只有做相机app才能用到，就不翻了。然后接着的Printing Content也不看了。

## Graphics & Animation

### Display Bitmaps Efficiently

学会如何处理和加载Bitmap可以让你的UI组件快速响应，并避免OOM。如果不小心的话，bitmap可以快速地消耗你的内存。移动设备的内存是有限的，android设备对于每个app来说只有16m的可用内存。但是请记住，许多设备都设了一个较高的上限。像摄影图片的Bitmap是非常占内存的，比如一个500w像素的相机拍出来2592x1936像素的照片，如果用ARGB_8888格式的bitmap（默认格式）大约占据19M（2592x1936*4）内存，一下就把内存上限搞完了。一般bitmap都要比在屏幕上显示的尺寸要大，所以受限于有限的内存，我们可以使用匹配UI控件大小的低分辨率的版本。高分辨率的图像没有任何可见的好处，还会占用宝贵的内存和引发额外的性能问题。

`BitmapFactory`为创建来自不同资源的Bitmap提供了几种编译方法`decodeByteArray, decodeFile, decodeResource等`。这些方法会试图为已构建的bitmap分配内存并可以轻易导致OOM，每个编译方法都有额外的签名可以让你指定编译选项。用`BitmapFactory.Options`，设置inJustDecodeBounds为true来让编译时避免内存分配...(Setting the inJustDecodeBounds property to true while decoding avoids memory allocation, returning null for the bitmap object but setting outWidth, outHeight and outMimeType. This technique allows you to read the dimensions and type of the image data prior to construction (and memory allocation) of the bitmap.)
为了防止OOM, 在decode之前要检查bitmap的边界，除非你绝对信任数据源可以提供给你可预期的尺寸的图像，并能合适地匹配在可用内存内。
```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

>To tell the decoder to subsample the image, loading a smaller version into memory, set inSampleSize to true in your BitmapFactory.Options object. For example, an image with resolution 2048x1536 that is decoded with an inSampleSize of 4 produces a bitmap of approximately 512x384. Loading this into memory uses 0.75MB rather than 12MB for the full image (assuming a bitmap configuration of ARGB_8888). Here’s a method to calculate a sample size value that is a power of two based on a target width and height. A power of two value is calculated because the decoder uses a final value by rounding down to the nearest power of two, as per the inSampleSize documentation.

这段不好翻就懒得翻了。
```
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
        // height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) > reqHeight
                && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }

    return inSampleSize;
}
```
To use this method, first decode with inJustDecodeBounds set to true, pass the options through and then decode again using the new inSampleSize value and inJustDecodeBounds set to false:
```
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    // First decode with inJustDecodeBounds=true to check dimensions
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // Calculate inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```
然后这方法可以把一个大的bitmap加载到一个比较小的imageView中：
`mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));`

上节课中的BitmapFactory.decode*方法，如果源数据来自磁盘或者网络或者任何非内存外的地方，那么这个方法就不能执行在主线程。加载数据的时间不可预测，而且是由很多因素决定的（硬盘读取速度，网络速度，图片尺寸，CPU，etc）。那么可以用Asynctask。它提供了简单的方法来在后台线程工作，并把结果发送到主线程。示例：
```
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    private final WeakReference<ImageView> imageViewReference;
    private int data = 0;

    public BitmapWorkerTask(ImageView imageView) {
        // Use a WeakReference to ensure the ImageView can be garbage collected
        imageViewReference = new WeakReference<ImageView>(imageView);
    }

    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        data = params[0];
        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
    }

    // Once complete, see if ImageView is still around and set bitmap.
    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            if (imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```
对imageView的弱引用确保了asynctask不会影响到imageView和其引用的对象被GC（回收）。没法确定当任务完成后imageview还存在，所以你得在onPostExecute中检查它的引用。比如任务完成之前，用户离开activity或者配置改变（屏幕旋转），imageview都可能不在存在。
```
//这代码可以看出复用的重要性。
public void loadBitmap(int resId, ImageView imageView) {
    BitmapWorkerTask task = new BitmapWorkerTask(imageView);
    task.execute(resId);
}
```
常见的views比如listView和gridView，当与asynctask结合使用时，又引入另一个问题。为了高效利用内存，当用户滚动时，这些控件会回收子views。没法保证任务结束时，相关的view没有为下一个子view做准备而被回收。进一步来说，没法保证异步任务开始时的顺序和结束时的顺序（是一样的）。

//TODO 下面一段。。由于对弱引用不了解。暂不翻译
Create a dedicated Drawable subclass to store a reference back to the worker task. In this case, a BitmapDrawable is used so that a placeholder image can be displayed in the ImageView while the task completes:

 static class AsyncDrawable extends BitmapDrawable {
    private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

    public AsyncDrawable(Resources res, Bitmap bitmap,
            BitmapWorkerTask bitmapWorkerTask) {
        super(res, bitmap);
        bitmapWorkerTaskReference =
            new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
    }

    public BitmapWorkerTask getBitmapWorkerTask() {
        return bitmapWorkerTaskReference.get();
    }
}
Before executing the BitmapWorkerTask, you create an AsyncDrawable and bind it to the target ImageView:

public void loadBitmap(int resId, ImageView imageView) {
    if (cancelPotentialWork(resId, imageView)) {
        final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
        final AsyncDrawable asyncDrawable =
                new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
        imageView.setImageDrawable(asyncDrawable);
        task.execute(resId);
    }
}
The cancelPotentialWork method referenced in the code sample above checks if another running task is already associated with the ImageView. If so, it attempts to cancel the previous task by calling cancel(). In a small number of cases, the new task data matches the existing task and nothing further needs to happen. Here is the implementation of cancelPotentialWork:

public static boolean cancelPotentialWork(int data, ImageView imageView) {
    final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

    if (bitmapWorkerTask != null) {
        final int bitmapData = bitmapWorkerTask.data;
        // If bitmapData is not yet set or it differs from the new data
        if (bitmapData == 0 || bitmapData != data) {
            // Cancel previous task
            bitmapWorkerTask.cancel(true);
        } else {
            // The same work is already in progress
            return false;
        }
    }
    // No task associated with the ImageView, or an existing task was cancelled
    return true;
}
A helper method, getBitmapWorkerTask(), is used above to retrieve the task associated with a particular ImageView:

private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
   if (imageView != null) {
       final Drawable drawable = imageView.getDrawable();
       if (drawable instanceof AsyncDrawable) {
           final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
           return asyncDrawable.getBitmapWorkerTask();
       }
    }
    return null;
}
The last step is updating onPostExecute() in BitmapWorkerTask so that it checks if the task is cancelled and if the current task matches the one associated with the ImageView:

 class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (isCancelled()) {
            bitmap = null;
        }

        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            final BitmapWorkerTask bitmapWorkerTask =
                    getBitmapWorkerTask(imageView);
            if (this == bitmapWorkerTask && imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
This implementation is now suitable for use in ListView and GridView components as well as any other components that recycle their child views. Simply call loadBitmap where you normally set an image to your ImageView. For example, in a GridView implementation this would be in the getView() method of the backing adapter.

加载单个bitmap到界面上是很简单的，但是如果你需要一次加载一大堆图片，就有点复杂了。许多情况下（比如用GridView或者ViewPager,RecyclerView）屏幕上的图片数和即将滚动到屏幕的图片理论上可以是无限的。这些组件通过当子views离开屏幕时对其进行回收，降低了内存的使用。GC还会在你没有保有长期引用时释放你加载的bitmap。一切都是那么棒棒哒(be all good and well)，但是为了保持流畅和快速加载UI，你要避免每次图片回到屏幕时都反复处理图片。那么内存和磁盘（内部存储）缓存就派上用场了，可以让控件快速重新加载、处理图片。（译者注：我们可以通过第三方加载图片库来达到这个目的，但是翻译这里是为了了解机制和思想）

内存缓存(memory cache)以占据应用内存空间为代价，提供了对bitmap的快速访问。LruCache尤其适合缓存bitmap的任务，在一个LinkedHashMap中持有最近的引用对象，并在缓存超限之前释放了最少被使用的成员。注意，在以前，流行的内存缓存的实现是用弱引用或者软引用 bitmap cache，但这并不推荐。从android2.3GC对于弱引用和软引用的回收变得更剧烈(aggressive)，这让他们相当低效(faily ineffective)。3.0之前，bitmap数据的备份被存在本地内存且不会被可预期的方法释放，潜在地可能导致OOM。
为了选择合适大小的LruCache，一些因素需要考虑：
- 你的activity/application 剩余的内存有多密集(intentsive)？
- 屏幕一次加载多少图片？有多少需要准备好以显示到屏幕上？
- 设备屏幕尺寸和DPI是多大？一个xhdpi的设备相对于低dpi的设备会需要更大的缓存来在内存中hold相同数量的图片。
- bitmaps的尺寸和配置是什么样的？每个bitmap会占据多大内存？
- 图片会被访问多频繁？会不会有一些比其他的访问更频繁？如果是，也许你应该保持特定项在一直在内存里，或使用多个LruCache对象来应对不同组bitmaps。
- 你能权衡质量和数量么(balance quality against quantity)？有时候存储大量低质量的bitmaps更有用，潜在地(potentially)在后台任务中再加载高质量的版本。
没有特定的尺寸或者公式能适配所有app，取决于你来分析使用情况并做出合适的方案。太小的cache可能导致毫无好处的额外开支(overhead)，太大的cache可能导致OOM，并让app所剩内存无几。示例：
```
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Get max available VM memory, exceeding this amount will throw an
    // OutOfMemory exception. Stored in kilobytes as LruCache takes an
    // int in its constructor.
    final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

    // Use 1/8th of the available memory for this memory cache.
    final int cacheSize = maxMemory / 8;

    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap bitmap) {
            // The cache size will be measured in kilobytes rather than
            // number of items.
            return bitmap.getByteCount() / 1024;
        }
    };
    ...
}

public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }
}

public Bitmap getBitmapFromMemCache(String key) {
    return mMemoryCache.get(key);
}
```
这个例子中，1/8的app内存分配给(allocated for)缓存，在一个普通的hdpi的设备，这是大概4MB的最小值(a minimum of around 4MB)，（32/8=4）
所以这大概能在内存里缓存2.5页的图片。当加载bitmap到imageview中时，LruCache先被检查。如果入口找到了，那么会立刻使用来更新imageview，否则会产生一个后台线程来处理图片：
```
public void loadBitmap(int resId, ImageView imageView) {
    final String imageKey = String.valueOf(resId);

    final Bitmap bitmap = getBitmapFromMemCache(imageKey);
    if (bitmap != null) {
        mImageView.setImageBitmap(bitmap);
    } else {
        mImageView.setImageResource(R.drawable.image_placeholder);
        BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }
}
```
BitmapWorkerTask也需要更新以增加入口(entries)到内存缓存中：
```
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final Bitmap bitmap = decodeSampledBitmapFromResource(
                getResources(), params[0], 100, 100));
        addBitmapToMemoryCache(String.valueOf(params[0]), bitmap);
        return bitmap;
    }
    ...
}
```

memory cache对于提升访问最近的bitmaps速度来说非常有用，但是你不能依赖这个方法。像GridView这样带有大量数据源的控件可以轻松填满内存空间。你的app不能被另一个任务比如电话打断，而且在后台时它可能就被干掉，然后内存缓存就会被销毁。一旦用户恢复了，a你又不得不再次处理每个图片。磁盘缓存(disk cache)就适用于这些情况，它能持久保存(persist)处理后的bitmaps，并在图片不再在内存中可用的时候减少加载事迹。当然从磁盘解析图片要比从内存中慢。而且应该在后台搞定，因为磁盘读取时间是不可预测的。
如果需要更频繁地访问，也许ContentProvider是个存储缓存的更佳地点。比如图片浏览app。
下面代码演示了添加disk cache到已经存在的memory cache中：
```
private DiskLruCache mDiskLruCache;
private final Object mDiskCacheLock = new Object();
private boolean mDiskCacheStarting = true;
private static final int DISK_CACHE_SIZE = 1024 * 1024 * 10; // 10MB
private static final String DISK_CACHE_SUBDIR = "thumbnails";

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Initialize memory cache
    ...
    // Initialize disk cache on background thread
    File cacheDir = getDiskCacheDir(this, DISK_CACHE_SUBDIR);
    new InitDiskCacheTask().execute(cacheDir);
    ...
}

class InitDiskCacheTask extends AsyncTask<File, Void, Void> {
    @Override
    protected Void doInBackground(File... params) {
        synchronized (mDiskCacheLock) {
            File cacheDir = params[0];
            mDiskLruCache = DiskLruCache.open(cacheDir, DISK_CACHE_SIZE);
            mDiskCacheStarting = false; // Finished initialization
            mDiskCacheLock.notifyAll(); // Wake any waiting threads
        }
        return null;
    }
}

class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final String imageKey = String.valueOf(params[0]);

        // Check disk cache in background thread
        Bitmap bitmap = getBitmapFromDiskCache(imageKey);

        if (bitmap == null) { // Not found in disk cache
            // Process as normal
            final Bitmap bitmap = decodeSampledBitmapFromResource(
                    getResources(), params[0], 100, 100));
        }

        // Add final bitmap to caches
        addBitmapToCache(imageKey, bitmap);

        return bitmap;
    }
    ...
}

public void addBitmapToCache(String key, Bitmap bitmap) {
    // Add to memory cache as before
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }

    // Also add to disk cache
    synchronized (mDiskCacheLock) {
        if (mDiskLruCache != null && mDiskLruCache.get(key) == null) {
            mDiskLruCache.put(key, bitmap);
        }
    }
}

public Bitmap getBitmapFromDiskCache(String key) {
    synchronized (mDiskCacheLock) {
        // Wait while disk cache is started from background thread
        while (mDiskCacheStarting) {
            try {
                mDiskCacheLock.wait();
            } catch (InterruptedException e) {}
        }
        if (mDiskLruCache != null) {
            return mDiskLruCache.get(key);
        }
    }
    return null;
}

// Creates a unique subdirectory of the designated app cache directory. Tries to use external
// but if not mounted, falls back on internal storage.
public static File getDiskCacheDir(Context context, String uniqueName) {
    // Check if media is mounted or storage is built-in, if so, try and use external cache dir
    // otherwise use internal cache dir
    final String cachePath =
            Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState()) ||
                    !isExternalStorageRemovable() ? getExternalCacheDir(context).getPath() :
                            context.getCacheDir().getPath();

    return new File(cachePath + File.separator + uniqueName);
}
```
纵使初始化磁盘缓存需要磁盘操作，因此不应该发生在主线程，但是捏，这并不代表缓存在初始化之前不可访问。为了处理这个，上面的实现中，一个锁对象确保了直到缓存被初始化app不从磁盘缓存中读取。相比内存缓存在UI线程被检查后，磁盘缓存在后台被检查。磁盘操作绝对不该在主线进行。(Disk operations should never take place on the UI thread.) 当图片处理完成后，最终的bitmap被加到内存和磁盘缓存以便未来的使用。

>Luckily, you have a nice memory cache of bitmaps that you built in the Use a Memory Cache section. This cache can be passed through to the new activity instance using a Fragment which is preserved by calling setRetainInstance(true)). After the activity has been recreated, this retained Fragment is reattached and you gain access to the existing cache object, allowing images to be quickly fetched and re-populated into the ImageView objects.

运行时的配置改变比如屏幕方向改变，导致android销毁并restart activity以应用新配置。你肯定想避免不得不重新处理所有图片以让用户有平滑、快速的体验。幸运的是，你拥有了建立好的bitmap的内存缓存。这个缓存可以传到新的activity实例，使用由于调用了`setRetainInstance(true)`被保存的fragment。activity被重建后，保留的fragment会被重新attach，然后你可以访问已经存在的cache对象，这可以让图片快速解析并populate到UI中。下面是使用fragment的例子：
```
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    RetainFragment retainFragment =
            RetainFragment.findOrCreateRetainFragment(getFragmentManager());
    mMemoryCache = retainFragment.mRetainedCache;
    if (mMemoryCache == null) {
        mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
            ... // Initialize cache here as usual
        }
        retainFragment.mRetainedCache = mMemoryCache;
    }
    ...
}

class RetainFragment extends Fragment {
    private static final String TAG = "RetainFragment";
    public LruCache<String, Bitmap> mRetainedCache;

    public RetainFragment() {}

    public static RetainFragment findOrCreateRetainFragment(FragmentManager fm) {
        RetainFragment fragment = (RetainFragment) fm.findFragmentByTag(TAG);
        if (fragment == null) {
            fragment = new RetainFragment();
            fm.beginTransaction().add(fragment, TAG).commit();
        }
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setRetainInstance(true);
    }
}
```
测试的话，可以用retain，然后不retain，看看旋转屏幕的反应。你会注意到几乎没啥延迟，当你保持了cache的时候(You should notice little to no lag as the images populate the activity almost instantly from memory when you retain the cache.)。内存缓存找不到的图片最好要在disk里面能找到，不然就会正常处理。

android3.0之后，引入了`BitmapFactory.Options.inBitmap`成员，如果设了这个选项，使用这个Options对象的编码方法在加载内容时会尝试重用已经存在的bitmap。这意味着bitmap的内存会被复用，因此提升性能。接下来不翻译了。。[Managing Bitmap Memory](http://developer.android.com/intl/zh-cn/training/displaying-bitmaps/manage-memory.html)

这节课把之前的融合到一起，告诉你如何在后台线程加载多个bitmap到ViewPager和GridView并缓存bitmap，同时处理了并发和配置改变。滑动的view模式是一个导航图片库的详情的极佳方式。你可以通过使用由PagerAdapter支持的ViewPager来实现。但是捏，更爽的支持adapter(backing adapter)是用`FragmentStateAdapter`的子类，当fragment离开屏幕时，它可以自动销毁和保存Viewpager中fragments状态并且占用较低内存。如果你的图片数量不多，并且确定他们不会超出应用内存上限，那就用普通的PagerAdapter或者FragmentPagerAdapter可能更合适。
接下来的代码好像也不太实用，不翻译了。。[Displaying Bitmaps in Your UI](http://developer.android.com/intl/zh-cn/training/displaying-bitmaps/display-bitmap.html)

### [Displaying Graphics with OpenGL ES](http://developer.android.com/intl/zh-cn/training/graphics/opengl/index.html)
android框架提供了大量标准工具来创建吸引人的，功能多样的图像界面。但是，如果你想要对你app在屏幕上绘制的东西有更多的控制，或者想到3d图像里探险，你需要使用一种与众不同的东西。Android框架提供的OpenGL ES接口提供了一组用来显示高级的，动态图像，只有你想不到没有它做不到，而且还可以享受到很多设备上都有的图形处理单元加速(GPU)带来的好处
这节课带你学习使用OpenGL进行开发的基础，包括配置、绘制对象、移动绘制元素和响应触摸输入。（比较高级，老规矩，不翻译）

### Animating Views Using Scenes and Transitions

为了响应用户输入和其他事件，activity的ui经常会变化。比如包含用户可以输入查询的表格的activity可以当用户提交的时候隐藏这个表格并展现一个搜索结果的列表。这些情况下，为了提供视觉上的连贯，你可以在你的UI中的不同view树(hhierarchies)之间做出动画。这些动画给用户操作的反馈并且帮助他们理解你app是怎么工作的。
android包含了*transitions*框架，这可以让你在不同的view树之间轻松生成动画。框架通过根据时间的变化改变他们的属性值来为views产生运行时的动画。框架内置常见的特效动画，并让你创建自定义动画和变化的生命周期回调(trnasition lifecycle callbacks)。
对于API 14(4.0) ~ 19(4.4.2)的版本，用`animateLayoutChanges`属性来对layouts进行动画。(也就是说其他版本不需要用这个属性？)
框架有这些特性：
- 组级别(Group-level)的动画：应用一或多个动画效果到view树中的所有views上。
- 基于转换(transition)的动画：基于开始和结束view的属性值变化的animation
- 内置动画：包括定义好的常见的特效比如淡入，移动
- 支持资源文件：从layout资源文件加载view树和内置动画
- 生命周期回调：定义了提供对animation和树变化过程的精确控制的回调(Defines callbacks that provide finer control over the animation and hierarchy change process.)

>The transitions framework provides abstractions for scenes, transitions, and transition managers. These are described in detail in the following sections. To use the framework, you create scenes for the view hierarchies in your app that you plan to change between. Next, you create a transition for each animation you want to use. To start the animation between two view hierarchies, you use a transition manager specifying the transition to use and the ending scene. This procedure is described in detail in the remaining lessons in this class.'

框架对所有俩view树种的views的改变进行了动画。一个view树可能简单的只有一个view，也可能复杂地有一个ViewGroup包含一组精密的views。框架通过改变从开始view树到最终view树之间随着时间，改变一个或者更多属性值来动画每个view。transitions框架与view树和动画相关联。框架的目的就是存储view树的状态，在这些树之间改变以修改设备屏幕外观，并通过存储和应用动画描述(animation definitions)来动画这种改变。

一个存储view树的场景，包括所有的view和其属性值。一个view树可以是简单的view或者复杂的views和子布局。在场景里存储views树状态使你可以把它转换到另一个场景。transition框架允许你从布局资源文件或者代码里的ViewGroup对象创建场景(Scene)。如果需要动态创建view树或者运行时修改它，在代码里创建场景就很实用。

大多数情况下(In most cases)你不需要明显地(explicitly)创建一个开始场景。如果你已经应用了一个transition，框架会用之前的结束场景作为接下来transition的开启场景。如果你没有，框架会收集当前屏幕上views的状态信息。一个场景可以定义自己的，当场景改变时的动作。这个特性在你transition到一个场景时清理view的设置很有用。(this feature is useful for cleaning up view settings after you transition to a scene.)
除了view树和属性值，场景还可以存储view树的父类引用。这个root view叫scene root。影响到场景的场景和动画的改变发生在scene root内。

transition框架中，动画创建了一系列描绘了开始和结束场景之间的改变的帧画面(frame)。关于动画的信息存储在transition对象中。为了开启动画，你得用一个`TransitionManager`实例来应用transition。框架可以在俩不同的场景，或者当前场景的不同状态间进行transition。

框架包含了一系列内置的transitions以应付常用的动画特效，比如淡化(fading)和改变views的大小(resizing)。你还可以用动画框架的API来定义自己的transitions来创建动画特效。transition框架还让你能在一个包含一组独立的内置或者自定义的transitions的transition集合中组合不同的动画效果。(The transitions framework also enables you to combine different animation effects in a transition set that contains a group of individual built-in or custom transitions.)
transition的生命周期与activity很类似，它代表了框架监视的开始和完成动画之间的transition的状态。在重要的生命周期状态上，框架调用了回调方法，你可以implement来对你UI在transition的不同阶段中进行调整。

transition框架的限制(limitations)：
- 应用到SurfaceView的动画可能无法正确显示。SurfaceView示例由非主线程更新，所以更新可以与其他view的动画不同步。
- 某些特定transition类型应用到TextureView时可能无法产生预期的动画效果。
- 继承AdapterView的类，比如ListView，管理子views的方式与transition框架不兼容，如果你想动画基于AdapterView的views，设备的显示可能会挂起(hang)
- *如果你试图用动画来resize一个TextView，在它完全resized之前，文本可能会跳到一个新的位置。为了避免这个问题，不要动画包含文本的views。*

你可以直接从布局文件创建一个`Scene`实例，当文件中的view树大多情况下是固定不变的时候你就可以这么做。这样的场景(The resulting scene)代表了你创建场景实例时的view树的状态。如果你改变了view树，你得重新创建场景。框架从文件中的整个view树创建了场景，而不能从其一部分来创建。
为了从布局文件场景场景实例，用你布局的ViewGroup来解析根场景(scene root)，然后调用`Scene.getSceneForLayout()`，传入根场景和布局文件id。

 下面的代码展示了如何用相同的场景根元素来创建不同的两个场景。还演示了你可以不用暗示他们是有关联的来加载多个未关联的场景对象(you can load multiple unrelated Scene objects without implying that they are related to each other)。
 示例由下面的定义好的布局组成：
 - 主布局是一个包含文本标签和子布局的activity的布局。
 - 第一个场景是带有俩文本区域的相对(or相关？)布局
 - 第二个场景的布局也有俩文本，但是顺序不同。
 示例特意这么设计这一所有的动画都能在activity的主布局的子布局中产生。主布局的文本保持不变。
主布局长这个样子：
res/layout/activity_main.xml
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/master_layout">
    <TextView
        android:id="@+id/title"
        ...
        android:text="Title"/>
    <FrameLayout
        android:id="@+id/scene_root">
        <include layout="@layout/a_scene" />
    </FrameLayout>
</LinearLayout>
```
这个布局包含一个文本和一个用来作为scene root的子布局。用于第一个场景的布局包括在主布局内。这使得app可以将它显示为初始界面并把它加载到场景中，因为框架只能把一整个布局文件加载到场景中。(The layout for the first scene is included in the main layout file. This allows the app to display it as part of the initial user interface and also to load it into a scene, since the framework can load only a whole layout file into a scene.)

First scene的布局文件(被包括到主布局中了)：
res/layout/a_scene.xml
```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/scene_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    <TextView
        android:id="@+id/text_view1
        android:text="Text Line 1" />
    <TextView
        android:id="@+id/text_view2
        android:text="Text Line 2" />
</RelativeLayout>
```
Second scene的布局也包括俩文本(id相同)但是摆放顺序不同：
res/layout/another_scene.xml
```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/scene_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    <TextView
        android:id="@+id/text_view2
        android:text="Text Line 2" />
    <TextView
        android:id="@+id/text_view1
        android:text="Text Line 1" />
</RelativeLayout>
```

当你定义好俩相对布局后，你就可以分别获得scene对象了。这样使你之后可以在两种UI之间转换。为了获得场景，你需要一个根场景和布局id的引用。:
```
Scene mAScene;
Scene mAnotherScene;

// Create the scene root for the scenes in this app
mSceneRoot = (ViewGroup) findViewById(R.id.scene_root);

// Create the scenes
mAScene = Scene.getSceneForLayout(mSceneRoot, R.layout.a_scene, this);
mAnotherScene =
    Scene.getSceneForLayout(mSceneRoot, R.layout.another_scene, this);
```
app中，现在有俩基于view树的Scene对象，他们都是用 res/layout/activity_main.xml中的`FrameLayout`元素定义的根场景。

你也可以代码中由ViewGroup对象来创建Scene。当你需要直接修改view树或者需要动态生成时，你可以这么做。
`Scene(sceneRoot, viewHierachy)`构造方法来从view树中创建场景。它跟Scene.getSceneForLayout()方法效果是一样一样的

代码中创建scene的示例：
```
Scene mScene;

// Obtain the scene root element
mSceneRoot = (ViewGroup) mSomeLayoutElement;

// Obtain the view hierarchy to add as a child of
// the scene root when this scene is entered
mViewHierarchy = (ViewGroup) someOtherLayoutElement;

// Create a scene
mScene = new Scene(mSceneRoot, mViewHierarchy);
```
框架可以让你定义进入或者退出场景时的场景动作。很多情况下，自定义场景动作都不是必须的，因为框架自动在场景间做出动画改变。
Scene actions在这些情况很有用：动画不在同一个view树的views。你可以用退出和进入的场景动作来动画开启和结束场景的views；
动画哪些transition框架不能自动动画的views。比如ListView对象。更多信息，看『[Limitations](http://developer.android.com/training/transitions/overview.html#Limitations)』
为了提供自定义场景动作，把你的actions定义为Runnable对象，并传入Scene.setExitAction()或者setEnterAction()。框架会在运行transition动画之前在启动场景执行setExitAction，在运行完动画后在结束场景里执行setEnterAction(这里文档可能写的有问题，应该是执行其设定的Action而不是setXXAction这一方法)。注意：**不要**用场景动作来在开启和结束场景的**views间传数据**。

这节课教你在俩场景之间，用内置的transition来移动，变大变小，淡化views。上一课中，你学会了怎么创建表示不同view树的状态的场景了。一旦你定义好开启和结束场景，你得创建一个定义了动画的`Transition`对象。框架允许你在资源文件中指定内置的transition，并在代码中填充它(inflate)，或者直接在代码里创建内置的transition实例。

    Class       |  Tag|    Attributes| Effect
---------------|-----|--------------|-------
AutoTransition |<autoTransition/>|-|默认transition. 淡出，移动，调整大小，淡入views，这个顺序。
Fade|<fade/>|android:fadingMode="[fade_in| fade_out| fade_in_out]"|fade_in_out (default) 先淡出后淡入。
ChangeBounds|<changeBounds/>|-|移动并调整views大小。

从资源文件创建transition实例。这招可以让你不用改activity的代码就能修改你的transition定义。对于从你app代码中分离复杂的transition定义也很有用。
在资源文件中指定内置的transition，按照这个步骤：
1. 在项目中加入`res/transition`文件夹
2. 在这个文件夹中创建一个xml文件
3. 为了内置的transition，加一个xml节点

例如，下面的文件指定了Fade transition:
res/transition/fade_transition.xml
`<fade xmlns:android="http://schemas.android.com/apk/res/android" />`
然后你得在activity里填充(inflate)这个transition示例：
```
Transition mFadeTransition =
        TransitionInflater.from(this).
        inflateTransition(R.transition.fade_transition);
```
这招对于动态创建transition对象很好用。如果你在代码里修改UI，并创建简单的参数很少或者没有的内置transition的话。
为了创建内置transition的实例，调用transition子类的构造方法。比如： `Transition mFadeTransition = new Fade();`
一般为了响应一个事件比如用户操作时你会应用transition来在不同views树之间改变。比如，假设有一个搜索app：用户输入搜索词汇点击搜索，app变为便是搜索结果的布局的场景，这时app淡出搜索按钮，淡入搜索结果的transition。
当你为了响应activity的某些事件而做出场景的变化时，调用`TransitionManager.go()`静态方法传入结束场景和用于动画的transition实例：`TransitionManager.go(mEndingScene, mFadeTransition);`框架会用结束场景的view树来改变根场景内的view树，同时应用transition实例指定的animation。开始场景就是上个transition的结束场景。如果没有，开启场景就会自动选择UI的当前状态。
如果你没指定transition实例，transition管理器可以应用一个自动会在大多数合理处理的transition（an automatic transition that does something reasonable for most situations）。

框架默认应用transition到开启和结束场景中所有的views上。某些情况你可能只想应用到场景的一部分views中。比如，框架不支持应用动画到ListView对象，所以你不该试着在transition中动画他们。框架允许你选择你想动画的特定views。每个transition动画的view都是一个`target`，你只能选择与场景关联的view树的view作为target。为了移除/增加target表中的view，在开启transition之前调用add/removeTarget()即可。 

为了从animation获得最大的效果(get the most impact from an animation)，你应该将其匹配到场景的改变的类型(match it to the type of changes that occur between the scenes)。比如，如果你在场景间移除并添加了一些views，一个淡入淡出动画提供了很好的暗示表示一些views不再存在。如果你在屏幕移动views到不同的地方，动画移动过程可能是更好的选择，因为这样用户能注意到views的新地点。

没必要只选一个动画，因为transition框架允许你在一个包含一组独立的transition的集合中，组合动画特效。
为了在xml中从一组transition集合中定义一个transition set，在`res/transitions/`文件夹下创建资源文件，并罗列所有transitions在`transisiotnSet`元素下。比如下面代码展示如何制定一个与AutoTransition类有相同行为的transition集合。
```
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:transitionOrdering="sequential">
    <fade android:fadingMode="fade_out" />
    <changeBounds />
    <fade android:fadingMode="fade_in" />
</transitionSet>
```
用`TransitionInflater.from()`来填充为TransitionSet对象。它继承自Transition，这样就像其他transition实例一样，你可以通过transition manager来用它。

改变view树不是修改UI的唯一方法。你还可以在当前树内通过修改、添加，移除子views来做出改变。比如你可以只用一个layout就实现搜索交互。通过带一个搜索框和搜索icon的layout开始，为了改变UI来展示结果，当用户点击搜索按钮的时候，通过`ViewGroup.removeView()`方法来移除按钮，然后通过`ViewGroup.addView()`来添加搜索结果。如果是俩几乎相同的view树，那么你可能想用这个方法。比起为了UI上小小的区别不得不创建和维护俩独立的layout文件，显然在代码里修改包含一个view树的一个layout文件比较方便。
如果你用这个方法(in this fashion)在当前view树中做出改变，就不需要创建一个scene。作为替代，你可以在view树的俩状态间使用一个delayed transition。这个transitions framework的特性由当前的view树的状态开始，记录了你对views做出的changes，然后当系统重绘UI时应用动画了这些改变的transition。

在一个view树内创建延迟transition，按照这个步骤：
1. 当触发transition时，调用TransitionManager.beginDelayedTransition()方法，传入你要改变的views的父view(providing the parent view of all the views you want to change and the transition to use.)框架会存储当前子views的状态和属性值。
2. 按照需要对子views做出变化，框架会记录这些改变和他们的属性值。
3. 当系统根据你的改变重绘UI时，框架会根据初始和新状态来动画这些改变。
下面演示使用delayed transition如何动画textview添加到一个view树：
res/layout/activity_main.xml
```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/mainLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    <EditText
        android:id="@+id/inputText"
        android:layout_alignParentLeft="true"
        android:layout_alignParentTop="true"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
    ...
</RelativeLayout>
```
MainActivity.java
```
private TextView mLabelText;
private Fade mFade;
private ViewGroup mRootView;
...

// Load the layout
this.setContentView(R.layout.activity_main);
...

// Create a new TextView and set some View properties
mLabelText = new TextView();
mLabelText.setText("Label").setId("1");

// Get the root view and create a transition
mRootView = (ViewGroup) findViewById(R.id.mainLayout);
mFade = new Fade(IN);

// Start recording changes to the view hierarchy
TransitionManager.beginDelayedTransition(mRootView, mFade);

// Add the new TextView to the view hierarchy
mRootView.addView(mLabelText);

// When the system redraws the screen to show this update,
// the framework will animate the addition as a fade in
```

transition的生命周期跟activity有点像，它代表了框架从TransitionManager.go()到动画完成之间的transition状态。在重要的生命周期状态时，框架会调用通过`TransitionListener`接口定义的回调。transition生命周期回调挺有用的。比如，从开始vie树复制一个view的property到结束view树。你并不能简单地复制，因为结束的view树直到动完成后才被inflated。Instead，你得存到一个变量中，然后等框架完成transition后复制到结束的view树中(在activity中实现`TransitionListener.onTransitionEnd()`)。

创建自定义的transition，例如，你可以定义一个transition使text的前景色(foreground color)和输入框变成灰色(暗示它在新场景中已经不可用)自定义的transition就像内置类型的一样，会应用动画到开启和结束场景中的子view。但是，你得提供**获取属性值和生成动画**的代码。你可能还想为你的animation定义一个target views的子集。自定义的transition长这个样子：
```
public class CustomTransition extends Transition {
    @Override
    public void captureStartValues(TransitionValues values) {}

    @Override
    public void captureEndValues(TransitionValues values) {}

    @Override
    public Animator createAnimator(ViewGroup sceneRoot,
                                   TransitionValues startValues,
                                   TransitionValues endValues) {}
}
```

transition的动画使用的是Property Animation。它会在特定的内一段时间改变一个view属性(Property animations change a view property between a starting and ending value over a specified period of time)，这样框架就需要这个属性的开始值和结束值来构建animation。
但是捏，property animation通常只需要所有view的属性值的一个子集(a small subset of all the view's property values)比如，一个颜色动画需要颜色属性值，而移动动画只需要位置属性值。因此框架不提供全部的属性值给一个transition。Instead, 框架调用回调方法，在里面可以让transition获取它需要的属性值并存储到框架中。

为了传递开始view的值给框架，实现captureStartValues(transitionValues)方法即可。框架在开始场景中为每个view调用这个方法。其参数是一个TransitinoValue对象，它包含了对view的引用和一个Map对象，in which你可以存储你需要的view值。在你的实现中，取回(retrieve)这些属性值并通过存在map中来传递给框架。为了确保属性值的key不会和其他TransitionValues的key冲突，遵循这个命名规则： `package_name:transition_name:property_name`，下面是实现captureStartValues方法的实例：
```
public class CustomTransition extends Transition {

    // Define a key for storing a property value in
    // TransitionValues.values with the syntax
    // package_name:transition_class:property_name to avoid collisions
    private static final String PROPNAME_BACKGROUND =
            "com.example.android.customtransition:CustomTransition:background";

    @Override
    public void captureStartValues(TransitionValues transitionValues) {
        // Call the convenience method captureValues
        captureValues(transitionValues);
    }


    // For the view in transitionValues.view, get the values you
    // want and put them in transitionValues.values
    private void captureValues(TransitionValues transitionValues) {
        // Get a reference to the view
        View view = transitionValues.view;
        // Store its background property in the values map
        transitionValues.values.put(PROPNAME_BACKGROUND, view.getBackground());
    }
    ...
}
```
同样低，框架会在结束场景为每个target view调用captureEndValues(TransitionValues)：
```
@Override
public void captureEndValues(TransitionValues transitionValues) {
    captureValues(transitionValues);
}
```
这个例子，captureStartValues和xxxEndXXXX都调用captureValues放阿飞来获取并存储values。这个方法获取的view属性是相同的，但是在开始和结束场景中的值是不同的。框架为开始和结束状态的view保留了不同的maps。

为了给开始和结束场景中view的状态变化以动画特效，你得复写createAnimator方法来提供一个animator。当框架调用这个方法时，它在sceene root 中传递view和TransitionValues对象(包含了你capture的开始和结束值)。
框架调用createAnimator()方法的次数取决于发生在开始和结束场景间的变化情况。例如，假设一个淡入淡出动画被实现为自定义transition，如果开始场景有5个（2个结束时被移除）targets，结束场景有3个开始场景中的target，和一个新增的target，框架就会调用6次createAnimator：3次调用淡出和淡入的两边场景都有的targets，2次调用移除的targets的动画，还有一次新target的淡入动画。
对于两遍都存在的target views，框架提供了TransitionValues对象给startValues和endValues参数。。对于只存在开始或者结束场景的views，只要把相应的参数改为null即可。
创建自定义transition的时候，实现`createAnimator(ViewGroup, TransitionValues, TransitionValues) `方法，使用你获得的view属性值来创建animator对象并返回给框架。具体实例可以看看[CustomTransition](http://developer.android.com/samples/CustomTransition/index.html)；关于属性动画可以看[PropertyAnimation](http://developer.android.com/guide/topics/graphics/prop-animation.html)。

### Adding Animations

动画添加了细微的视觉线索，可以通知用户你app是啥情况并且提高对你app界面的认知(mental model of your app's inteerface)。动画在屏幕状态改变时尤其有用，比如加载内容或者新操作可用时。动画还能增加你app的颜值，给你app一种高大上的赶脚(a higher quality feel)。请记住，过度使用动画或者不合时宜地使用都可能是有害的，比如他们可用造成延迟。

淡入淡出（crossfading）动画（又叫溶解dissolve）逐渐地淡出一个UI控件，同步地(simultaneously)淡入另外一个控件。这个动画在你想切换内容或者views时很实用。Crossfades非常细微、短暂，但是提供了从一个屏幕到下一个的流畅转换。不管怎么说，如果你不用它，转换就感觉很急促、突然。

创建你想crossfade的俩views，示例中包含了一个progress指示器和一个可滚动的textview：
```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/content"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView style="?android:textAppearanceMedium"
            android:lineSpacingMultiplier="1.2"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@string/lorem_ipsum"
            android:padding="16dp" />

    </ScrollView>

    <ProgressBar android:id="@+id/loading_spinner"
        style="?android:progressBarStyleLarge"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center" />

</FrameLayout>
```
设置动画：
1. 为你想crossfade的views创建成员变量。之后在动画过程中改动views时会用上这些引用。
2. 对即将要淡入的view，设置可见性为GONE。这防止view占据布局空间，还能从布局计算中剔除，加快了处理速度。
3. 在一个成员变量中缓存`config_shortAnimTime`系统属性。这个属性为动画定义了标准的"short"持续时间。这个持续时间对于轻微的或者频繁的动画很适合。（如果你想用的话，config_longAnimTime, config_mediumAnimTime也可以）
用之前的layout作为activity的content view:
```
public class CrossfadeActivity extends Activity {

    private View mContentView;
    private View mLoadingView;
    private int mShortAnimationDuration;

    ...

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_crossfade);

        mContentView = findViewById(R.id.content);
        mLoadingView = findViewById(R.id.loading_spinner);

        // Initially hide the content view.
        mContentView.setVisibility(View.GONE);

        // Retrieve and cache the system's default "short" animation time.
        mShortAnimationDuration = getResources().getInteger(
                android.R.integer.config_shortAnimTime);
    }
```
views准备好后，通过下面的步骤来crossfade:
1. 对于要淡入的view，把alpha值设为0，可见性设为VISIBLE(一开始是GONE)，这会让view可见，但是完全透明。
2. 对于要淡入的view，动画它的alpha值使他从0到1。与此同时，对于要淡出的view，alpha值从1到0.
3. 在`Animator.AnimatorListener`中用`OnAnimationEnd()`方法，把淡出的view的可见性设为GONE。尽管其alpha值是0，设为GONE后可以防止其占据空间和布局计算。
```
private View mContentView;
private View mLoadingView;
private int mShortAnimationDuration;
...

private void crossFade(){
    // Set the content view to 0% opacity but visible, so that it is visible
    // (but fully transparent) during the animation.
    mContentView.setAlpha(0f);
    mContentView.setVisibility(View.VISIBLE);
    
    // Animate the content view to 100% opacity, and clear any animation
    // listener set on the view.
    mContentView.animate()
            .alpha(1f)
            .setDurartion(mShortAnimationDuration)
            .setListener(null);
    
    // Animate the loading view to 0% opacity. After the animation ends,
    // set its visibility to GONE as an optimization step (it won't
    // participate in layout passes, etc.)
    mLoadingView.animate()
            .alpha(0f)
            .setDuration(mShortAnimationDuration)
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    mLoadingView.setVisibility(View.GONE);
                }
            });
}

```
（吐槽：只不过想搞个淡入淡出，这么麻烦？）

屏幕滑动是一种在完整屏幕间切换的常见的transition，例如app导航教程(setup wizards)、幻灯片(slideshows)。

创建你之后会用于fragment内容的layout，另外还得为其内容定义一个string：
```
<!-- fragment_screen_slide_page.xml -->
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/content"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <TextView style="?android:textAppearanceMedium"
        android:padding="16dp"
        android:lineSpacingMultiplier="1.2"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/lorem_ipsum" />
</ScrollView>
```
创建一个Fragment类，在其onCreateView()中返回刚刚创建的layout。然后你可以在父activity中随时需要想给用户展示的时候创建fragment的实例。
ViewPager有内置的滑动手势来对应pages的转换，默认会显示屏幕滑动动画，所以你不需要创建动画。ViewPager用PagerAdapter来为新pages的显示提供支持(supply)，所以PagerAdapter会用到你之前创建的fragment类。

要开始的话，创建一个包含ViewPager的layout：
```
<!-- activity_screen_slide.xml -->
<android.support.v4.view.ViewPager
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```
创建满足下面条件的activity：
- 设置content view为带有pager的布局。
- 创建继承`FragmentStatePagerAdapter`的类，实现`getItem()`方法来提供屏幕滑动Fragment的实例作为新的页面。这个adapter还要求实现getCount()，which返回了adapter会创建的pages的数量。
- 将PagerAdapter与ViewPager连接(Hook up)。
- 通过在fragments的虚拟栈(stack)中向后移动(move backwards)来处理设备的返回键 。如果用户在第一个page，在activity的回退栈中返回。
```
import android.support.v4.app.Fragment;
import android.support.v4.app.FragmentManager;
...
public class ScreenSlidePagerActivity extends FragmentActivity {
    /**
     * The number of pages (wizard steps) to show in this demo.
     */
    private static final int NUM_PAGES = 5;

    /**
     * The pager widget, which handles animation and allows swiping horizontally to access previous
     * and next wizard steps.
     */
    private ViewPager mPager;

    /**
     * The pager adapter, which provides the pages to the view pager widget.
     */
    private PagerAdapter mPagerAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_screen_slide);

        // Instantiate a ViewPager and a PagerAdapter.
        mPager = (ViewPager) findViewById(R.id.pager);
        mPagerAdapter = new ScreenSlidePagerAdapter(getSupportFragmentManager());
        mPager.setAdapter(mPagerAdapter);
    }
    @Override
    public void onBackPressed() {
        if (mPager.getCurrentItem() == 0) {
            // If the user is currently looking at the first step, allow the system to handle the
            // Back button. This calls finish() on this activity and pops the back stack.
            super.onBackPressed();
        } else {
            // Otherwise, select the previous step.
            //**这里Google自打脸了，我记得AndroidDesign中明确说过
            //Pager中不要记录page的页面，点击返回不要回到历史page**
            mPager.setCurrentItem(mPager.getCurrentItem() - 1);
        }
    }
    /**
     * A simple pager adapter that represents 5 ScreenSlidePageFragment objects, in
     * sequence.
     */
    private class ScreenSlidePagerAdapter extends FragmentStatePagerAdapter {
        public ScreenSlidePagerAdapter(FragmentManager fm) {
            super(fm);
        }

        @Override
        public Fragment getItem(int position) {
            return new ScreenSlidePageFragment();
        }

        @Override
        public int getCount() {
            return NUM_PAGES;
        }
    }
}
```

为了显示自定义的滑动动画，可以实现`ViewPager.PageTransformer`接口并提供(supply)给ViewPager。接口只暴露了一个方法`transformPage()`。屏幕transition的每一点(at each point in...)，方法对于每个可见的(通常只有一个)page和不在屏幕的相邻page调用一次。比如page3可见，然后用户滑动到page4，那么transformPage()被调用给page2、3、4在手势的每个阶段。
在你对transformPage()的实现中，你搞清楚哪些pages需要基于屏幕上page的position来进行transform变化，然后创建自定义animation。这个参数可以从transformPage方法中的position参数获得。它代表任意page相对于屏幕中间的位置。随着用户在pages之间滚动，这个值也随之动态变化。当一个page填满了屏幕，他的position值就是0，当其从右侧消失，其position就是1。如果用户在1、2页面间滑动到半路(halfway)，page1就有-0.5的position，page2就是0.5。（也就是说往左滑动position就变成负值，往右就是正值。滑动完毕也就是page不再显示就会变成绝对值为1）基于page在屏幕上的position，你可以通过设置page的属性方法比如setAlpha(), setTranslationX(), setScaleY()。来创建自定义animation。
当你拥有了PageTransformer的实现，调用`setPageTransformer`来应用自定义动画。例如:
```
ViewPager mPager = (ViewPager) findViewById(R.id.pager);
...
mPager.setPageTransformer(true, new ZoomOutPageTransformer());
```
页面放大转换(Zoom-out page transformer)会在滚动的时候缩小并且淡化pages。当page接近中心时，它会变到正常尺寸并淡入。可以直接用：
```
public class ZoomOutPageTransformer implements ViewPager.PageTransformer {
    private static final float MIN_SCALE = 0.85f;
    private static final float MIN_ALPHA = 0.5f;

    public void transformPage(View view, float position) {
        int pageWidth = view.getWidth();
        int pageHeight = view.getHeight();

        if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);

        } else if (position <= 1) { // [-1,1]
            // Modify the default slide transition to shrink the page as well
            float scaleFactor = Math.max(MIN_SCALE, 1 - Math.abs(position));
            float vertMargin = pageHeight * (1 - scaleFactor) / 2;
            float horzMargin = pageWidth * (1 - scaleFactor) / 2;
            if (position < 0) {
                view.setTranslationX(horzMargin - vertMargin / 2);
            } else {
                view.setTranslationX(-horzMargin + vertMargin / 2);
            }

            // Scale the page down (between MIN_SCALE and 1)
            view.setScaleX(scaleFactor);
            view.setScaleY(scaleFactor);

            // Fade the page relative to its size.
            view.setAlpha(MIN_ALPHA +
                    (scaleFactor - MIN_SCALE) /
                    (1 - MIN_SCALE) * (1 - MIN_ALPHA));

        } else { // (1,+Infinity]
            // This page is way off-screen to the right.
            view.setAlpha(0);
        }
    }
}
```
层次页面转换动画(depthPageTransformer)代码，可以直接用：
```
public class DepthPageTransformer implements ViewPager.PageTransformer {
    private static final float MIN_SCALE = 0.75f;

    public void transformPage(View view, float position) {
        int pageWidth = view.getWidth();

        if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);

        } else if (position <= 0) { // [-1,0]
            // Use the default slide transition when moving to the left page
            view.setAlpha(1);
            view.setTranslationX(0);
            view.setScaleX(1);
            view.setScaleY(1);

        } else if (position <= 1) { // (0,1]
            // Fade the page out.
            view.setAlpha(1 - position);

            // Counteract the default slide transition
            view.setTranslationX(pageWidth * -position);

            // Scale the page down (between MIN_SCALE and 1)
            float scaleFactor = MIN_SCALE
                    + (1 - MIN_SCALE) * (1 - Math.abs(position));
            view.setScaleX(scaleFactor);
            view.setScaleY(scaleFactor);

        } else { // (1,+Infinity]
            // This page is way off-screen to the right.
            view.setAlpha(0);
        }
    }
}
```

卡片翻转动画：
为卡片翻转创建动画。你需要俩动画，一个用于前面的卡片从左边动画出去，一个是前面的卡片从左边动画进来。同样滴，对于背面的卡片也需要一个从右边进来的动画，一个从右边出去的动画。
card_flip_left_in.xml
```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Before rotating, immediately set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:duration="0" />

    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="-180"
        android:valueTo="0"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 1. -->
    <objectAnimator
        android:valueFrom="0.0"
        android:valueTo="1.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```
card_flip_left_out.xml
```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="0"
        android:valueTo="180"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```
card_flip_right_in.xml
```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Before rotating, immediately set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:duration="0" />

    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="180"
        android:valueTo="0"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 1. -->
    <objectAnimator
        android:valueFrom="0.0"
        android:valueTo="1.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```
card_flip_right_out.xml
```
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Rotate. -->
    <objectAnimator
        android:valueFrom="0"
        android:valueTo="-180"
        android:propertyName="rotationY"
        android:interpolator="@android:interpolator/accelerate_decelerate"
        android:duration="@integer/card_flip_time_full" />

    <!-- Half-way through the rotation (see startOffset), set the alpha to 0. -->
    <objectAnimator
        android:valueFrom="1.0"
        android:valueTo="0.0"
        android:propertyName="alpha"
        android:startOffset="@integer/card_flip_time_half"
        android:duration="1" />
</set>
```
卡片的每面都是可以包含任何内容的layout，比如俩屏幕的文本，图片，或者任何组合的views。你会在你想动画的fragments中用到这俩布局。
对于卡片的两面，创建俩fragment，其中返回了你之前定义的layouts。你可以在父activity中你想展示卡片的时候创建fragment的实例，代码展示了父activity内的嵌套fragment(nested fragment classes)：
```
public class CardFlipActivity extends Activity {
    ...
    /**
     * A fragment representing the front of the card.
     */
    public class CardFrontFragment extends Fragment {
        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                Bundle savedInstanceState) {
            return inflater.inflate(R.layout.fragment_card_front, container, false);
        }
    }

    /**
     * A fragment representing the back of the card.
     */
    public class CardBackFragment extends Fragment {
        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                Bundle savedInstanceState) {
            return inflater.inflate(R.layout.fragment_card_back, container, false);
        }
    }
}
```
现在你需要在父activity中显示fragments。在其布局文件中你可以创建Framelayout作为容器，这样你可以实时添加fragments。然后在activity设置好content view。在activity创建的时候就显示默认的fragment通常是个好主意：
```
public class CardFlipActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_activity_card_flip);

        if (savedInstanceState == null) {
            getFragmentManager()
                    .beginTransaction()
                    .add(R.id.container, new CardFrontFragment())
                    .commit();
        }
    }
    ...
}
```
现在你有了卡片前面，你可以在合适的时候用翻转动画来展示背面。创建展示卡片另一面的动画需要做这些事：
- 为fragment转换设置你之前创建好的自定义动画
- 用新的fragment替换现在显示的fragment，并应用自定义动画
- 把之前显示的fragment加入回退栈(back stack)这样用户点击返回卡片就可以翻转回去。
```
private void flipCard() {
    if (mShowingBack) {
        getFragmentManager().popBackStack();
        return;
    }
    // Flip to the back.
    mShowingBack = true;

    // Create and commit a new fragment transaction that adds the fragment for
    // the back of the card, uses custom animations, and is part of the fragment
    // manager's back stack.

    getFragmentManager()
            .beginTransaction()

            // Replace the default fragment animations with animator resources
            // representing rotations when switching to the back of the card, as
            // well as animator resources representing rotations when flipping
            // back to the front (e.g. when the system Back button is pressed).
            .setCustomAnimations(
                    R.animator.card_flip_right_in,
                    R.animator.card_flip_right_out,
                    R.animator.card_flip_left_in,
                    R.animator.card_flip_left_out)

            // Replace any fragments currently in the container view with a
            // fragment representing the next page (indicated by the
            // just-incremented currentPage variable).
            .replace(R.id.container, new CardBackFragment())

            // Add this transaction to the back stack, allowing users to press
            // Back to get to the front of the card.
            .addToBackStack(null)
            // Commit the transaction.
            .commit();
}
```

点击放大view的效果：

创建一个包含小的和你想放大的内容的大的版本的layout（好拗口）。下面的代码用ImageButton作为图片的索缩略图和一个显示放大版的Imageview：
```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/container"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">

        <ImageButton
            android:id="@+id/thumb_button_1"
            android:layout_width="100dp"
            android:layout_height="75dp"
            android:layout_marginRight="1dp"
            android:src="@drawable/thumb1"
            android:scaleType="centerCrop"
            android:contentDescription="@string/description_image_1" />

    </LinearLayout>

    <!-- This initially-hidden ImageView will hold the expanded/zoomed version of
         the images above. Without transformations applied, it takes up the entire
         screen. To achieve the "zoom" animation, this view's bounds are animated
         from the bounds of the thumbnail button above, to its final laid-out
         bounds.
         -->

    <ImageView
        android:id="@+id/expanded_image"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:visibility="invisible"
        android:contentDescription="@string/description_zoom_touch_close" />

</FrameLayout>
```
应用了layout后，设置一下触发放大动画的事件处理器(event handlers)：
```
public class ZoomActivity extends FragmentActivity {
    // Hold a reference to the current animator,
    // so that it can be canceled mid-way.
    private Animator mCurrentAnimator;

    // The system "short" animation time duration, in milliseconds. This
    // duration is ideal for subtle animations or animations that occur
    // very frequently.
    private int mShortAnimationDuration;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_zoom);

        // Hook up clicks on the thumbnail views.

        final View thumb1View = findViewById(R.id.thumb_button_1);
        thumb1View.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                zoomImageFromThumb(thumb1View, R.drawable.image1);
            }
        });

        // Retrieve and cache the system's default "short" animation time.
        mShortAnimationDuration = getResources().getInteger(
                android.R.integer.config_shortAnimTime);
    }
    ...
}
```

一般来说你得从正常尺寸的view的边界动画到大版本view的边界。你得做这些事：
1. 分配高分辨率的图片到隐藏的"zoomed-in"(放大的)ImageView中。你需要在分支线程加载图像以防阻塞UI线程，并且在UI线程设置bitmap。理想情况下，bitmap不该比屏幕尺寸大。
2. 计算开始和结束时ImageView的边界。
3. 分别同步地动画4个位置和尺寸属性X, Y, SCALE_X, SCALE_Y，从开始到结束边界。这4个动画加到一个`AnimatorSet`中这样可以同时启动。
4. 当用户放大图片后，点击屏幕时需要运行一个类似的动画以退出放大(zoom back out)。
(译者注：下面代码仅供学习，因为android L后有了自带的transition效果: ActivityOptionsCompat.makeSceneTransitionAnimation)
```
private void zoomImageFromThumb(final View thumbView, int imageResId) {
    // If there's an animation in progress, cancel it
    // immediately and proceed with this one.
    if (mCurrentAnimator != null) {
        mCurrentAnimator.cancel();
    }

    // Load the high-resolution "zoomed-in" image.
    final ImageView expandedImageView = (ImageView) findViewById(
            R.id.expanded_image);
    expandedImageView.setImageResource(imageResId);

    // Calculate the starting and ending bounds for the zoomed-in image.
    // This step involves lots of math. Yay, math.
    final Rect startBounds = new Rect();
    final Rect finalBounds = new Rect();
    final Point globalOffset = new Point();

    // The start bounds are the global visible rectangle of the thumbnail,
    // and the final bounds are the global visible rectangle of the container
    // view. Also set the container view's offset as the origin for the
    // bounds, since that's the origin for the positioning animation
    // properties (X, Y).
    thumbView.getGlobalVisibleRect(startBounds);
    findViewById(R.id.container)
            .getGlobalVisibleRect(finalBounds, globalOffset);
    startBounds.offset(-globalOffset.x, -globalOffset.y);
    finalBounds.offset(-globalOffset.x, -globalOffset.y);

    // Adjust the start bounds to be the same aspect ratio as the final
    // bounds using the "center crop" technique. This prevents undesirable
    // stretching during the animation. Also calculate the start scaling
    // factor (the end scaling factor is always 1.0).
    float startScale;
    if ((float) finalBounds.width() / finalBounds.height()
            > (float) startBounds.width() / startBounds.height()) {
        // Extend start bounds horizontally
        startScale = (float) startBounds.height() / finalBounds.height();
        float startWidth = startScale * finalBounds.width();
        float deltaWidth = (startWidth - startBounds.width()) / 2;
        startBounds.left -= deltaWidth;
        startBounds.right += deltaWidth;
    } else {
        // Extend start bounds vertically
        startScale = (float) startBounds.width() / finalBounds.width();
        float startHeight = startScale * finalBounds.height();
        float deltaHeight = (startHeight - startBounds.height()) / 2;
        startBounds.top -= deltaHeight;
        startBounds.bottom += deltaHeight;
    }

    // Hide the thumbnail and show the zoomed-in view. When the animation
    // begins, it will position the zoomed-in view in the place of the
    // thumbnail.
    thumbView.setAlpha(0f);
    expandedImageView.setVisibility(View.VISIBLE);

    // Set the pivot point for SCALE_X and SCALE_Y transformations
    // to the top-left corner of the zoomed-in view (the default
    // is the center of the view).
    expandedImageView.setPivotX(0f);
    expandedImageView.setPivotY(0f);

    // Construct and run the parallel animation of the four translation and
    // scale properties (X, Y, SCALE_X, and SCALE_Y).
    AnimatorSet set = new AnimatorSet();
    set
            .play(ObjectAnimator.ofFloat(expandedImageView, View.X,
                    startBounds.left, finalBounds.left))
            .with(ObjectAnimator.ofFloat(expandedImageView, View.Y,
                    startBounds.top, finalBounds.top))
            .with(ObjectAnimator.ofFloat(expandedImageView, View.SCALE_X,
            startScale, 1f)).with(ObjectAnimator.ofFloat(expandedImageView,
                    View.SCALE_Y, startScale, 1f));
    set.setDuration(mShortAnimationDuration);
    set.setInterpolator(new DecelerateInterpolator());
    set.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            mCurrentAnimator = null;
        }

        @Override
        public void onAnimationCancel(Animator animation) {
            mCurrentAnimator = null;
        }
    });
    set.start();
    mCurrentAnimator = set;

    // Upon clicking the zoomed-in image, it should zoom back down
    // to the original bounds and show the thumbnail instead of
    // the expanded image.
    final float startScaleFinal = startScale;
    expandedImageView.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View view) {
            if (mCurrentAnimator != null) {
                mCurrentAnimator.cancel();
            }

            // Animate the four positioning/sizing properties in parallel,
            // back to their original values.
            AnimatorSet set = new AnimatorSet();
            set.play(ObjectAnimator
                        .ofFloat(expandedImageView, View.X, startBounds.left))
                        .with(ObjectAnimator
                                .ofFloat(expandedImageView, 
                                        View.Y,startBounds.top))
                        .with(ObjectAnimator
                                .ofFloat(expandedImageView, 
                                        View.SCALE_X, startScaleFinal))
                        .with(ObjectAnimator
                                .ofFloat(expandedImageView, 
                                        View.SCALE_Y, startScaleFinal));
            set.setDuration(mShortAnimationDuration);
            set.setInterpolator(new DecelerateInterpolator());
            set.addListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    thumbView.setAlpha(1f);
                    expandedImageView.setVisibility(View.GONE);
                    mCurrentAnimator = null;
                }

                @Override
                public void onAnimationCancel(Animator animation) {
                    thumbView.setAlpha(1f);
                    expandedImageView.setVisibility(View.GONE);
                    mCurrentAnimator = null;
                }
            });
            set.start();
            mCurrentAnimator = set;
        }
    });
}
```
 
动画布局的变化：
布局动画是一种每次你对布局配置改变时系统运行的预加载动画。你需要做的就是在layout中设置一个属性，以告诉系统来动画这些布局改变，然后系统默认的动画就会自动为你执行(carry out)。如果你需要提供自定义布局动画，创建`LayoutTransition`对象然后提供它一个带有setLayoutTransition方法的布局。

在布局文件中，加入`android:animateLayoutChanges`属性为true即可。现在，你所需要做的就是添加、移除，或者更新布局的item然后item就会自动动画了：
```
private ViewGroup mContainerView;
...
private void addItem() {
    View newView;
    ...
    mContainerView.addView(newView, 0);
}
```

## Connectivity & the Cloud

### Transferring Data Without Draining the Battery
这节课演示了计划和执行下载的最佳实践，比如缓存，polling，和预加载(prefetch)。你可以学到(You'll learn how the power-use profile of the wireless radio can affect your choices on when, what, and how to transfer data in order to minimize impact on battery life)巴拉巴拉。

一个完全活跃的无线电([Wireless radio](http://baike.baidu.com/link?url=e2IzUhL5A4UKxeZGCT_YZBx2t5BsKFuGJk7PmG22nG6sOjjn2T61ogUIzClwCvr4fRcCHHt8URr7ULy32nehrK-tolk3AFj7AUzFczz_wsW))非常消耗电量(consumes significant power)，所以为了省电不用的时候它会在不同的能量状态之间切换，同时在需要的时候为了减少等待时间，与为无线电加电所关联(while attempting to minimize latency associated with "powering up" the radio when it's required)。
这种对于典型的3G网络无线电的状态机器由3种能量状态构成(The state machine for a typical 3G network radio consists of three energy states)：
1. 满电力：连接活跃时使用，允许设备以最高速率传输数据。
2. 低电力：大约消耗满电力一半的电量的中间(intermediate)状态。
3. 待机(standby)：在没有活跃的或者必须的网络连接期间。消耗最小电量。
尽管后俩状态消耗更少的电量，他们也带来了网络请求的明显延迟。从low状态到full电力状态大概要花1.5s，而从待机到full要花费2s以上。为了最小化等待时间，状态机器(state machine)使用延迟来推迟到更低级别的能量状态的转换。下图是AT&T(美国第二大移动运营商)对于典型3G无线电的时间：
![](http://developer.android.com/images/efficient-downloads/mobile_radio_state_machine.png)
Figure 1. Typical 3G wireless radio state machine.

在每个设备上，radio state machine，尤其是关联转换的延迟(尾巴时间)和开启等待时间(sartup latency)，根据其使用的无线电技术（2G, 3G, LTE，etc）而不同，并与运营商和设备的配置有关。
这节课根据AT&T提供的数据，描述了典型3G无线电的state machine。但是，一般的原则和最佳事件的结论适用于所有无线电的实现。这招对于典型的web浏览尤其有效，因为他避免了用户浏览网页时的不爽的延迟。相对低的tail-time保证了一旦浏览结束，无线电就可以移动到较低的能量状态。
不幸的是，这招在现代的智能手机系统会导致app的低效率。因为app既运行在前台（延迟很要命）又运行在后台（电池寿命需要优先考虑）。*怎么解决呢？那就是我们今天的产品，新一代锂电池，小体积，高容量！仅售998！*咳咳，开个玩笑。

每次你创建新的网络连接，无线电都会切换到full power状态。对于之前说的典型3G无线电状态机器，它在你传送数据的持续时间内都保持full power——外加5s的尾巴时间——接下来就是12s的低能量状态。也就是说，每次数据传送都会引起无线电消耗20s的能量。
在实际应用中，这意味着传送未捆绑数据(unbundled)1s，就会保持无限电激活18s。回到Full power状态跟变成闲置(idle)状态差不多。
结果就是，每分钟都会消耗高电量状态18s，剩下42s消耗低电量状态。
相比之下，同样的app，每分钟传送3s数据会让无线电保持高电量状态仅仅8s，并让他保持在低电量状态仅仅是额外的12s。
第二个例子允许无线电每分钟闲置额外的40s，这可以大幅度减少电量消耗。
![](http://developer.android.com/images/efficient-downloads/graphs.png)
Figure 2. Relative wireless radio power use for bundled versus unbundled transfers.

预加载数据是有效减少数据传送任务次数的方法。Prefetching让你在一次连接中全速下载所有你可能在特定的时间段需要的数据.
通过预加载你要交换的数据，你减少了无线电为了下载数据而激活的次数。作为结果，你不仅仅节省了电池电量，还优化了等待时间，降低了需求的带宽。
但是捏，用的过于频繁，预加载也带来了增加消耗电量和带宽使用的风险(下载了不需要的数据)。确保预加载没有造成app启动延迟也很重要。(In practical terms that might mean processing data progressively, or initiating consecutive transfers prioritized such that the data required for application startup is downloaded and processed first.)
你预加载多频繁取决于要下的数据的大小和它被用到的可能性。作为一个粗略的指导，基于上述的state machine，对于有50%可能性使用概率的数据，一般你可以下载大概6s(大概1-2Mb)（you can typically prefetch for around 6 seconds (approximately 1-2 Mb) before the potential cost of downloading unused data matches the potential savings of not downloading that data to begin with.）
通常来说，预加载是个好习惯，这样你只需要初始化另一个下载每2-5分钟，以1-5mb的顺序。（you will only need to initiate another download every 2 to 5 minutes, and in the order of 1 to 5 megabytes.）遵循这个规则，大量下载——比如视频——应该以规律的间隔（每2-5分钟）来大块大块的吃肉哦不，下载。高效地预加载只有可能在接下来几分钟会被播放的视频。注意，进一步的下载应该打包，下个部分会谈到。而且这些近似值会基于连接类型和速度而不同。
来看几个实例：
**音乐播放器：**
你应该选择预加载整个相册，但是可能用户在第一首歌听完就停止了，所那样你就浪费了一大堆带宽和电量。
更好的方案是，维持一首歌的缓存(buffer)和这首正在播放的歌。对于流音乐，比如维持会保持无线电一直活跃的连续的流，不如考虑用HTTP live streaming来传送突然的音频流，模拟上述的预加载方法。
**新闻阅读器**：
许多资讯类app试图在一个栏目被选中后就下载头条数据来减少带宽，完整的文章只当用户想读的时候才加载。这个方法，当用户滚动头条（标题）的时候无线电会为了用户的阅读任务而强制保持活跃。不仅如此，在能量状态间的持续切换回导致切换栏目或者读文章时造成明显延迟。
更好的方案是在启动时就预加载合理数量的数据，从第一组新闻头条（标题）和缩略图开始——确保开始时一个较低的等待时间——然后接着加载剩余的头条和缩略图，以及至少第一批标题列表中每个文章的内容。
另一个选择是预加载每个文章头条、缩略图、文本，可能的话甚至全部的文章图片——一般在后台设定好计划来加载。这个方法冒着消耗大量宽带的电池电量的风险下载了可能用不到的数据，所以要小心使用。还有一个解决方法是仅当wifi下才下载全部内容，可能的话仅当充电时才下载。

批量传送和连接

一个每20s ping一次服务器的app，只要确认app在运行并且对用户可见，就会无限地保持无线电开启状态，导致几乎没有数据传输却消耗大量电量的后果。
要记住，打包你的数据传送和创建待处理的传送队列很重要。其核心理念是在每次传送任务期间传送尽可能多的数据。为了限制你需求的任务数量。
任何时间敏感的数据传送或者按需加载——比如下载全尺寸图片——都应该事先有规律地更新。计划好的更新应该在按需加载的同时来执行，在一定间隔后下一波计划好的更新会被执行。这个方法通过在必须的时间敏感的图片下载时的piggy-backing减轻了执行常规更新的代价。

一般来说，复用已有的网络连接比初始化新的要更有效率。复用使得网络更智能地对拥堵和数据问题作出反应。比起创建多个同步的连接来下载数据，或者连接(chaining)多个连续的(consecutive)GET请求，你应该尽可能打包(bundle)这些请求成单独的一个GET请求。
比如，为每个新闻文章做一个单一的请求比为几个新闻目录做多个请求要高效。无线电为了传送终端给带有客户端服务端超时的确认包(acknowledgement packets)，需要保持活跃。所以不用的时候要关闭连接是个好习惯，而不是等待超时。
话虽如此(That said)，过早关闭连接会让他没法复用，然后为了建立新连接就会导致额外的开支(overhead)。有用的妥协办法是立刻关闭连接，但是但是仍然要在固有的timeout过期之前关闭。

定期更新的最佳频率会基于不同设备状态、网络连接、用户行为和明确的用户偏好而不同。
这节课会审查(examine)你的刷新频率如何能最好地减轻基于无线电状态机器的后台更新的效果。
接下来讲的是[用GCM来轮询](http://developer.android.com/intl/zh-cn/training/efficient-downloads/regular_updates.html#GCM)(polling)，这里不翻了。

多余的下载是多余的：
最基本的减少下载的方法是只下载你需要的东西。就数据而言(In terms of data)，这意味着实现REST APIS允许你指定具体查询条件比如上次你更新的时间，来限制返回的数据。同样地，下载图片的时候，服务端减少图片尺寸也是个好习惯，而不是下载全尺寸的然后在客户端裁剪。
另外一个重要技巧是避免下载重复数据。你可以通过频繁的缓存来实现。永远缓存固定的资源，包括按需下载的比如全尺寸的图片，只要合理的可能性(for as long as reasonably possible)。按需的资源应该单独存储，这样可以让你定期地刷新你按需缓存来管理其尺寸。
为了确保你的缓存不会导致app显示旧的(stale)数据，在请求内容最新更新和过期的时候记得从HTTP响应头(HTTP response headers)提取时间。这可以让你决定相关内容应不应该被刷新。
```
long currentTime = System.currentTimeMillis();

HttpURLConnection conn = (HttpURLConnection) url.openConnection();

long expires = conn.getHeaderFieldDate("Expires", currentTime);
long lastModified = conn.getHeaderFieldDate("Last-Modified", currentTime);

setDataExpirationDate(expires);

if (lastModified < lastUpdateTime) {
  // Skip update
} else {
  // Parse update
}
```
这个方法，你可以高效地缓存动态内容，同时确保不会导致你app显示旧数据。你可以在不受管理的外部缓存文件夹缓存不敏感的数据：`Context.getExternalCacheDir();`或者，你可以用受到管理/安全的应用缓存。这一这个缓存在系统存储空间不足时可能会被清理掉。`Context.getCacheDir()`

用HttpURLConnection缓存（因为不用这玩意了，所以显然，不翻）

基于连接类型来修改下载模式：
当谈到对电量的影响，不是所有连接类型都相等的。不止是WiFi比无线电（译者注：可以认为是流量）耗电更少这么简单，而且无线电的不同技术对于电量也有不同影响(implications)。
大多数情况下，WiFi无线电可以提供更高带宽并消耗更少电量。结论就是，你应该在任何连接到WiFi的时候去努力下载数据。你可以用broadcast receiver来监听WiFi的连接，来执行重要的下载，计划好的更新，甚至临时增加更新频率。
用更高的带宽来以更少的频率下载更多的数据（好拗口）
连接到流量时，更高的带宽通常带来的是更高电量消耗。这意味着LTE一般比3G消耗更多能量（LTE>3G>2G）
与此同时，高带宽以为你可以更频繁加载数据，同样的时间能下载更多数据。可能有点反直觉(Perhaps less intuitively)，因为电池消耗的尾巴时间相对更高，在每次数据传输任务中，保持无线电活跃更长时间会更高效。
比如，一个LTE电磁波是3G的带宽和能量消耗的2倍，你应该在每次任务中下载4倍的数据——或者大约10mb。当下载这么多数据时，考虑到你预加载在可用的本地内存和周期性地flush你的预加载缓存很重要。
你可以用ConnectivityManager来决定激活的无线电，并修改你预加载的流程。
```
ConnectivityManager cm =
 (ConnectivityManager)getSystemService(Context.CONNECTIVITY_SERVICE);

TelephonyManager tm =
  (TelephonyManager)getSystemService(Context.TELEPHONY_SERVICE);
  
NetworkInfo activeNetwork = cm.getActiveNetworkInfo();
 
int PrefetchCacheSize = DEFAULT_PREFETCH_CACHE;
 
switch (activeNetwork.getType()) {
  case (ConnectivityManager.TYPE_WIFI): 
    PrefetchCacheSize = MAX_PREFETCH_CACHE; break;
  case (ConnectivityManager.TYPE_MOBILE): {
    switch (tm.getNetworkType()) {
      case (TelephonyManager.NETWORK_TYPE_LTE | 
            TelephonyManager.NETWORK_TYPE_HSPAP): 
        PrefetchCacheSize *= 4;
        break;
      case (TelephonyManager.NETWORK_TYPE_EDGE | 
            TelephonyManager.NETWORK_TYPE_GPRS): 
        PrefetchCacheSize /= 2;
        break;
      default: break;
    }
    break;
  }
  default: break;
}
```

---

以上是Building apps部分，还有重要的Best practice部分我会在另一篇文章中写完。
