- [QEventLoop解读](#qeventloop解读)
  - [使用](#使用)
  - [疑问](#疑问)
    - [解答](#解答)


# QEventLoop解读

```QEventLoop```可以说是Qt最重要的一个基础类, 是Qt的事件循环机制核心, 下面就从源码角度解读它的实现以及特性

## 使用

在我们编写Qt代码时,在main函数都会定义 ```QCoreApplication```类型变量并调用```exec()```函数. 实际上这个函数内部就是使用了```QEventLoop```实现.

首先看构造函数
```C++
QEventLoop::QEventLoop(QObject *parent): QObject(*new QEventLoopPrivate, parent)
{
    Q_D(QEventLoop);
    if (!QCoreApplication::instance() && QCoreApplicationPrivate::threadRequiresCoreApplication()) {
        qWarning("QEventLoop: Cannot be used without QApplication");
    } else {
        d->threadData->ensureEventDispatcher();
    }
}

QAbstractEventDispatcher *QThreadData::createEventDispatcher()
{
    QAbstractEventDispatcher *ed = QThreadPrivate::createEventDispatcher(this);
    eventDispatcher.storeRelease(ed);
    ed->startingUp();
    return ed;
}
```

* 先判断QCoreApplication是否已经创建
* 创建事件分发者，实际上```createEventDispatcher```会根据不同平台创建不同dispatcher.(文章后续依赖的是Linux平台，所以跟dispatcher有关的都以Linux平台为例子)

接下来看看核心函数
```C++

int QEventLoop::exec(ProcessEventsFlags flags)
{
    //we need to protect from race condition with QThread::exit
    QMutexLocker locker(&static_cast<QThreadPrivate *>(QObjectPrivate::get(d->threadData->thread))->mutex);

    // ...

    struct LoopReference {
        QEventLoopPrivate *d;
        QMutexLocker &locker;

        bool exceptionCaught;
        LoopReference(QEventLoopPrivate *d, QMutexLocker &locker) : d(d), locker(locker), exceptionCaught(true)
        {
            d->inExec = true;
            d->exit.storeRelease(false);
            ++d->threadData->loopLevel;
            d->threadData->eventLoops.push(d->q_func());
            locker.unlock();
        }

        ~LoopReference()
        {
            if (exceptionCaught) {
            }
            locker.relock();
            QEventLoop *eventLoop = d->threadData->eventLoops.pop();
            Q_ASSERT_X(eventLoop == d->q_func(), "QEventLoop::exec()", "internal error");
            Q_UNUSED(eventLoop); // --release warning
            d->inExec = false;
            --d->threadData->loopLevel;
        }
    };
    LoopReference ref(d, locker);

    // remove posted quit events when entering a new event loop
    QCoreApplication *app = QCoreApplication::instance();
    if (app && app->thread() == thread())
        QCoreApplication::removePostedEvents(app, QEvent::Quit);

    while (!d->exit.loadAcquire())
        processEvents(flags | WaitForMoreEvents | EventLoopExec);

    ref.exceptionCaught = false;
    return d->returnCode.load();
}```

可以看到
* 整段代码最终会除开一些多余定义，最终都会走到 ```processEvents```函数

```C++
bool QEventLoop::processEvents(ProcessEventsFlags flags)
{
    Q_D(QEventLoop);
    if (!d->threadData->hasEventDispatcher())
        return false;
    return d->threadData->eventDispatcher.load()->processEvents(flags);
}

bool QEventDispatcherUNIX::processEvents(QEventLoop::ProcessEventsFlags flags)
{
    Q_D(QEventDispatcherUNIX);
    d->interrupt.store(0);

    // we are awake, broadcast it
    emit awake();
    QCoreApplicationPrivate::sendPostedEvents(0, 0, d->threadData);

    // ...

    timespec *tm = nullptr;
    timespec wait_tm = { 0, 0 };

    if (!canWait || (include_timers && d->timerList.timerWait(wait_tm)))
        tm = &wait_tm;

    d->pollfds.clear();
    d->pollfds.reserve(1 + (include_notifiers ? d->socketNotifiers.size() : 0));

    // ...

    switch (qt_safe_poll(d->pollfds.data(), d->pollfds.size(), tm)) {
    // ...

    if (include_timers)
        nevents += d->activateTimers();

    // return true if we handled events, false otherwise
    return (nevents > 0);
}
```

* 最终会走到```qt_safe_poll```这个函数进行处理
* 实际上```qt_safe_poll```会调用```poll```函数

## 疑问

既然```QEventLoop```最终依赖poll函数

1. 信号发送和事件发送是怎么通过QEventLoop进行发送的？

### 解答

回顾一下如何异步发送事件
* 利用 ```QCoreApplication::postEvent```发送

看看其内部实现
```C++
void QCoreApplication::postEvent(QObject *receiver, QEvent *event, int priority)
{
    Q_TRACE_SCOPE(QCoreApplication_postEvent, receiver, event, event->type());

    QThreadData * volatile * pdata = &receiver->d_func()->threadData;
    QThreadData *data = *pdata;
    if (!data) {
        // posting during destruction? just delete the event to prevent a leak
        delete event;
        return;
    }


    QMutexUnlocker locker(&data->postEventList.mutex);

    // ...

    // delete the event on exceptions to protect against memory leaks till the event is
    // properly owned in the postEventList
    QScopedPointer<QEvent> eventDeleter(event);
    Q_TRACE(QCoreApplication_postEvent_event_posted, receiver, event, event->type());
    data->postEventList.addEvent(QPostEvent(receiver, event, priority));
    eventDeleter.take();
    event->posted = true;
    ++receiver->d_func()->postedEvents;
    data->canWait = false;
    locker.unlock();

    QAbstractEventDispatcher* dispatcher = data->eventDispatcher.loadAcquire();
    if (dispatcher)
        dispatcher->wakeUp();
}
```

精简代码后, 可以看到
* ```data->postEventList.addEvent(QPostEvent(receiver, event, priority));``` 把事件加入到对应队列中
* 获取并唤醒对应 dispatcher的事件循环

**怎么唤醒呢？**

```C++
void QThreadPipe::wakeUp()
{
    if (wakeUps.testAndSetAcquire(0, 1)) {
#ifndef QT_NO_EVENTFD
        if (fds[1] == -1) {
            // eventfd
            eventfd_t value = 1;
            int ret;
            EINTR_LOOP(ret, eventfd_write(fds[0], value));
            return;
        }
#endif
        char c = 0;
        qt_safe_write(fds[1], &c, 1);
    }
}
```

* 就是利用了前面poll函数, 先注册一个描述符，向其写入数据，poll函数就会马上返回并处理


**怎么处理呢？**

回顾 processEvents函数我们就知道了

```C++
bool QEventDispatcherUNIX::processEvents(QEventLoop::ProcessEventsFlags flags)
{
    emit awake();
    QCoreApplicationPrivate::sendPostedEvents(0, 0, d->threadData);
}

// 由于代码有点长这里直接给出调用栈
sendPostedEvents
->   sendEvent(receiver, event)
->      notifyInternal2(receiver, event)
->          doNotify(receiver, event)
->              QCoreApplicationPrivate::notify_helper(receiver, event)
->                  QObject::event(event)
```

总结下来：
* ```QEventLoop``` 事件循环依赖poll函数(linux平台)
* 事件发送通过唤醒poll函数 进而调用对应处理
* 全部跟异步相关的都依赖``QEventLoop```