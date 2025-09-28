- [QMediaPlayer解读](#qmediaplayer解读)
  - [简介](#简介)
  - [使用](#使用)
  - [分析](#分析)
    - [QMediaPlayer](#qmediaplayer)
      - [构造函数](#构造函数)
      - [疑问](#疑问)
      - [解答](#解答)
      - [添加播放媒体源](#添加播放媒体源)
        - [QVideoWidget](#qvideowidget)


# QMediaPlayer解读

## 简介
```QMediaPlayer```是QT 多媒体播放器类, 是对多媒体相关的高度集成了多媒体相关功能的类。虽然高度集成了多媒体一些功能，但是底层并没有自己实现音视频的编解码功能，需要针对系统平台依赖不同多媒体后端，比如在Windows 需要依赖LAVFilter, 在Linux 需要依赖GStreamer。

## 使用

这里引用官方例子：

播放单个视频：
```C++
player = new QMediaPlayer;
connect(player, SIGNAL(positionChanged(qint64)), this, SLO(positionChanged(qint64)));

player->setMedia(QUrl::fromLocalFile("/Users/me/Music/coolsong.mp3"));
player->setVolume(50);
player->play();
```

播放列表视频
```C++
playlist = new QMediaPlaylist;
playlist->addMedia(QUrl("http://example.com/movie1.mp4"));
playlist->addMedia(QUrl("http://example.com/movie2.mp4"));
playlist->addMedia(QUrl("http://example.com/movie3.mp4"));
playlist->setCurrentIndex(1);

player = new QMediaPlayer;
player->setPlaylist(playlist);

videoWidget = new QVideoWidget;
player->setVideoOutput(videoWidget);
videoWidget->show();

player->play();
```

这里需要关注两个类：```QMediaPlaylist```：播放列表, 
```QVideoWidget```：画面展示窗口

## 分析

### QMediaPlayer

简单直接从例子深入源码, 这里以 Qt5.12.5为例

#### 构造函数
```C++
QMediaPlayer::QMediaPlayer(QObject *parent, QMediaPlayer::Flags flags):
    QMediaObject(*new QMediaPlayerPrivate,
                 parent,
                 playerService(flags))
{
    Q_D(QMediaPlayer);
    
    // 获取插件服务类的提供者
    d->provider = QMediaServiceProvider::defaultServiceProvider();
    if (d->service == 0) {
        d->error = ServiceMissingError;
    } else {
        
        // 获取多媒体控制者
        d->control = qobject_cast<QMediaPlayerControl*>(d->service->requestControl(QMediaPlayerControl_iid));
        
        d->networkAccessControl = qobject_cast<QMediaNetworkAccessControl*>(d->service->requestControl(QMediaNetworkAccessControl_iid));
        if (d->control != 0) {
            // 信号槽链接等操作 
        }
    }
}
```
2处关键
* ```QMediaServiceProvider::defaultServiceProvider()```
  * 可以看作服务提供者,可以用来获取各种服务
* ```d->service->requestControl(QMediaPlayerControl_iid)```
  * 用于获取具体服务类, 所有跟多媒体相关的控制操作都是调用此控制类比如 设置音量, 进度条定位,播放/暂停等

#### 疑问

看了上面代码是否有这样疑问: 
* d->service 哪里初始化的? 
* 这个service的具体的类是哪个呢?
* d->control的具体实现类又是哪个?

#### 解答

* 答案在 ```playerService(flags)```

    ```C++
    static QMediaService *playerService(QMediaPlayer::Flags flags)
    {
        QMediaServiceProvider *provider = QMediaServiceProvider::defaultServiceProvider();

        // ....

        return provider->requestService(Q_MEDIASERVICE_MEDIAPLAYER);
    }

    ```
    可以看到 service由provider获取的

* 而service具体是哪个类呢?   
  * <font color=red>这里要根据平台相关的,以Linux平台为例, Linux下多媒体依赖gstreamer,所以 service的具体实现类就是 ```QGstreamerPlayerService```</font>
  * <font color=red>而如果是windows下则是 ```DirectShowPlayerService```</font>
  * <font color=red>```QGstreamerPlayerService``` 具体实现其实就是GStreamer API的封装</font>

* 既然底层实现是gstreamer那么 d->control就可以轻易推测也跟gstreamer有关的: ```QGstreamerPlayerControl```  

#### 添加播放媒体源

QMediaPlayer是如何添加一个媒体源呢?

##### QVideoWidget

从上面我们知道了 ```QMediaplayer``` 底层实现上, 各个关联类, 那么底层获取的码流是怎么绘制到对应的widget上的呢？

从官方例子知道, 调用```setVideoOutput```才能把视频流关联到窗口, 那么我们看看这个函数源码：

```C++
void QMediaPlayer::setVideoOutput(QVideoWidget *output)
{
    Q_D(QMediaPlayer);

    if (d->videoOutput)
        unbind(d->videoOutput);

    // We don't know (in this library) that QVideoWidget inherits QObject
    QObject *outputObject = reinterpret_cast<QObject*>(output);

    // 关键点在 bind函数
    d->videoOutput = outputObject && bind(outputObject) ? outputObject : 0;
}
```

关键点在```bind(outputObject)```

```C++
bool QMediaObject::bind(QObject *object)
{
    QMediaBindableInterface *helper = qobject_cast<QMediaBindableInterface*>(object);

    // ...

    return helper->setMediaObject(this);
}
```
```bind()```最终调用了 ```QMediaBindableInterface::setMediaObject```, 
而QVideoWidget 继承自 QMediaBindableInterface, 那么直接看而QVideoWidget对setMediaObject重载后的实现：

```C++
bool QVideoWidget::setMediaObject(QMediaObject *object)
{
    //....

    if (d->service) {
        if (d->createWidgetBackend()) {
            // Nothing to do here.
        } else if ((!window() || !window()->testAttribute(Qt::WA_DontShowOnScreen))
                && d->createWindowBackend()) {
            if (isVisible())
                d->windowBackend->showEvent();
        } else if (d->createRendererBackend()) {
            if (isVisible())
                d->rendererBackend->showEvent();
        } else {
            d->service = 0;
            d->mediaObject = 0;

            return false;
        }

        connect(d->service, SIGNAL(destroyed()), SLOT(_q_serviceDestroyed()));
    } else {
        d->mediaObject = 0;

        return false;
    }

    return true;
}
```

可以看到 调用了 createWidgetBackend,

```C++
bool QVideoWidgetPrivate::createWidgetBackend()
{
    if (QMediaControl *control = service->requestControl(QVideoWidgetControl_iid)) {
        if (QVideoWidgetControl *widgetControl = qobject_cast<QVideoWidgetControl *>(control)) {
            widgetBackend = new QVideoWidgetControlBackend(service, widgetControl, q_func());

        }
    }
}
```

这里从服务种请求了某种媒体控制，可以看看请求了什么控制：

```C++
QMediaControl *QGstreamerPlayerService::requestControl(const char *name)
{
    // ......

    if (!m_videoOutput) {

        // if ()....
#if defined(HAVE_WIDGETS)
        else if (qstrcmp(name, QVideoWidgetControl_iid) == 0)
            m_videoOutput = m_videoWidget;
#endif

        if (m_videoOutput) {
            increaseVideoRef();
            m_control->setVideoOutput(m_videoOutput);
            return m_videoOutput;
        }
    }

    return 0;
}
```
可以看到 前面说的某种控制就是 ```QVideoWidgetControlBackend* m_videoWidget```


那接下来 继续往下走可以看到

```C++
widgetBackend = new QVideoWidgetControlBackend(service, widgetControl, q_func());


QVideoWidgetControlBackend::QVideoWidgetControlBackend(
        QMediaService *service, QVideoWidgetControl *control, QWidget *widget)
    : m_service(service)
    , m_widgetControl(control)
{
    // .... 

    QBoxLayout *layout = new QVBoxLayout;

    QWidget *videoWidget = control->videoWidget();

    layout->addWidget(videoWidget);

    widget->setLayout(layout);
}
```

到这里可以看到 
* 调用了```control->videoWidget()```, 而这个control就是 m_videoWidget, 
* 而widget就是我们前面设置的QVideoWidget, 真正绘制视频流的窗口是 videoWidget, 只不过是videoWidget通过布局方式添加到QVideoWidget上面。

```C++
QWidget *QGstreamerVideoWidgetControl::videoWidget()
{
    createVideoWidget();
    return m_widget;
}

void QGstreamerVideoWidgetControl::createVideoWidget()
{
    if (m_widget)
        return;

    m_widget = new QGstreamerVideoWidget;

    m_widget->installEventFilter(this);
    m_videoOverlay.setWindowHandle(m_windowId = m_widget->winId());
}

void QGstreamerVideoOverlay::setWindowHandle(WId id)
{
    m_windowId = id;

    if (isActive())
        setWindowHandle_helper(id);
}

void QGstreamerVideoOverlay::setWindowHandle_helper(WId id)
{
#if GST_CHECK_VERSION(1,0,0)
    if (m_videoSink && GST_IS_VIDEO_OVERLAY(m_videoSink)) {
        gst_video_overlay_set_window_handle(GST_VIDEO_OVERLAY(m_videoSink), id);
    }
}
```

走到 ```setWindowHandle_helper``` 就可以看得出来：
* <font color=red>通过获取videoWidget 的窗口ID</font>
* <font color=red>然后将ID 绑定到 gstreamer的输出上, 然后gstreamer的视频流就会在对应窗口上绘制</font>


