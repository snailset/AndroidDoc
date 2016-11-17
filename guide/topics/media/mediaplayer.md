# 媒体播放
Android多媒体框架支持多种一般媒体类型，以致于你能轻松整合音频、视频和图像到应用中。你可以通过使用[MediaPlayer]()APIs来播放来自应用资源(raw资源)媒体文件、文件系统中的单独文件、或者网络中的数据流中的的音频或者视频

这个文档展示了如何写一个交互性好、系统性能高以及用户体验好的媒体播放应用。

> 注意：你只能通过标准输出设备播放音频文件。现在，只能是移动设备扬声器或者蓝牙耳机。你不能在电话期间在会话音频播放声音文件。

## 基础
在Android框架中，用下面的类来播放声音或者视频

[MediaPlayer]()
- 这个类是播放声音和视频的主要API。

[AudioManager]()
- 这个类管理音频资源和音频输出设备。

## Manifest声明
在使用MediaPlayer开发应用之前，确保manifest文件中有允许使用相关特性的适当声明。

- **网络许可** ——如果你要使用MediaPlayer来引入基于网络的内容，你的应用必须获取网络使用权。
``` xml
<uses-permission android:name="android.permission.INTERNET" />
```
-  **唤醒锁许可**——如果你需要播放器保持屏幕常亮或者保持处理器不休眠，或者使用[MediaPlayer.setScreenOnWhilePlaying()]()或者[MediaPlayer.setWakeMode()]()方法，你必须获取这个许可。
``` xml
<uses-permission android:name="android.permission.WAKE_LOCK" />
```

## 使用MediaPlayer
媒体框架最重要的组件是[MediaPlayer]()类。这个类的对象仅需小小的设置就能取得、解码和播放音频和视频。它支持几种不同的媒体资源，例如：
- 本地资源
- 内部URL，比如通过Content Resolver获得的
- 外部uRL(流媒体)

关于Android支持的媒体格式列表，请见[Android支持的媒体格式]()文档。

这里是一个如何播放本地raw资源(保存在应用的`res/raw/`文件夹)的例子.
``` java
MediaPlayer mediaPlayer = MediaPlayer.create(context, R.raw.sound_file_1);
mediaPlayer.start(); // 不需要调用 prepare(); create()已经做了
```
这个情况下，`raw`资源是一个系统不用通过任何特殊方式解析的文件。然而，这个资源的内容不应该是没有加工的音频。它应该经过适当地编码和格式化成一种Android支持的格式。

这里是如何通过URI播放一个在系统里的本地资源(例如，是通过Content Resolver获取的)。

``` java
Uri myUri = ....; // 在这里初始化Uri
MediaPlayer mediaPlayer = new MediaPlayer();
mediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
mediaPlayer.setDataSource(getApplicationContext(), myUri);
mediaPlayer.prepare();
mediaPlayer.start();
```

像这样播放远程URL的HTTP流媒体：
``` java
String url = "http://........"; // 这里是你的URL
MediaPlayer mediaPlayer = new MediaPlayer();
mediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
mediaPlayer.setDataSource(url);
mediaPlayer.prepare(); // 也许要花很久(为了缓冲等)
mediaPlayer.start();
```

> 注意：如果你要通过URL来引进在线媒体文件，这个文件必须能够渐进式下载。

> 警告：当使用[setDataSource()]()时，必须要捕捉或通过[IllegalArgumentException]()和[IOException]()，因为你引用的文件可能不存在。

### 异步准备

[MediaPlayer]()的使用原则上是简单。然而，正确地整合它到典型Android应用中，需要思考几个事情，这很重要。例如，[prepare]()的执行需要花费很长时间，因为它涉及到获取和解码媒体数据。因此，在这种情况下，任何方法都可能花很长时间执行，你不应该在应用UI线程调用它。这样做会让UI挂起直到方法返回，这是一个很差的用户体检，并且会导致ANR错误(应用无响应)。即使你认为你的资源会加载很快，但是记住在UI任何事情超过十分之一秒响应会导致明显的暂停，这将会使用户留下你的应用很慢的印象。

为了避免UI线程挂起、大量产生其他线程去准备[MediaPlayer]()，并且能在结束时通知主线程。你可以自己编写线程逻辑，但是，系统还提供了一种便捷的方法实现它，就是通过使用[prepareAsync()]()方法， 这是一种常见的模式。这个方法在后台准备媒体且能立即返回。当媒体准备完毕，[MediaPlayer.OnPreparedListener]()中的[onPrepared]()方法会被调用，可以通过[setOnPreparedListener()]()来配置。

### 管理状态
关于[MediaPlayer]()你需要注意的另一方面是它是基于状态的。就是说，在编写代码时，你需要知道[MediaPlayer]()的内部状态，因为某些状态只有播放器在特定的状态才有效。如果你在错误的状态下操作，系统也许会抛异常，或者会出现其他不可预知的行为。

在[MediaPlayer]()类的文档中展示了完整的状态图，说明了那个方法会将[MediaPlayer]()从一种状态转换到另一种。例如，当你创建一个新的[MediaPlayer]()，它是在`ldle`状态。在那个时候，你应该通过调用[setDataSource()]()初始化，使它到达`initialized`状态。之后，你必须通过使用[prepare()]()或者[prepareAsync()]()方法准备它。当[MediaPlayer]()准备完毕，它会进入`Prepared`状态，这就意味着你可以调用[start()]()播放媒体.那时，正如图上所陈述的那样，你可以通过调用[start()]()、[pause()]()和[seekTo()]()这样的方法使其在`Started`、`Paused`和`PlaybackCompleted`状态中转换。另外，当你调用了[stop()]()，直到你重新准备[MediaPlayer]()，你不能再次调用`start()`方法。

在编写与[MediaPlayer]()交互的代码记住多想想[状态图](https://developer.android.com/images/mediaplayer_state_diagram.gif)，因为在错误的状态下调用方法是bug的常见原因。

### 释放MediaPlayer
[MediaPlayer]()会消耗宝贵的系统资源。因此，你要时常防范，确保[MediaPlayer]()实例在不需要时不被挂起。当你使用它时，应该调用[release()]()确保分配给它的系统资源得到合适地释放。例如，你在使用[MediaPlayer]()时，activity收到调用[onStop()]()，你需要释放[MediaPlayer]()，因为当你的activity没有与用户交互时，持有系统资源是讲不通的（除非你在后台播放媒体，这会在下一个部分讨论）。当然，在你的activity恢复或者重启，你需要创建一个新的[MediaPlayer]()，并且在恢复播放之前要再次准备它。

这里是如果释放并且使[MediaPlayer]()置为空：
``` java
mediaPlayer.release();
mediaPlayer = null;
```
举个例子，想想这个问题，假设当你的activity停止了，你忘了释放[MediaPlayer]()，当时在activity重新启动的时候又创建了一个。正如你知道的，当用户改变屏幕方向（或者通过其他的方式改变设备配置），系统是通过重启activity的方式（默认方式）处理的，这样的话，当用户来来回回旋转设备，切换横屏和竖屏，也许会很快消耗完所有系统资源，因为每一次屏幕方向的，你创建了一个新的[MediaPlayer]()，但是从来不会释放。（关于更多运行时重启的信息，请参见[运行改变处理]()。）

你也许想知道如果当用户离开activity时，继续播放“后台媒体”会发生什么，就像音乐应用的行为一样。在这种情况下，需要用[Service]()来控制[MediaPlayer]()，正如在[使用Service进行MediaPlayer]()。

## 使用Service进行MediaPlayer
如果你要在后台播放媒体应用，甚至是应用不在屏幕上——就是说，当用户正与其他应用交互时，你还想继续播放，你必须开启一个[Service]()并且通过它控制[MediaPlayer]()实例。你应该小心这样的计划，因为，用户和系统希望知道一个运行在后台服务的应用如何与系统的其他部分交互。如果你的应用没有满足他们希望的，用户也许会有个糟糕的体验。这部分描述了你应该知道的主要问题，并且提供了几个如何达成它们的建议。

### 异步运行
首先，就像[Activity]()，所有在[Service]()的工作实际上是默认运行在一个线程，如果你在相同的应用中运行一个activity和一个service，它们默认使用相同的线程（主线程）。因此，service需要很快处理完进来的意图，并且不要在响应它们的时候进行大量的运算。如果需要进行任何大量工作或者阻塞调用，必须异步完成这些任务：使用你自己实现的另一个线程，或者使用异步处理框架。

例如，当你在主线程中使用[MediaPlayer]()，你应该调用[prepareAsync]()而不要调用[prepare]()，当准备完成，你可以开始播放，会通知[MediaPlayer.OnPreparedListener]()，你需要实现它。例如：
``` java
public class MyService extends Service implements MediaPlayer.OnPreparedListener {
    private static final String ACTION_PLAY = "com.example.action.PLAY";
    MediaPlayer mMediaPlayer = null;

    public int onStartCommand(Intent intent, int flags, int startId) {
        ...
        if (intent.getAction().equals(ACTION_PLAY)) {
            mMediaPlayer = ... //在这里初始化
            mMediaPlayer.setOnPreparedListener(this);
            mMediaPlayer.prepareAsync(); // 异步准备，不要阻塞主线程
        }
    }

    /** 当MediaPlayer就绪后调用 */
    public void onPrepared(MediaPlayer player) {
        player.start();
    }
}
```
### 处理异步错误
在一个同步操作中，错误通常会表现为一个异常或者一个错误的代码，当时当你使用异步资源时，你应该确保你的应用能得到合适地错误通知。在使用[MediaPlayer]()的情况下，你可以通过实现这个通过实现[MediaPlayer.OnErrorListener]()实现这个，设置你的[MediaPlayer]()实例：
``` java
public class MyService extends Service implements MediaPlayer.OnErrorListener {
    MediaPlayer mMediaPlayer;

    public void initMediaPlayer() {
        // ...在这里初始化MediaPlayer...

        mMediaPlayer.setOnErrorListener(this);
    }

    @Override
    public boolean onError(MediaPlayer mp, int what, int extra) {
        // ... 适当地应对 ...
        // MediaPlayer已经到达错误状态，必须重新设置！
    }
}
```

记住当错误发生时，[MediaPlayer]()会达到错误状态（请参见[MediaPlayer]()文档中的完整状态图），这很重要，在你使用它之前你需要重置它。

### 使用唤醒锁
当设计后台媒体播放应用时，service运行时设备也许会休眠。因为当设备休眠时，Android系统极力保护电池，系统会关闭任何手机不需要的功能，包括CPU和WIFI硬件。然而，如果你的service正在播放或者缓冲流媒体音乐，你想防止系统干涉播放。

为了确保你的service能在这些条件下继续运行，你必须使用“唤醒锁”。唤醒锁是一种向系统表示即使在手机空闲的情况下，应用也可以获取到使用的功能的方式。

> 注意：你应该保守的使用唤醒锁，只在真正需要时才使用它，因为它会明显减少设备电池的寿命。

为了确保当你的[MediaPlayer]()正在播放时，CPU能持续运行，在初始化[MediaPlayer]()的时候，调用[setWakeMode]()方法。一旦你这样做了，[MediaPlayer]()在播放时会持有指定的锁，在暂定或停止时释放这个锁。
``` java
mMediaPlayer = new MediaPlayer();
// ... 这里其他的初始化操作 ...
mMediaPlayer.setWakeMode(getApplicationContext(), PowerManager.PARTIAL_WAKE_LOCK);
```
然而，在这个例子中，获取到的唤醒锁仅仅保证CPU保持工作。如果通过网络流媒体播放并且正在使用WIFI，你也许也要持有[WifiLock]()，这个你需要手动获取和释放。因此，当你开始准备通过远程URL使用[MediaPlayer]()，你应该创建和获取WIFI锁。例如：
``` java
WifiLock wifiLock = ((WifiManager) getSystemService(Context.WIFI_SERVICE))
    .createWifiLock(WifiManager.WIFI_MODE_FULL, "mylock");

wifiLock.acquire();
```
当你暂定或者停止媒体播放，不再需要网络，你应该释放这个锁：
``` java
wifiLock.release();
```

### 作为前台service运行
Service通常用于执行后台任务，例如获取邮件、同步数据、下载内容等等。在这些情况下，用户不需要知道service的执行情况，甚至不需要注意这些service是否中断并且之后又重启。

但是考虑到这个服务是要播放音乐。显而易见，这个service用户需要知道，并且任何打断都要严重影响用户体验。另外，在这个service这行期间，用户有可能希望与它交互。这种情况下，这个service应该运行成“前台service”。在系统中前台service有着很高的优先级——系统几乎不会杀死这个service，因为它对用户十分重要。service运行在前台时，必须提供一个状态栏通知，以确保用户知道正在运行的service，并且允许用户打开一个activity与service交互。

为了把service变成前台service，你必须为状态栏创建一个[Notification]()，并且在[Service]()调用[startForeground()]()。例如：
``` java
String songName;
// assign the song name to songName
PendingIntent pi = PendingIntent.getActivity(getApplicationContext(), 0,
                new Intent(getApplicationContext(), MainActivity.class),
                PendingIntent.FLAG_UPDATE_CURRENT);
Notification notification = new Notification();
notification.tickerText = text;
notification.icon = R.drawable.play0;
notification.flags |= Notification.FLAG_ONGOING_EVENT;
notification.setLatestEventInfo(getApplicationContext(), "MusicPlayerSample",
                "Playing: " + songName, pi);
startForeground(NOTIFICATION_ID, notification);
```

当你的service运行在前台，你配置的通知在设备通知区域中是可见的。如果用户选择通知，系统会调用你提供的[PendingIntent]()。在上面的例子中，打开了一个Activity(`MainActivity`)。

图1展示了通知如何展示给用户的：

![](https://developer.android.com/guide/topics/media/images/notification1.png)
![](https://developer.android.com/guide/topics/media/images/notification2.png)
图1 前台通知的截屏，在状态栏显示通知（左），在扩大的视图中显示（右）。

只有在服务真正执行一些用户需要知道的事情，才应该持有前台服务。一旦不是这个，应该调用[stopForeground]()释放。
``` java
stopForeground(true);
```

关于这个的更多信息，请参见关于[服务]()和[状态栏通知]()文档。

### 处理音频焦点
即使是在任何给定的时间，只能运行一个activity，但是Android是一个多任务的环境。这就在使用音频时给应用提出了一个特殊的挑战，因为只有一个音频输出设备，而也许有几个竞争使用的service。在Android2.2之前，没有建立一个机制来解决这个问题，在某些情况下会导致很差的用户体验。例如，当用户正在听音乐时，另一个应用需要通知用户非常重要的事情，用户也许由于大声的音乐而没有听见通知的声音。Android2.2开始，平台提供了一个应用之间协商使用设备音频输出设备的方案。这个机制称为音频焦点。

当你的应用需要音频输出，例如音乐或者通知，你需要请求音频焦点。一旦有了焦点，就可以使用音频输出设备，但是应该监听焦点改变。如果在得到失去音频焦点的通知后，应该立即杀死音频或者降低到安静的水平（被认为是"ducking"——这是指哪一个是合适地标志），并且重新接收到焦点之后又重启大声的播放。

音频焦点是一个自然地合作。就是说，希望（并且高度鼓舞）应用遵从音频焦点条例，但是系统并不强迫遵守规则。如果应用想在失去音频焦点后仍然播放大声的音乐，系统并不能阻止它。然而，用户可能体验的非常糟糕，并且有可能卸载这个不礼貌的应用。

为了请求音频焦点，你必须调用[AudioManager]()中的[requestAudioFocus()]()，正如下面实例描述的那样：
``` java
AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
int result = audioManager.requestAudioFocus(this, AudioManager.STREAM_MUSIC,
    AudioManager.AUDIOFOCUS_GAIN);

if (result != AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // could not get audio focus.
}
```
[requestAudioFocus()]()的第一个参数是[AudioManager.OnAudioFocusChangeListener]()，它的[onAudioFocusChange()]()方法会在音频焦点改变时被调用。因此，你应该在你的service和antivity中实现这个接口。例如：
``` java
class MyService extends Service
                implements AudioManager.OnAudioFocusChangeListener {
    // ....
    public void onAudioFocusChange(int focusChange) {
        // Do something based on focus change...
    }
}
```

`focusChange`参数表明了音频焦点是如何改变的，是下面的值之一（它们都是在[AudioManager]()定义的常量）：

- [AUDIOFOCUS_GAIN]()：你获得了音频焦点。
- [AUDIOFOCUS_LOSS]()：你失去了音频焦点一段可假定的很长时间。你必须停止所有的音频播放。因为你预期在很长时间内不会获得音频焦点，尽可能清空你的资源是一个很好的方式。例如，你应该释放[MediaPlayer]()。
- [AUDIOFOCUS_LOSS_TRANSIENT]()：你暂时失去焦点，但是很快会重新获得。你必须停止你所有的音频播放，但是你可以保持资源，因为很可能一会就会重新获得焦点，
- [AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK]()：你暂时失去焦点，但是你允许继续安静地播放音频（用很小的声音），而不用完全杀死音频。

这里是一个实现的例子：
``` java
public void onAudioFocusChange(int focusChange) {
    switch (focusChange) {
        case AudioManager.AUDIOFOCUS_GAIN:
            // 恢复播放
            if (mMediaPlayer == null) initMediaPlayer();
            else if (!mMediaPlayer.isPlaying()) mMediaPlayer.start();
            mMediaPlayer.setVolume(1.0f, 1.0f);
            break;

        case AudioManager.AUDIOFOCUS_LOSS:
            // 失去焦点不可控制的时间，需要停止播放并释放资源
            if (mMediaPlayer.isPlaying()) mMediaPlayer.stop();
            mMediaPlayer.release();
            mMediaPlayer = null;
            break;

        case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
            // 失去很短一段时间，但是必须要停止
            // 播放。不需要释放媒体播放资源，因为一会可能会恢复
            if (mMediaPlayer.isPlaying()) mMediaPlayer.pause();
            break;

        case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
            // 失去焦点一小段时间，但是可以小声的播放
            if (mMediaPlayer.isPlaying()) mMediaPlayer.setVolume(0.1f, 0.1f);
            break;
    }
}
```
在检测出系统API大于8，你可以创建一个`AudioFocusHelper`实例。例如：
``` java
if (android.os.Build.VERSION.SDK_INT >= 8) {
    mAudioFocusHelper = new AudioFocusHelper(getApplicationContext(), this);
} else {
    mAudioFocusHelper = null;
}
```

### 执行清理工作

正如之前提到的，[MediaPlayer]()对象会消耗很多系统资源，所以你应该只在需要时持有它，并且结束后调用[release()]()方法释放。明确地调用这个清理方法是很重要的，而不要依赖系统垃圾回收，因为在垃圾回收器回收[MediaPlayer]()之前也需要很长一点时间，因为垃圾回收器只在内存缺乏其他媒体相关资源不缺乏时才敏感。因此，当你使用service时，你总需要重写[onDestroy()]()方法以确保及时释放[MediaPlayer]()：
``` java
public class MyService extends Service {
   MediaPlayer mMediaPlayer;
   // ...

   @Override
   public void onDestroy() {
       if (mMediaPlayer != null) mMediaPlayer.release();
   }
}
```

除了在关闭时释放[MediaPlayer]()，你也需要在其他的时机释放它。如果你能预期到不能在很长时间内播放媒体（例如在失去音频焦点后），你需要明确地释放现有的[MediaPlayer]()，并且在之后重新创建。另一方面，如果你能预期到只是停止播放一小会，你应该持有[MediaPlayer]()以避免花费在重新创建和准备它。

## 处理AUDIO_BECOMING_NOISY意图
当发生导致音频变吵（通过外部扬声器发声）的事件发生时，许多优秀应用提供自动停止播放音频。例如，这个也许会发生在当用户用耳机听歌时突然从设备中拔出耳机。然而这个行为不是自动发生的。如果不实现这个功能，音频也许会通过设备的外部扬声器播放，也许不是用户想要的。

你可以通过处理[AUDIO_BECOMING_NOISY]()意图来确保你的app在这些情况下停止播放音乐，你可以通过在manifest文件中添加下面的代码来注册receiver。
``` java
<receiver android:name=".MusicIntentReceiver">
   <intent-filter>
      <action android:name="android.media.AUDIO_BECOMING_NOISY" />
   </intent-filter>
</receiver>
```

注册`MusicIntentReceiver`类作为那个意图的广播接受者。你应该这样实现这个类：
``` java
public class MusicIntentReceiver extends android.content.BroadcastReceiver {
   @Override
   public void onReceive(Context ctx, Intent intent) {
      if (intent.getAction().equals(
                    android.media.AudioManager.ACTION_AUDIO_BECOMING_NOISY)) {
          // 通知service停止播放
          // (例如通过Intent)
      }
   }
}
```

## 通过Content Resolver检索媒体
另一个在媒体播放应用中可以用的特性是能够检索用户设备上的音乐。你可以通过查询外部媒体[ContentResolver]()达成这个特性：
``` java
ContentResolver contentResolver = getContentResolver();
Uri uri = android.provider.MediaStore.Audio.Media.EXTERNAL_CONTENT_URI;
Cursor cursor = contentResolver.query(uri, null, null, null, null);
if (cursor == null) {
    // query failed, handle error.
} else if (!cursor.moveToFirst()) {
    // no media on the device
} else {
    int titleColumn = cursor.getColumnIndex(android.provider.MediaStore.Audio.Media.TITLE);
    int idColumn = cursor.getColumnIndex(android.provider.MediaStore.Audio.Media._ID);
    do {
       long thisId = cursor.getLong(idColumn);
       String thisTitle = cursor.getString(titleColumn);
       // ...process entry...
    } while (cursor.moveToNext());
}
```
为了在[MediaPlayer]()中使用它，你可以这样做：
``` java
long id = /* retrieve it from somewhere */;
Uri contentUri = ContentUris.withAppendedId(
        android.provider.MediaStore.Audio.Media.EXTERNAL_CONTENT_URI, id);

mMediaPlayer = new MediaPlayer();
mMediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
mMediaPlayer.setDataSource(getApplicationContext(), contentUri);

// ...prepare and start...
```