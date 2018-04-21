## 知道广播吗? 

`BroadcastReceiver` (广播接收器), 是 Android 四大组件之一. 可以**监听/接收**应用自身/应用之间/安卓系统发出的广播消息，并做出响应.

## 它有几种注册/使用方式

1. **静态注册**: 在 AndroidManifest.xml 里通过 <receive> 标签声明
2. **动态注册**: 在代码中创建并通过 `Context.registerReceiver()` 注册接收器

## 广播有哪些类型

发送的广播大体可以分为 2 类:

1. Normal broadcasts 普通广播也叫无序广播, 会异步的发送给所有的 Receiver, 接收到广播的顺序是不确定的, 有可能是同时.
2. Ordered broadcasts 有序广播, 广播会先发送给优先级高 (android:priority 然后动态注册优先) 的 Receiver, 而且这个 Receiver 有权修改广播内容或者是决定是继续发送到下一个 Receiver 或者是直接终止广播.

我们还可以进一步将广播细分为 5 类:

1. Normal Broadcast - 普通广播, 即开发者自己定义 intent 发送广播 sendBroadcast(intent)
2. System Broadcast - 系统广播, Android 中内置了多个系统广播, 只要涉及到手机的基本操作, 如开机, 网络状态变化等
3. Ordered Broadcast - 有序广播
4. Sticky Broadcast - 粘性广播, 粘性广播在发送后就一直存在于系统的消息容器里面, 等待对应的处理器去处理, 如果暂时没有处理器处理这个消息则一直在消息容器里面处于等待状态, 粘性广播的Receiver如果被销毁, 那么下次重建时会自动接收到消息数据
但该方式目前已经 deprecated.
5. Local Broadcast - 应用内广播

## 有时候基于数据安全考虑, 我们想发送广播只有自己 (本进程) 能接收到，那么该如何去做呢?

即使用应用内广播:

有下列方式使用本地广播:

#### 方法1

1. 注册广播时将 exported 属性设置为 false, 使得非本应用内部发出的此广播不被接收
2. 在广播发送和接收时, 增设相应 permission, 用于权限验证
3. 发送广播时指定该广播接收器所在的包名, 此广播只会发送到与包名相匹配的有效广播接收器中, 通过 
`intent.setPackage(packageName)` 指定包名

但是我们知道 APK 太容易被反编译, 注册广播的权限也只是一个字符串, 并不安全

#### 方法2 LocalBroadcastManager

LocalBroadcastManager 方式发送的应用内广播, **只能通过 LocalBroadcastManager 动态注册, 不能静态注册**

## BroadcastReceiver的生命周期 / onReceive 中能否开线程进行耗时操作

BroadcastReceiver 很短, 当它的 onReceive 方法执行完成后, 它的生命周期就结束了, 被系统杀掉的概率极高，如果在 onReceive 去开线程进行异步操作或者打开 Dialog 都有可能在没达到你要的结果时进程就被系统杀掉.

在官方文件中有提到

> Once you return from onReceive(), the BroadcastReceiver is no longer active, and its hosting process is only as important as any other application components that are running in it.

所以可以使用 Notificaiton 或者 Service (当然不能用bindService) 来进行其他操作, 不让系统回收该进程.

## 使用广播来更新界面是否合适?

更新界面也分很多种情况, 如果不是频繁地刷新, 使用广播来做也是可以的.

但对于较频繁地刷新动作, 不推荐. 广播的发送和接收是有一定的代价的, 它的传输是通过 Binder 进程间通信机制来实现的, 那么系统定会为了广播能顺利传递做一些进程间通信的准备.

除此之外，还可能有其他的因素让广播发送和到达是不准时的, Android 的 ActivityManagerService 有一个专门的消息队列来接收发送出来的广播, sendBroadcast执行完后就立即返回, 但这时发送来的广播只是被放入到队列, 并不一定马上被处理. 当处理到当前广播时, 又会把这个广播分发给注册的广播接收分发器 ReceiverDispatcher, ReceiverDispatcher 最后又把广播交给接 Receiver 所在的线程的消息队列去处理, 也就 UI 线程的 Message Queue. 整个过程从发送 -- AMS -- ReceiverDispatcher 进行了两次 Binder 进程间通信, 最后还要交到UI的消息队列, 如果基中有一个消息的处理阻塞了UI, 当然也会延迟你的onReceive的执行.

### onReceive 方法是运行主线程吗

是的, onReceive 方法运行在应用的主进程, 阻塞运行超过 5 秒就 ANR 了