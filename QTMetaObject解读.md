- [QT MetaObject解读](#qt-metaobject解读)
  - [QObject::connect 解读](#qobjectconnect-解读)
    - [回顾 moc文件](#回顾-moc文件)
    - [connect函数的绑定实现](#connect函数的绑定实现)

# QT MetaObject解读

## QObject::connect 解读

使用connect函数得例子如下：  
```connect(sender, SIGNAL(sig(int)), reciver, SLOT(slot(int)));```

注意到2个宏定义：```SIGNAL```, ```SLOT```

```C
# define SLOT(a)     qFlagLocation("1"#a QLOCATION)
# define SIGNAL(a)   qFlagLocation("2"#a QLOCATION)
```
* 这两个宏实际上是把函数名加工转换为字符串，作为标识

connect函数实际做一下处理：
* 检查传入得信号槽合法性
* 将槽函数绑定到对应信号上

具体看代码：

```C++
QMetaObject::Connection QObject::connect(const QObject *sender, const char *signal,
                                     const QObject *receiver, const char *method,
                                     Qt::ConnectionType type)
{
    if (sender == 0 || receiver == 0 || signal == 0 || method == 0) {
        qWarning("QObject::connect: Cannot connect %s::%s to %s::%s",
                 sender ? sender->metaObject()->className() : "(null)",
                 (signal && *signal) ? signal+1 : "(null)",
                 receiver ? receiver->metaObject()->className() : "(null)",
                 (method && *method) ? method+1 : "(null)");
        return QMetaObject::Connection(0);
    }
    QByteArray tmp_signal_name;

    if (!check_signal_macro(sender, signal, "connect", "bind"))
        return QMetaObject::Connection(0);
    const QMetaObject *smeta = sender->metaObject();
    const char *signal_arg = signal;
    ++signal; //skip code
    QArgumentTypeArray signalTypes;
    Q_ASSERT(QMetaObjectPrivate::get(smeta)->revision >= 7);
    QByteArray signalName = QMetaObjectPrivate::decodeMethodSignature(signal, signalTypes);


    // 通过信号名 找到其索引号
    int signal_index = QMetaObjectPrivate::indexOfSignalRelative(
            &smeta, signalName, signalTypes.size(), signalTypes.constData());
    if (signal_index < 0) {
        // check for normalized signatures
        tmp_signal_name = QMetaObject::normalizedSignature(signal - 1);
        signal = tmp_signal_name.constData() + 1;

        signalTypes.clear();
        signalName = QMetaObjectPrivate::decodeMethodSignature(signal, signalTypes);
        smeta = sender->metaObject();
        signal_index = QMetaObjectPrivate::indexOfSignalRelative(
                &smeta, signalName, signalTypes.size(), signalTypes.constData());
    }
    if (signal_index < 0) {
        err_method_notfound(sender, signal_arg, "connect");
        err_info_about_objects("connect", sender, receiver);
        return QMetaObject::Connection(0);
    }
    signal_index = QMetaObjectPrivate::originalClone(smeta, signal_index);
    signal_index += QMetaObjectPrivate::signalOffset(smeta);

    QByteArray tmp_method_name;
    int membcode = extract_code(method);

    if (!check_method_code(membcode, receiver, method, "connect"))
        return QMetaObject::Connection(0);
    const char *method_arg = method;
    ++method; // skip code

    // 这里开始处理槽函数
    QArgumentTypeArray methodTypes;

    // 获取槽函数名
    QByteArray methodName = QMetaObjectPrivate::decodeMethodSignature(method, methodTypes);
    const QMetaObject *rmeta = receiver->metaObject();
    int method_index_relative = -1;
    Q_ASSERT(QMetaObjectPrivate::get(rmeta)->revision >= 7);
    switch (membcode) {
    case QSLOT_CODE:

        // 获取槽函数对应的索引
        method_index_relative = QMetaObjectPrivate::indexOfSlotRelative(
                &rmeta, methodName, methodTypes.size(), methodTypes.constData());
        break;
    case QSIGNAL_CODE:
        method_index_relative = QMetaObjectPrivate::indexOfSignalRelative(
                &rmeta, methodName, methodTypes.size(), methodTypes.constData());
        break;
    }
    if (method_index_relative < 0) {
        // check for normalized methods
        tmp_method_name = QMetaObject::normalizedSignature(method);
        method = tmp_method_name.constData();

        methodTypes.clear();
        methodName = QMetaObjectPrivate::decodeMethodSignature(method, methodTypes);
        // rmeta may have been modified above
        rmeta = receiver->metaObject();
        switch (membcode) {
        case QSLOT_CODE:
            method_index_relative = QMetaObjectPrivate::indexOfSlotRelative(
                    &rmeta, methodName, methodTypes.size(), methodTypes.constData());
            break;
        case QSIGNAL_CODE:
            method_index_relative = QMetaObjectPrivate::indexOfSignalRelative(
                    &rmeta, methodName, methodTypes.size(), methodTypes.constData());
            break;
        }
    }

    if (method_index_relative < 0) {
        err_method_notfound(receiver, method_arg, "connect");
        err_info_about_objects("connect", sender, receiver);
        return QMetaObject::Connection(0);
    }

    // 比对信号和槽参数一致性
    if (!QMetaObjectPrivate::checkConnectArgs(signalTypes.size(), signalTypes.constData(),
                                              methodTypes.size(), methodTypes.constData())) {
        qWarning("QObject::connect: Incompatible sender/receiver arguments"
                 "\n        %s::%s --> %s::%s",
                 sender->metaObject()->className(), signal,
                 receiver->metaObject()->className(), method);
        return QMetaObject::Connection(0);
    }

    int *types = 0;
    if ((type == Qt::QueuedConnection)
            && !(types = queuedConnectionTypes(signalTypes.constData(), signalTypes.size()))) {
        return QMetaObject::Connection(0);
    }

#ifndef QT_NO_DEBUG
    QMetaMethod smethod = QMetaObjectPrivate::signal(smeta, signal_index);
    QMetaMethod rmethod = rmeta->method(method_index_relative + rmeta->methodOffset());
    check_and_warn_compat(smeta, smethod, rmeta, rmethod);
#endif

    // 这里开始绑定
    QMetaObject::Connection handle = QMetaObject::Connection(QMetaObjectPrivate::connect(
        sender, signal_index, smeta, receiver, method_index_relative, rmeta ,type, types));
    return handle;
}
```

看看如何把信号/槽函数转换为索引

```C++
int QMetaObjectPrivate::indexOfSlotRelative(const QMetaObject **m,
                                            const QByteArray &name, int argc,
                                            const QArgumentType *types)
{
    return indexOfMethodRelative<MethodSlot>(m, name, argc, types);
}

template<int MethodType>
static inline int indexOfMethodRelative(const QMetaObject **baseObject,
                                        const QByteArray &name, int argc,
                                        const QArgumentType *types)
{
    for (const QMetaObject *m = *baseObject; m; m = m->d.superdata) {
        Q_ASSERT(priv(m->d.data)->revision >= 7);
        int i = (MethodType == MethodSignal)
                 ? (priv(m->d.data)->signalCount - 1) : (priv(m->d.data)->methodCount - 1);
        const int end = (MethodType == MethodSlot)
                        ? (priv(m->d.data)->signalCount) : 0;

        for (; i >= end; --i) {
            int handle = priv(m->d.data)->methodData + 5*i;

            // 比对函数名从而获取索引
            if (methodMatch(m, handle, name, argc, types)) {
                *baseObject = m;
                return i;
            }
        }
    }
    return -1;
}
```

注意到这里的变量 上面函数的各种参数是什么意思呢？

### 回顾 moc文件

我们尝试看看QT 由qmake生成的各个类对应的moc文件我们看到自动生成的一些代码:
例子:

```C++

QT_BEGIN_MOC_NAMESPACE
QT_WARNING_PUSH
QT_WARNING_DISABLE_DEPRECATED
struct qt_meta_stringdata_AvmCarModel_t {
    QByteArrayData data[8];
    char stringdata0[76];
};
#define QT_MOC_LITERAL(idx, ofs, len) \
    Q_STATIC_BYTE_ARRAY_DATA_HEADER_INITIALIZER_WITH_OFFSET(len, \
    qptrdiff(offsetof(qt_meta_stringdata_AvmCarModel_t, stringdata0) + ofs \
        - idx * sizeof(QByteArrayData)) \
    )
static const qt_meta_stringdata_AvmCarModel_t qt_meta_stringdata_AvmCarModel = {
    {
QT_MOC_LITERAL(0, 0, 11), // "AvmCarModel"
QT_MOC_LITERAL(1, 12, 4), // "back"
QT_MOC_LITERAL(2, 17, 0), // ""
QT_MOC_LITERAL(3, 18, 7), // "okClick"
QT_MOC_LITERAL(4, 26, 11), // "cancelClick"
QT_MOC_LITERAL(5, 38, 12), // "modelChanged"
QT_MOC_LITERAL(6, 51, 11), // "colorChange"
QT_MOC_LITERAL(7, 63, 12) // "numPalChange"

    },
    "AvmCarModel\0back\0\0okClick\0cancelClick\0"
    "modelChanged\0colorChange\0numPalChange"
};
#undef QT_MOC_LITERAL

static const uint qt_meta_data_AvmCarModel[] = {

 // content:
       8,       // revision
       0,       // classname
       0,    0, // classinfo
       6,   14, // methods
       0,    0, // properties
       0,    0, // enums/sets
       0,    0, // constructors
       0,       // flags
       1,       // signalCount

 // signals: name, argc, parameters, tag, flags
       1,    0,   44,    2, 0x06 /* Public */,

 // slots: name, argc, parameters, tag, flags
       3,    0,   45,    2, 0x08 /* Private */,
       4,    0,   46,    2, 0x08 /* Private */,
       5,    1,   47,    2, 0x08 /* Private */,
       6,    1,   50,    2, 0x08 /* Private */,
       7,    1,   53,    2, 0x08 /* Private */,

 // signals: parameters
    QMetaType::Void,

 // slots: parameters
    QMetaType::Void,
    QMetaType::Void,
    QMetaType::Void, QMetaType::Int,    2,
    QMetaType::Void, QMetaType::Int,    2,
    QMetaType::Void, QMetaType::Int,    2,

       0        // eod
};


QT_INIT_METAOBJECT const QMetaObject AvmCarModel::staticMetaObject = { {
    &KeyEventOpt::staticMetaObject,
    qt_meta_stringdata_AvmCarModel.data,
    qt_meta_data_AvmCarModel,
    qt_static_metacall,
    nullptr,
    nullptr
} };

```

注意到 
* ```staticMetaObject``` 就是这个类的metaobject
* ```qt_meta_data_AvmCarModel```就是用于信号槽名字跟索引匹配的关键结构体
* ```qt_meta_data_AvmCarModel``` 则是用于保存这个类信号槽的信息


```staticMetaObject``` 对应初始化
```C++
struct Q_CORE_EXPORT QMetaObject{
    // ...
    struct { // private data
        const QMetaObject *superdata;
        const QByteArrayData *stringdata; // qt_meta_stringdata_AvmCarModel.data
        const uint *data;                 // qt_meta_data_AvmCarModel
        typedef void (*StaticMetacallFunction)(QObject *, QMetaObject::Call, int, void **); // qt_static_metacall
        StaticMetacallFunction static_metacall;
        const QMetaObject * const *relatedMetaObjects;
        void *extradata; //reserved for future use
    } d;
}
```

回到前面 ```indexOfMethodRelative```函数匹配过程，就可以看到
实际上 就是获取 ```qt_meta_stringdata_AvmCarModel```里面的字符串去比较

### connect函数的绑定实现

继续 往下看```QObject::connect```源码：

```C++
    QMetaObject::Connection handle = QMetaObject::Connection(QMetaObjectPrivate::connect(
        sender, signal_index, smeta, receiver, method_index_relative, rmeta ,type, types));
```
这里就是实际上将信号跟槽绑定的地方

```C++

QObjectPrivate::Connection *QMetaObjectPrivate::connect(const QObject *sender,
                                 int signal_index, const QMetaObject *smeta,
                                 const QObject *receiver, int method_index,
                                 const QMetaObject *rmeta, int type, int *types)
{
    QObject *s = const_cast<QObject *>(sender);
    QObject *r = const_cast<QObject *>(receiver);

    int method_offset = rmeta ? rmeta->methodOffset() : 0;
    Q_ASSERT(!rmeta || QMetaObjectPrivate::get(rmeta)->revision >= 6);

    // 这里是槽函数函数指针
    QObjectPrivate::StaticMetaCallFunction callFunction =
        rmeta ? rmeta->d.static_metacall : 0;

    // .....

    QScopedPointer<QObjectPrivate::Connection> c(new QObjectPrivate::Connection);
    c->sender = s;
    c->signal_index = signal_index;
    c->receiver = r;
    c->method_relative = method_index;
    c->method_offset = method_offset;
    c->connectionType = type;
    c->isSlotObject = false;
    c->argumentTypes.store(types);
    c->nextConnectionList = 0;
    c->callFunction = callFunction;

    // 将槽函数绑定到信号对应的 metaobject
    QObjectPrivate::get(s)->addConnection(signal_index, c.data());

    // ....

    return c.take();
}

```