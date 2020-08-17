---
title: '实现一个简单的音乐播放器'
date: 2020-05-31 18:08:00
tags: [Service]
categories: [Android,Service]
---

### 前言
如何实现一个音乐播放器，其实算是一个基础的问题，我们在一开始学习安卓四大组件的Service组件的时候，就会碰上。其实Google的官方教程对这方面有比较详细的介绍。

文章介绍以下方面的几个问题。

- Service实现
- 通知栏处理
- 音频焦点管理
- 线控以及蓝牙适配
- 桌面控件

以上部分功能只做大致介绍

<!-- more -->

### 播放服务实现
可以参考安卓开发者官网的教程[在 Service 中使用 MediaPlayer](https://developer.android.google.cn/guide/topics/media/mediaplayer#mpandservices)。

关于我们的音乐播放器的绑定Service是否要使用AIDL的方式，其实不一定，如果我们需要音乐播放服务在另一个进程，那么是需要使用AIDL的方式的，如果不是，其实可以直接继承Binder类，在学习Service的时候我们应该知道一个AIDL的服务端怎么写(继承Binder类，实现接口)。

Google官方也是希望我们在大多数的时候直接继承Binder类来实现一个绑定服务的，可以参考[创建绑定服务#扩展 Binder 类](https://developer.android.google.cn/guide/components/bound-services#Creating)。

既然我们的播放器的实现在Service里面，那么我们需要写一些方法来控制播放以及从播放器获取当前的播放数据。

接口:
```java
public interface IPlayerController {

    void startBkgAudio(String url, String id);

    boolean isPlaying();

    PlayerCurrentData getCurrentPlayData();

}
```

服务端实现:
```kotlin
class RelaxPlayerBinder(private val player: IPlayerController) : Binder(), IPlayerController {

    override fun startBkgAudio(url: String?, id: String?) {
        player.startBkgAudio(type, minute, url, id)
    }

    override fun isPlaying(): Boolean {
        return player.isPlaying
    }

    override fun getCurrentPlayData(): PlayerCurrentData {
        return player.currentPlayData
    }
}
```

播放器实现:
```java
public class RelaxPlayerService extends Service implements IPlayerController {
    private RxMediaPlayerManager mBkgAudioMediaPlayer;

    @Override
    public void onCreate() {
        super.onCreate();
        initMediaPlayer();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return START_NOT_STICKY;
    }

    /**
     * 初始化播放器
     */
    private void initMediaPlayer() {
        mBkgAudioMediaPlayer = new RxMediaPlayerManager(this);
        mBkgAudioMediaPlayer.subscribeMediaPlayer()
                .compose(RxSchedulerUtil.ioToMain())
                .subscribe(audioBkgObserver);
    }

    // ####################### 播放器操作 start ########################

    @Override
    public void startBkgAudio(String url, String id) {
        mCurrentData.bkgAudioUrl = url;
        mCurrentData.bkgAudioId = id;
        MediaDataSource dataSource = new MediaDataSource(Uri.parse(url));
        MediaOption opt = new MediaOption.Builder(dataSource)
                .setLoop(true)
                .build();
        mBkgAudioMediaPlayer.applyMediaOption(opt);
        mBkgAudioMediaPlayer.start()
                .subscribe(RxUtil.nothingObserver());
        if (mPlayerEvent != null) {
            mPlayerEvent.onPlayStart();
        }
        //开启计时器
        long time = mCountDownSecond > 0 ? mCountDownSecond * 1000 : mCurrentData.minute * 60 * 1000;
        calculateTime(time);
    }

    @Override
    public boolean isPlaying() {
        return mBkgAudioMediaPlayer.isPlaying();
    }

    @Override
    public PlayerCurrentData getCurrentPlayData() {
        return mCurrentData;
    }

    // ####################### 播放器操作 end ########################
}
```


### 通知栏处理
当我们在后台播放音乐，设备为了节省电量可能在Service运行播放期间进入休眠状态，系统可能会关闭掉不必要的功能，比如CPU和WLAN硬件，所以在这种情况下，我们需要防止系统干扰播放，需要在初始化MediaPlayer时调用setWakeMode()方法，如果是播放网络音乐可能还得获得WiFi Lock，具体可以参考[使用唤醒锁定](https://developer.android.google.cn/guide/topics/media/mediaplayer#wakelocks)。

我们的音乐播放器如果想要在最小化的情况下还继续播放，而不至于在最小化一段时间之后就被系统杀掉，需要启用前台服务的形式，前台服务需要bind一个通知。

在我们bind一个媒体播放通知栏的时候，一般会考虑使用`MediaStyle`样式的通知栏，可以考虑引入最新的`androidx.media`包。
```
implementation 'androidx.media:media:1.1.0'
```

`MediaStyle`的样式可以接收最多5个按钮，最小化的时候3个(由于关闭按钮是可以一直在的，所以可以认为是4个，6个)，如下：
![big_style](/images/media_style_big@2x.png)
然后长按这个通知栏出现如下:
![small_style](/images/media_style_small@2x.png)
我们新增的Action Icon，系统会经过一层处理，转换成可适配的样式。


初始化：
我们在Service的onCreate里面初始化通知和MediaSession(Allows interaction with media controllers, volume keys, media buttons, and transport controls.)，看介绍是关于硬件设备比如耳机的线控相关的事件的接收的。
```java
private void initNotify() {
    mHandler = new Handler(Looper.getMainLooper());
    mSessionManager = new MediaSessionManager(this, mBinder, mHandler);
    mNotifyManager = new NotifyManager(this, mBinder, mSessionManager.getToken());
}
```

通知管理类:
```java
public class NotifyManager {

    private Service mService;
    private Notification mNotify;
    private IPlayerController mController;
    private MediaSessionCompat.Token mToken;
    private NotificationManager mNotificationManager;

    public NotifyManager(Service service, IPlayerController controller, MediaSessionCompat.Token token) {
        mService = service;
        mController = controller;
        mToken = token;
        mNotificationManager = (NotificationManager) mService.getSystemService(NOTIFICATION_SERVICE);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            int importance = NotificationManager.IMPORTANCE_DEFAULT;
            NotificationChannel channel = new NotificationChannel(AppKey.NotificationChannel.WH_NOTIFICATION_CHANEL_ID,
                    AppKey.NotificationChannel.WH_NOTIFICATION_CHANEL_NAME, importance);
            mNotificationManager.createNotificationChannel(channel);
        }
    }

    private NotificationCompat.Builder create(boolean isPlaying) {
        int pp = isPlaying ? R.drawable.relax_notify_pause : R.drawable.relax_notify_play;
        NotificationCompat.Builder builder = new NotificationCompat.Builder(mService,
                AppKey.NotificationChannel.WH_NOTIFICATION_CHANEL_ID)
                .setOngoing(true)
                .setSmallIcon(R.drawable.base_tools_heart_warm)
                .setContentTitle(mService.getString(R.string.relax_tools))
                .setContentText(DetailType.getTypeString(mService, mController.getCurrentPlayData().type))
                .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
                .addAction(pp, "播放/暂停", retrievePlaybackAction(RelaxPlayerService.ACTION_PLAY_PAUSE));
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            androidx.media.app.NotificationCompat.MediaStyle style = new androidx.media.app.NotificationCompat.MediaStyle()
                    .setMediaSession(mToken)
                    .setShowActionsInCompactView(0);
            builder.setStyle(style);
        }
        return builder;
    }

    private PendingIntent retrievePlaybackAction(final String action) {
        Intent intent = new Intent(action);
        return PendingIntent.getBroadcast(mService, 0, intent, 0);
    }

    private void setupNotify(boolean isPlaying) {
        mNotify = create(isPlaying).build();
    }

    public void showNotify(boolean isPlaying) {
        setupNotify(isPlaying);
        mService.startForeground(AppKey.NotificationChannel.WH_NOTIFICATION_ID, mNotify);
    }

    public void hideNotify() {
        mNotificationManager.cancel(AppKey.NotificationChannel.WH_NOTIFICATION_ID);
        mService.stopForeground(true);
    }
}
```
然后在音乐状态改变的时候调用对应的方法即可。

最后实现的效果:
![small_style](/images/media_notify_effict@2x.png)

### 线控以及蓝牙适配
我们在上面也提到了MediaSession这个类，这个类，可以接受一些事件，我们做处理即可。
```java
private static final long MEDIA_SESSION_ACTIONS =
    PlaybackStateCompat.ACTION_PLAY |
    PlaybackStateCompat.ACTION_PLAY_PAUSE |
    PlaybackStateCompat.ACTION_PAUSE |
    PlaybackStateCompat.ACTION_STOP;
mMediaSession.setCallback(mCallback, handler);
mMediaSession.setPlaybackState(new PlaybackStateCompat.Builder().setActions(MEDIA_SESSION_ACTIONS).build());
private MediaSessionCompat.Callback mCallback = new MediaSessionCompat.Callback() {

    @Override
    public void onPlay() {
        super.onPlay();
    }

};
```

耳机拔插等也是监听一些特定的广播，比如:
```java
intentFilter.addAction(AudioManager.ACTION_AUDIO_BECOMING_NOISY); //有线耳机拔出变化
intentFilter.addAction(BluetoothHeadset.ACTION_CONNECTION_STATE_CHANGED); //蓝牙耳机连接变化
```


### 音频焦点管理
获取AudioManager作一些处理，当然可能存在别的app，获取了焦点不释放的问题。


### 桌面控件
继承AppWidgetProvider点击事件发送一些PendingIntent处理。