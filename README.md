# TrainingNote
***
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

***

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

***

###Building a Dynamic UI with Fragments
>To create a dynamic and multi-pane user interface on Android, you need to encapsulate UI components and activity behaviors into modules that you can swap into and out of your activities. You can create these modules with the Fragment class, which behaves somewhat like a nested activity that can define its own layout and manage its own lifecycle.
为了创建动态的、多面板的UI，你需要封装UI控件和activity行为成模块，这样可以包进你的activity(swap into and out of your activities)。
你可以用Fragment类来创建这些模块，它就像一个嵌套的activity一样，可以有自己的布局和生命周期。你可以把Fragment看做模块化的activity。有自己的布局、生命周期、输入事件，还能在activity运行时动态添加、移除。（有点像你可以在不同activity中复用的子activity）

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
注意，当你在activity的布局文件中定义fragment时，是无法运行时移除的。
为了能动态添加或移除Fragment，你得用FragmentManager来创建FragmentTransaction。动态处理Fragment时有一点很重要————你的activity布局必须包含一个container View作为你插入fragment的容器。
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
>Keep in mind that when you perform fragment transactions, such as replace or remove one, it's often appropriate to allow the user to navigate backward and "undo" the change. 
请记住，当你执行fragment事务比如replace或者remove时，允许用户能回到之前的状态并"撤销"改变通常是合适的。当你执行这些操作，并`addToBackStack()`时，被remove的fragment实际上是处于stopped状态而非destroyed。如果用户返回并恢复之前的fragment，它会重新启动(restarts)。如果你不加到back stack，那么被remove或replace的fragment就会被销毁。
`addToBackStack()`有一个可选的String参数，给这个transaction指定了独一无二的名字。这个名字没什么卵用，除非你打算用fragment操作中的FragmentManager.BackStackEntry里的API。

所有的Fragment之间的通讯都应该通过相连接的activity来完成，俩fragment绝不应该直接交流。Fragment和activity通讯示例：
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

宿主activity（host activity）可以通过`findFragmentById()`来找到fragment实例，然后直接调用其public方法，以实现发消息给fragment。
例如，上面举得例子中activity可能包含另外一个fragment，通过回调中返回的数据来让该fragment显示相应的item：
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
```

你可以用下面的方法，创建一个shared preference文件或者获得已经存在的sp：
- `getSharedPreferences()` 通过指定文件名，来获得相应的sp。你可以在app中的任何Context下使用该方法
- `getPreferences()` 在Activity中调用该方法，可以获得此activity专属的sp。不需要指定文件名。
- `PreferenceManager.getDefaultSharedPreferences()` 包级别的sp。这个实际上返回的就是以`context.getPackageName() + "_preferences"`作为文件名的sp
目前，所有app不需要指定特殊权限就可以读取外部存储。但是这一点将来会改变。所以为了确保你的app能像预期一样工作，你需要在manifest中添加
` <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />`
但是捏，如果你的app使用了WRITE_EXTERNAL_STORAGE权限，那么就默认也具有读取权限了。

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
或者，你只需要缓存一些文件，可以用`createTempFile()`，下面的示例演示了从url中提取文件名，并用它在app的内部cache文件夹中创建文件：
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
你的app的内部存储文件夹放在在android文件系统中的一个特殊位置，它是由你的包名决定的。严格来说，只要你把文件模式设为可读，其他app就可以读取你app的内部文件。但是，他还得知道你app的包名、文件名。所以只要你把内部存储文件设为`MODE_PRIVATE`，其他app就绝对没法访问。

在访问外部存储之前，你应该总是先判断是否可用（因为可能用户连接存储到PC，或者移除SD卡）：
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
>Public files
Files that should be freely available to other apps and to the user. When the user uninstalls your app, these files should remain available to the user.
For example, photos captured by your app or other downloaded files.
公共文件：
可以让用户和其他app自由访问的文件。应用被卸载时，这些文件应该保留。例如由你app拍摄或者下载的文件。

>Private files
Files that rightfully belong to your app and should be deleted when the user uninstalls your app. Although these files are technically accessible by the user and other apps because they are on the external storage, they are files that realistically don't provide value to the user outside your app. When the user uninstalls your app, the system deletes all files in your app's external private directory.
For example, additional resources downloaded by your app or temporary media files.

私有文件：
你app合法拥有的，卸载时应当被删除的文件。尽管技术上来说，用户和其他app可以访问这些文件，但是他们在脱离你app之后就没什么意义（价值）。当用户卸载app时，系统会删掉你外部私人目录中的文件。例如，你app下载的扩充资源或者缓存的媒体文件。

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
如果要保存私有文件，用`getExternalFilesDir()`（这个方法创建的每个目录都会被加到一个包含了了你所有外部存储文件的父目录）：
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
如果预定义的子文件夹名称不能满♂足♀你，你可以调用`getExternalFilesDir()`并传入null。这样会返回你app在外部存储的私有文件夹的根目录（以下简称为根目录）。记住，`getExternalFilesDir()`会在根目录下创建一个目录，而这个根目录会在app卸载的时候也被删除（因为它是私有文件夹呀）。
>Regardless of whether you use getExternalStoragePublicDirectory() for files that are shared or getExternalFilesDir() for files that are private to your app, it's important that you use directory names provided by API constants like DIRECTORY_PICTURES. These directory names ensure that the files are treated properly by the system. For instance, files saved in DIRECTORY_RINGTONES are categorized by the system media scanner as ringtones instead of music.
无论你用getExternalStoragePublicDirectory()还是getExternalFilesDir()保存文件，有一件事很重要：使用系统提供的文件夹名的API比如DIRECTORY_PICUTRES。这些文件夹名可以确保系统能正确处理里面的文件。比如保存在`DIRECTORY_RINGTONES`里的文件会被系统媒体扫描器分类到铃声而不是音乐。

如果你知道你要保存的数据有多大，你可以直接使用`getFreeSpace()`或`getTotalSpace()`查询可用空间而不需要抛出`IOException`。这俩方法分别提供：现在可用的空间；存储器的总空间。但是捏，系统并不保证你获得的可用空间就是你能写的空间（字节）。如果返回的空间比你想写入的多几mb，或者文件系统占用了90%以下。否则你可能不该写入。
并不一定非得在保存文件之前查询可用空间，你可以直接写入，然后catch an `IOException`。比如你想把PNG图片转成JPEG，你就不知道文件有多大。

你可以直接调用文件的delete方法来删除。如果文件保存在内部存储，可以用`mContext.deleteFile(fileName);`来定位并删除文件。当app被卸载时，系统会删掉：内部存储的所有文件；外部存储中，使用getExternalFilesDir()创建的文件。但是呢，你应该定期清理所有用`getCacheDir()`创建的缓存文件，定期删除其他你不需要的文件。

***

###Interacting with Other Apps
通常你会用explicit intent(显示的、明确的intent)来在activity间切换，它定义了你想启动的组件的类名。但是，如果你
需要进行独立于app之外的操作，比如查看地图，你得用implicit intent(隐式intent)。它不指定类名，而是声明了一个action。
Intent通常会与data联系在一起，比如你要查看的地址，或者你想发送的邮件信息，取决于你要创建的intent，data数据可能是Uri，
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
>Other kinds of implicit intents require "extra" data that provide different data types, such as a string. You can add one or more pieces of extra data using the various putExtra() methods.
By default, the system determines the appropriate MIME type required by an intent based on the Uri data that's included. If you don't include a Uri in the intent, you should usually use setType() to specify the type of data associated with the intent. Setting the MIME type further specifies which kinds of activities should receive the intent.

其他类型的隐式intent需要"extra" data来提供不同的数据类型，比如String。你可以使用putExtra()添加一或多个extra数据。默认情况下系统会基于
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

尽管android平台保证每个特定的intent都可以被内置的app来解析，在调用intent之前，你还是应该验证一下。因为如果你调用intent没有app能打开的话，你的app就会直接崩掉：
```
PackageManager packageManager = getPackageManager();
List activities = packageManager.queryIntentActivities(intent,
        PackageManager.MATCH_DEFAULT_ONLY);
boolean isIntentSafe = activities.size() > 0;
```
>You should perform this check when your activity first starts in case you need to disable the feature that uses the intent before the user attempts to use it. If you know of a specific app that can handle the intent, you can also provide a link for the user to download the app

你应该在activity刚启动的时候就检查intent，这样你可以在用户使用它（点击它）之前就禁用掉这个功能。如果你知道有特定app可以处理这个intent，你可以给用户提供一个下载链接。

创建好intent，设置好extra后，调用startActivity()给系统。系统如果识别到有多个activity可以处理，就会显示一个对话框让用户选择，如果只有一个，系统就会立刻打开。默认情况对话框底部都会有一个checkbox可以让用户选择默认打开的应用。这很方便，比如用户倾向于使用同一个浏览器或是相机app。但是捏，如果用户可能每次都想选不同的app比如分享操作，你得明确地创建一个chooser dialog：
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
启动Activity不必是单向的，你还可以启动其他activity并接受一个结果。比如你可以打开联系人让用户选择一个联系方式然后你接受这个联系人的详情作为一个result。当然，前提是该activity必须设计为可以返回一个result。这样你可以在`onActivityResult()`来接受回调。你用`startActivityForResult()`既可以用显式也可用隐式intent。如果启动的是你自己的activity，你应该用显示intent来确保接受的result是正确的。
startActivityForResult和普通的intent没什么区别，唯一不同就是你得传一个额外的int值作为请求码(request code)。当你收到result Intent时，回调会提供同一个request code这样你可以验证结果并决定如何处理。例如选择一个联系人：
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
另外，android 2.3之后，联系人app可以授予你的app一个临时权限来从Contacts Provider读取结果。但是只允许你读取特定的需求的联系人，所以你没法去查询一个联系人，除非你声明READ_CONTACTS权限。

如果你的app能处理可能对其他app有用的操作，你应该准备好从其他app接受action请求。比如如果你做了一个可以分享信息给朋友的app，你最好能支持ACTION_SEND intent。为了让其他app能启动你的activity，你需要在manifest里面添加相应activity的<intent-filter>元素。
当你的app被安装后，系统会识别你的intent filter并把信息添加到一个被所有app支持的intents的内部目录。当app用**隐式intent**调用startActivity或者xxxForResult，系统会找到能响应它的
activity。只有一个activity的intent filter满足下面的标准，系统才可能会把intent发给它：
- <action> 描述需要执行的动作的字符串。通常是平台定义好的比如ACTION_SEND, ACTION_VIEW。如果不是平台的常量是你自己定义的，必须使用全名。
- <data> 使用一个或者多个属性来指定与intent相关联的数据类型。你可以仅指定MIME type，URI前缀，URI scheme，或者它们的组合。如果你不需要指定data的Uri，比如你的activity使用了extra data而不是URI，你应该指定`android:mimeType`属性，如`text/plain`, `image/jpeg`
- <category> 提供额外的方式来个性化处理intent的activity。通常使用默认的CATEGORY_DEFAULT。如果没添加这个category，那你的activity就不能处理隐式的intent。
你可以在activity任何生命周期调用`getIntent()`来解析启动这个activity的intent。但是通常会在onCreate或者onStart里面：
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
你必须指定一个result code给这个result。然后你可以用Intent提供额外的数据。这个结果默认是RESULT_CANCELED，所以如果用户在操作完成之前点了返回，原activity就能收到一个取消的result。如果你只是需要返回一个代表几种可能的结果之一的数值，你可以直接把result code设为大于0的任意值。如果你不需要包含intent，你可以只传一个result code: `setResult(RESULT_COLOR_RED);
finish();` 这对于返回结果到你自己的app时很好用。因为这个result可以指向决定返回值的常量。不需要检查你的activity是startActivity还是ForResult调用的。只要写上setResult防止启动你的activity需要一个结果。如果不需要，结果会被忽略。

***

##Working with System Permissions

>To protect the system's integrity and the user's privacy, Android runs each app in a limited access sandbox. If the app wants to use resources or information of ites sandbox, the app has to explicitly request permission.
为了保护系统完整性和用户隐私，android在一个有限权限的沙箱里运行每个app。如果这个app像用沙箱之外的资源、信息，就需要明确地申请权限。

取决于权限有多敏感，系统可能会自动授予权限，或者让用户来选择授权与否。比如如果你的app要求打开设备的闪光灯，系统就自动授授权；但是如果你需要读取联系人，系统就会询问用户是否允许。取决于平台的版本，如果是Android5.1和之前的版本，就会在安装的时候让用户授权；如果是6.0或者更高版本，就会在运行时询问用户。通常来说，只要app想用到这个app本身不创造的资源、信息，或者进行影响到设备或者其他app的操作，都需要权限。例如访问网络、使用相机、开关wifi，都需要相应权限。你的app只需要它亲自操作的权限，并不需要其他app来执行或者提供信息的权限。比如如果你的app需要从联系人app中读取信息，就不需要READ_CONTACTS，而那个联系人app才需要。（但是如果你app直接读取用户的联系人时就需要）
申请权限时，需要在manifest的顶级（top-level）元素下加入例如`<uses-permission android:name="android.permission.SEND_SMS"/>`，这样就可以有发信息的权限。

从android6.0开始，用户可以随时撤回app的权限，哪怕app适配的是低版本的API。因此你应该测试你的app在缺失需要的权限时，能否正常运行。
如果你的app需要一个风险性权限，你必须在执行相关操作时每次都检查有没有这个权限。用户可能昨天用了相机，但并不能假设今天还有那个权限。
例如检查写入日历的权限：`int permissionCheck = ContextCompat.checkSelfPermission(thisActivity,Manifest.permission.WRITE_CALENDAR);`如果app拥有这权限，就会返回`PackageManager.PERMISSION_GRANTED`；如果返回PERMISSION_DENIED，app就得向用户申请权限。如果你的app在manifest列出了风险性权限，就必须询问用户来授权。android提供了一些方法来申请权限，这些方法会显示标准的dialog，而且你没法去自定义。
有时候可能想帮助用户理解你为啥需要某权限。比如用户打开一个相册软件，就不会对它需要相使用相机感到奇怪。但是用户可能想知道为什么它需要获得用户位置或联系人。在你申请权限之前，你应该考虑需不需要给用户一个解释。请记住，你并不想用铺天盖地的解释来压垮用户；如果你给太多解释，用户可能很烦，然后直接卸载掉了。
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
这个对话框描述了你需要访问的权限组，而非具体权限，对于每个permission group，用户每次只需要授权一次。如果你app请求同一个group的其他权限，系统会自动授权（调用你的onRequestPermissionsResult()回调，并传入
PERMISSION_GRANTED）即使用户已经授予同一个group的其他权限，你的app还是得每次需要的时候请求权限。此外，将来group可能会发生变化，你并不能依赖于同组的其他权限。当你在onRequestPermissionsResult中收到PERMISSION_DENIED时，并不能假设用户采取了什么操作（可能勾选了不在询问）。

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
你可以用`*/*`的MIME type，但是这样的话只有能处理泛型数据(generic data streams)的activity才会被匹配。

接受的app需要对Uri指向的数据的访问权限。推荐做法：
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

从4.0开始，实现一个高效的、用户友好的分享操作变得更加简单。使用ActionProvide，只要依附到actionBar的一个菜单项（menu item），就可以处理外观和点击的行为。至于ShareActionProvider，你只要提供一个share intent它就会帮你处理好。要用SAP的话，只要在菜单项里面加一个`actionProviderClass`属性就行了：
> 
    <menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
            android:id="@+id/menu_item_share"
            android:showAsAction="ifRoom"
            android:title="Share"
            android:actionProviderClass=
                "android.widget.ShareActionProvider" />
    ...
</menu>

