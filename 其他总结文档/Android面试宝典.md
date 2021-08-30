### 一  Android各系统版本适配

​     Android 5.0 :

​				Material Design风格，各种炫酷的UI动画效果

​    Android 6.0 ：

​				运行时权限申请

​	Android 7.0 ：

​				1.系统禁止应用向其他应用传递file://开头的uri，需要使用FileProvider去适配

​                2.Doze模式，系统禁止处于后台一段时间得应用访问网络，已达到省电目的

   Android 8.0 ：

​				1.通知需要设置channel

​				2.未知来源的应用，安装时，要先允许安装才能进一步安装未知来源的应用

​				3.启动前台service，需要使用startForgroudService方法，并且5秒之内，需要调用	    startForgroud方法发送通知

  Android 9.0 ：

​				1.新出刘海屏，需要针对各大厂商提供的api进行适配

​                2.权限变更，部分权限组变更

​                3.前台service需要申请FOREGROUND_SERVICE权限

​                4.非SDK api限制

### 二  Android事件分发机制

![image-20190530223156293](/Users/zwb/Documents/面试宝典/Interview/1.png)

![image-20190530223322797](/Users/zwb/Documents/面试宝典/Interview/2.png)

​		事件触摸产生的事件依次往下传递 Activity—>ViewGroup—>View     子视图处理不了的，会依次往上抛，知道事件被消费或者无处理

​		对于View来说，执行顺序onTouchListener > onTouchEvent > onClickListener



### 三  View的绘制流程

​		1. 首先，view的绘制流程包含几个关键对象，ViewRootImpl   PhoneWindow,WindowManager,WindowManagerService,IWindowSession等

​		2. window是一个抽象的概念，具体关联的就是view，每个activity都会有一个window呈现出来，在ActivityThread执行handleLaunchActivity的时候，会创建activity对象，生成PhoneWindow和WindowManager，使他们关联起来。并且初始化WindowMnagerGloable，生成WindowMnagerService代理对象，类型为IWindowManager

​		3. 在Activity的onCreate方法中，执行setContentView方法，实际上调用的是phoneWindow的setContentView方法，里面会初始化DecorView对象，并使用pull解析器解析xml文件，把相应的view添加进入DecorView中

	    4. 在Activity中的handleResumeActivity方法中，会调用windowManager的addView方法，把之前设置好的DecorView添加进去，里面依次调用的是WindowMnagerGloable.addView——> 内部生成一个ViewRootImpl，然后使用它的setView方法，里面会执行它的requestLayout方法(里面会判断是否需要重新measure，是否需要重新layout，是否需要重新draw)，如果需要的话，会依次measure子类，layout所以子类，draw自身背景，dispatchDraw它的孩子，然后绘制滚动条之类的
   	    5. 完成这一步之后，会使用IWindowManager得到一个IWindowSession对象，通过他的addToDisplay方法，把在ViewRootImpl中生成的一个IWindow对象以及其他的属性传递给WindowMnagerService，由它去进行绘制，这是一次IPC过程。之后就是WindowMnagerService端做的事情了，生成SufaceFlinger，把视图刷新到FrameBuffer当中，这样用户就可以看到界面的。
   	    6. 非UI线程一定不能刷新UI吗，不是这样的，刷新UI是在handleResumeActivity的时候，ViewRootImpl里面的requestLayout方法，里面有个checkThread动作，检查初始化ViewRootImpl时的线程与执行该方法的线程是否一致，所以，只要自己创建一个ViewRootImpl，保证生成与调用的方法在同一线程中就可以了。

### 四  Activity的启动流程

1. 几个关键的类 Instrumentation   ActivityManagerService    ActivityManagerProxy    IAcivityManager，ActivityThread   ApplicationThread    H
2. 桌面点击图标启动A应用的时候，会执行startSafely方法，里面最终会执行startActivityForResult方法，内部通过Instrumentation执行execStartActivity，然后里面会通过ActivityManager.getService()获取ActivityManagerService代理对象，类型为IAcivityManager，实际为ActivityManagerProxy，然后使用IAcivityManager去启动目标界面，并把当前app的ApplicationThread，mToken（IBinder）传递过去，分别代表当前应用的进程，当前Activity。在服务端会被封装成一个ActivItyRecord对象，然后内部会先检测目标Activity是否存在，不存在就直接报错。接着利用ActivItyRecord里面的ApplicaoinThreadProxy，反向通知桌面app，你可以休息了，实际的ApplicationThread会发生一个PAUSE_ACTIVITY的消息给H，由H通知ActivityThread调用handlePause。。当桌面已经pause之后，通过IAcivityManager告诉AMS，我已经睡了。接着AMS与目标界面通讯
3. AMS检查目标界面的进程是否启动，未启动的话启动一个新的进程，并执行ActivityThread中的main方法，里面生成ActivityThread对象，并执行Looper初始化，调用IAcivityManager把生成的ApplicationThread对象传进去，AMS这边会通过代理对象调用ApplicationThread的bingApplication方法，创建application，context，并attach使他们关联。完成之后，调用scheduleLaunchActivity,之后发生消息给H，执行ActivityThread中的launchActivity的方法,创建context，Instrumentation反射生成Activity，实例化PhoneWindow，然后使他们关联起来，接着调用handleResumeActivity的方法，使界面展示出来。

### 五  Binder机制

1. binder是android端进程间通信的方法，它继承自IBinder接口
2. binder是一种Client—Server模型，中间有一个ServiceManager，相当于一个注册表，保存Server端各binder实例的引用，通过一个简短的字符串去查找
3. ServiceManager的初始化：它是在Android系统init进程初始化的时候执行注册的，ServiceManager内部保持着一个为0的binder实例，然后他会通过该binder实例，把自己注册成服务BINDER_SET_CONTEXT_MGR，之后client端就可以通过这个为0的binder找到ServiceManager，然后再通过ServiceManager找到其他注册过的server
4. 其他的server也是通过这个为0的binder，生成自己的binder实例，并传递给serviceMnager去注册，之后client申请查找server的时候，ServiceManager会把一个引用返回给client
5. 关于一次拷贝。Android系统空间分为 "用户空间" "内核空间"  其中"用户空间"是非共享的，"内核空间"是共享的，普通的IPC方式(socket/管道)等需要经历从"用户空间"—"内核空间"—"用户空间" 两次数据拷贝的过程。但是binder机制对于内存管理有一个 内存映射的能力，通过mmap()实现，他可以把"内核空间"的一部分内存与"用户空间"的一部分内存映射起来，这样，修改了"内核空间"的这部分内容，也会体现到"用户空间"，反之亦然，这样就实现了一次拷贝，效率更高
6. 安全性：Android系统为每个进程设置了UID来标志这个进程。在使用binder服务的时候，在transact()等函数时，里面会做UID的权限检查，检查这个请求是善意的才能通过放行

### 六  Handler机制

1. 与Handler密切相关的还有Message、MessageQueue、Looper。

2. Message。Message有两个关键的成员变量：target、callback：

​        (1) target。就是发送消息的Handler

​        (2) callback。调用Handler.post(Runnable)时传入的Runnable类型的任务。post事件的本质也是创建了一个Message，将我们传入的这个runnable赋值给创建的Message的callback这个成员变量。

3. MessageQueue。消息队列用于存放消息（本质上并不是一个消息队列的数据结构），其中重点关注next()方法，它会返回下一个待处理的消息。

4. Looper。Looper消息轮询器其实是连接Handler和消息队列的核心。想要在一个线程中创建一个Handler，首先要通过Looper.prepare()创建Looper，之后还得调用Looper.loop()开启轮询。生成一个Looper之后，会使用ThreadLocal保存在线程本地

​       (1) prepare()。这个方法做了两件事：首先通过ThreadLocal.get()获取当前线程中的Looper，如果不为空则抛出RuntimeException。否则创建Looper，并通过ThreadLocal.set(looper)将当前线程与刚刚创建的Looper绑定。值得注意的是，上面的消息队列的创建其实就是发生在Looper的构造函数中。

​      (2) loop()。这个方法开启了整个事件机制的轮询。其本质是开启一个死循环，不断地通过MessageQueue的next()方法获取消息msg。拿到消息后会调用msg.target.dispatchMessage()来做处理。综上也就是调用handler.dispatchMessage()。

5. Handler。Handler重点在于发送消息和处理消息。

​      (1) 发送消息。其实发送消息除了 sendMessage 之外还有 sendMessageDelayed 和 post 以及 postDelayed 等等不同的方式。但它们的本质都是调用了 sendMessageAtTime。在 sendMessageAtTime 这个方法中调用了 enqueueMessage。在 enqueueMessage 这个方法中做了两件事：通过 msg.target = this 实现了消息与当前 handler 的绑定。然后通过 queue.enqueueMessage 实现了消息入队。实质上是每个消息都持有下一个消息的引用，此时会判断当前消息的发送时间，然后把message插入到不同的位置

​     (2) 处理消息。 消息处理的核心其实就是dispatchMessage()这个方法。这个方法里面的逻辑很简单，先判断 msg.callback 是否为 null，如果不为空则执行这个 runnable。如果为空则会判断handler的callback是否为空，不为空执行它的handleMessage方法，否则执行我们handler的handleMessage方法。

### 七  自定义view的流程

1. 自定义view继承现有的布局，如RelativeLayout,在其构造函数里面添加xml文件进行组合布局

2. 自定义view继承现有的布局或者ViewGroup/View,重写里面的onMeasure，onLayout(ViewGroup中才有)，onDraw方法

   (1) onMeasure：涉及到MeasureSpec测量规格，由32数组成，💰两位代表的是测量模式，后30位代表的大小，测量模式有EXACTLY(精确的大小，一般最后就是这么大)，AT_MOST(不能操作父布局大小)UNSPECIFIED(无限制，一般是系统使用，无意义)

![674980-b6f05d7b681ca9ad](/Users/zwb/Documents/面试宝典/Interview/3.jpeg)

3. onLayout 摆放child的位置

4. onDraw 执行真正的绘制工作

   ![966283-480bf9def58bed74](/Users/zwb/Documents/面试宝典/Interview/4.png)



​	5. 除此之外，可以在resource文件里面自定义控件的属性，在xml中引用，然后在自定义控件中读取出来

### 八  线程管理

1.  Android中的线程分UI线程，子线程，其中他们的通信方式是使用Handler机制，因为每开启一个线程，都会耗费一定的资源，那么为了避免这部分资源的消耗，创建线程的方式就要被管理起来，主要是用到线程池。好处：通过线程池中线程的重用，减少创建和销毁线程的性能开销。其次，能控制线程池中的并发数，否则会因为大量的线程争夺CPU资源造成阻塞。

2. Android中的线程池使用Executor来实现，真正的实现类是ThreadPoolExecutor，他只有一个方法execute()，该方法用来执行线程中的操作。

   (1) newFixedThreadPool  创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程。特点：只有核心线程数，并且没有超时机制，因此核心线程即使闲置时，也不会被回收，因此能更快的响应外界的请求.

   (2) newCachedThreadPool 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程  特点：没有核心线程，非核心线程数量没有限制， 超时为60秒.

   (3) newScheduledThreadPool 创建一个定长线程池，支持定时及周期性任务执行 特点：核心线程数是固定的，非核心线程数量没有限制， 没有超时机制.主要用于执行定时任务和具有固定周期的重复任务.

   (4) SingleThreadExecutor 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行  特点：只有一个核心线程，并没有超时机制.意义在于统一所有的外界任务到一个线程中， 这使得在这些任务之间不需要处理线程同步的问题.

3. **CorePoolSize** 
   线程的核心线程数。

   默认情况下，核心线程数会在线程中一直存活，即使它们处于闲置状态。

   **maximumPoolSize** 
   线程池所能容纳的最大线程数。

   当活动线程数达到这个数值后，后续的新任务将会被阻塞。

   **keepAliveTime** 
   非核心线程闲置时的超时时长，超过这个时长，非核心线程就会被回收，当ThreadPoolExector的allowCoreThreadTimeOut属性设置为True时，keepAliveTime同样会作用于核心线程。

   **unit** 
   用于指定keepAliveTime参数的时间单位，这是一个枚举，常用的有TimeUnit.MILLISECONDS（毫秒）、TimeUnit.SECONDS(秒)以及TimeUnit.MINUTES(分钟)等。

   **workQueue** 
   线程池中的任务队列，通过线程池execute方法提交的Runnable对象会存储在这个参数中。

   **threadFactory** 
   线程工厂，为线程池提供创建新线程的功能。ThreadFactory是一个接口，它只有一个方法，newThread（Runnable r），用来创建线程。

   

   （以下用currentSize表示线程池中当前线程数量）

   （1）当currentSize<corePoolSize时，没什么好说的，直接启动一个核心线程并执行任务。

   （2）当currentSize>=corePoolSize、并且workQueue未满时，添加进来的任务会被安排到workQueue中等待执行。

   （3）当workQueue已满，但是currentSize<maximumPoolSize时，会立即开启一个非核心线程来执行任务。

   （4）当currentSize>=corePoolSize、workQueue已满、并且currentSize>maximumPoolSize时，调用handler默认抛出RejectExecutionExpection异常。

### 九  设计模式

​    https://www.jianshu.com/p/7fa31c57cbb5

#####     单一职责：一个类应只包含单一的职责。

​         （1）一个类职责过大的话，首先引起的问题就是这个类比较大，显得过于臃肿，    同时其复用性是比较差的。

​         （2）其次就是如果修改某个职责，有可能引起另一个职责发生错误。这是我们极力所避免的，因此设计一个类时我们应当去遵循单一职责原则。

#####     开闭原则：对修改封闭，对扩展放开。

​          （1）修改原有的代码可能会导致原本正常的功能出现问题。

​          （2）因此，当需求有变化时，最好是通过扩展来实现，增加新的方法满足需求，而不是去修改原有代码。

#####     里氏替换原则：使用父类的地方能够使用子类来替换，反过来，则不行。

​          （1）使用子类对象去替换父类对象，程序将不会产生错误

​          （2）因此在程序中尽量使用基类类型来对对象进行定义，而在运行时再确定其子类类型，用子类对象来替换父类对象。

#####     依赖倒置原则：抽象不应该依赖细节，细节应该依赖抽象。

​          （1）即要面向接口编程，而不是面向具体实现去编程。

​          （2）高层模块不应该依赖低层模块，应该去依赖抽象。

#####     接口隔离原则：一个类对另一个类的依赖应该建立在最小的接口上。

​          （1）一个类不应该依赖他不需要的接口。

​          （2）接口的粒度要尽可能小，如果一个接口的方法过多，可以拆成多个接口。

#####     迪米特原则：也称为最少知识原则，一个对象应该对另一个对象有最少的了解。

​		（1）一个类对其他类知道的越少，耦合越小。

​        （2）当修改一个类时，其他类的影响就越小，发生错误的可能性就越小。

1. 单例模式：保证在程序整个生命周期只有一个对象生成

2. 观察者模式：定义对象间的一种一对多的依赖关系，当一个对象的状态发送改变时，所有依赖于它的对象都能得到通知并被自动更新。Android中的监听器可以认为是一对一的观察者模式。

3. Builder模式：将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。Notification的创建是通过buidler来执行。

4. 策略模式：策略模式定义了一些列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变换。

5. 状态模式：当一个对象的内在状态改变时允许改变其行为，这个对象看起来像是改变了其类。

6. 命令模式：将一个请求封装成一个对象，从而让你使用不同的请求把客户端参数化，对请求排队或者记录日志，可以提供命令的撤销和恢复功能。

   (策略模式，状态模式，命令模式，看起来非常类似 。   他们的不同其实是从不同的场景来看的，本质都差不多 )

7. 适配器模式：将一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够在一起工作。

8. 责任链模式：一个请求沿着一条“链”传递，直到该“链”上的某个处理者处理它为止。每个处理者都持有下一个处理者的引用，这就构成了链这种结构。典型的：击鼓传花

9. 桥接模式：将抽象部分与实现部分分离，使它们都可以独立的变化。Android里面体现的有WindowManager与WindowManagerGloable

### 十  ANR出现的原因？怎么解决ANR问题？

​      ANR(Application Not responding)。Android中，主线程(UI线程)如果在规定时内没有处理完相应工作，就会出现ANR。具体来说，ANR会在以下几种情况中出现:

1. 输入事件(按键和触摸事件)5s内没被处理

2. BroadcastReceiver的事件(onRecieve方法)在规定时间内没处理完(前台广播为10s，后台广播为60s)

3. service 前台20s后台200s未完成启动

4. ContentProvider的publish在10s内没进行完

分析ANR问题，需要结合Log以及trace文件。具体分析流程，可参照以下两篇文章：

https://www.jianshu.com/p/fa962a5fd939

https://blog.csdn.net/droyon/article/details/51099826



### 十一  内存泄露的场景？内存泄露如何去分析？

​        Java的内存泄漏就是对象不再使用的时候，无法被JVM回收。内存泄漏最终会引发Out Of Memory。

**常见的内存泄露有：**

​	•	单例模式引起的内存泄露。

​	•	静态变量导致的内存泄露。

​	•	非静态内部类引起的内存泄露。

​	•	使用资源时，未及时关闭引起内存泄露。

​	•	使用属性动画引起的内存泄露。

​	•	Webview导致的内存泄露。

而对于内存泄露的检测，常用的工具有LeakCanary、MAT（Memory Analyer Tools）、Android Studio自带的Profiler。关于用法，网上教程很多，可自行查阅。

解决webView带来的泄露问题：

1.原因：webView持有activity的引用而得不到释放，在AwContents中会执行registerComponentCallbacks,但是在webView.destory的时候，并不会反注册(旧的版本可以)，原因在于反注册在onDetchFromWindow的时候，里面会判断如果isDesotryed，就return，而webView.destory会使其为true。

2.解决办法：

​	(1). 开启新的进程，结束是退出整个进程

​	(2). 在destory之前，先获取其parent，然后remove它自己，使onDetchFromWindow先执行，然后再掉destory方法。



### 十二  Android性能优化

https://www.jianshu.com/p/3e44250ca2de

  主要分为以下几个方面：

1. ANR ：参考十
2. 内存溢出：OOM
3. 内存抖动：短时间内触发分配回收GC操作，循环中创建临时对象，onDraw中创建对象
4. 内存泄露：不在使用的对象无法及时被回收。bitmap，cursor，activity，handler
5. UI卡顿，布局优化，绘制优化：减少布局层级，include，merge，viewsuble
6. 线程优化，避免频繁创建销毁线程，使用线程池
7. 启动优化：冷启动  热启动  温启动

### 十三  如何实现启动优化，有什么工具可以使用？

   启动优化分为两个方面

1. 显示优化：设置背景，避免启动进来的时候显示为白屏或者黑屏
2. 程序优化：在初始化的时候，避免执行过重的操作，可以延后执行的放到子线程去初始化

重点提到了systrace这个工具，详细用法可以参考下面几篇文章：

https://blog.csdn.net/Kitty_Landon/article/details/79192377

https://www.cnblogs.com/baiqiantao/p/7700511.html

https://blog.csdn.net/xiyangyang8/article/details/50545707

https://blog.csdn.net/cxq234843654/article/details/74388328



adb shell am start -S -W 包名/启动类的全限定名 计算启动时间



adb shell am start -S -W com.ucloudlink.simbox/com.ucloudlink.simbox.view.activity.SplashActivity



Stopping: com.ucloudlink.simbox

Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.ucloudlink.simbox/.view.activity.SplashActivity }

Status: ok

Activity: com.ucloudlink.simbox/.view.activity.LoginActivity

ThisTime: 979

TotalTime: 178438

WaitTime: 3611

Complete



ThisTime : 最后一个 Activity 的启动耗时(例如从 LaunchActivity - >MainActivity「adb命令输入的Activity」 , 只统计 MainActivity 的启动耗时)

TotalTime : 启动一连串的 Activity 总耗时.(有几个Activity 就统计几个)

WaitTime : 应用进程的创建过程 + TotalTime .



最后总结一下 ： 如果需要统计从点击桌面图标到 Activity 启动完毕，可以用WaitTime作为标准，但是系统的启动时间优化不了，所以优化冷启动我们只要在意 **ThisTime 即可。**



过滤在控制台过滤displayed也可以看到thisTime



systrace的用法：

进入sdk下的platform-tools/systrace/，可以看到systrace实际上是一个python脚本，也就是说使用命令行需要安装python环境

使用命令行的一般格式是

python systrace.py -option



其中option可以使用

python systrace.py --help



来获得，一般来说我们只需要使用 -o ，-b， -t，-a 对应输出文件路径，buffersize，time，需要分析的应用程序包名

获取可用的tag，输入

python systrace.py -l

会提示当前可以用的tag，也会包含具体的释义

举个例子

python systrace.py -t 3 -o ~/mytrace.html -a com.android.test gfx view wm am res sync



### 十四  Android进程保活

1. 双进程守护，在5.0以上的版本无效，因为5.0以上会直接把应用相关的进程全部杀掉
2. JobSchedule，定期执行检查任务，在5.0，5.1，6.0有效，之上无效，因为后面有省电相关的处理
3. 前台service，提升应用的级别，使应用存活更久
4. 使用系统推送，把应用进程拉起来，部分手机需要开启自启动
5. 自启动，白名单
6. i

### 十五  图片优化

1. 使用质量压缩：保持图片像素不变的情况下，改变图片的透明度及位深度，无法改变加载在内存bitmap的大小，一般用来存储文件，使文件变小，然后上传到服务器
2. 尺寸压缩：改变加载的图片的像素大小，可以做到内存减小
3. 采样率压缩：改变采样率，可以减小加载到内存中的图片大小
4. 遵循一个原则：大图变小用压缩，小图变大用矩阵Matrix



### 十六  Broadcast的分类？有序，无序？粘性，非粘性？本地广播？



注：广播分为静态注册和动态注册，静态注册是在AndroidManifest.xml文件中配置；动态注册是在运行的Activity中声明广播对象并注册（动态注册时必须解注册，否则会奔溃）。其中Android8.0在AndroidManifest.xml文件中静态注册广播接收失效是由于官方对耗电量的优化，避免APP滥用广播的一种处理方式。除了少部分的广播仍支持静态注册（如开机广播），其余的都会出现失效的情况。

##### **广播分为四类：有序广播、无序广播、本地广播、粘性广播。**

1. 无序广播：无序广播是一种完全异步的广播，通过context的sendBroadcast方式进行发送，消息发送的速度很快，但是消息是无序的。

在没有一个匹配的广播接收者接到之前，没办法停止intent的传递。

2. 有序广播：有序广播通过sendOrderedBroadcast来发送，广播接收器接收的消息按照优先级依次执行。针对广播接收者监听的每一个行为的优先级可以在清单文件注册广播时，同时添加监听行为的优先级—>intent-filter中的android：priority属性进行设置（-1000—1000）。数值越大优先级越高。在广播接收者接收到广播后，可以使用 setResultData（）方法把处理的结果发送给下一个广播，下一个广播通过getResultData（）拿到上一个广播发送的结果。并可以使用abortBroadcast函数来让系统丢弃该广播，使当前行为的广播不在传递。



3. 本地广播：本地广播是通过LocalBroadcastManager内置的Handler来实现的，只是利用了IntentFilter的match功能，至于BroadcastReceiver 换成其他接口也无所谓，顺便利用了现成的类和概念而已。在register()的时候保存BroadcastReceiver以及对应的IntentFilter，在sendBroadcast()的时候找到和Intent对应的BroadcastReceiver，然后通过Handler发送消息，触发executePendingBroadcasts()函数，再在后者中调用对应BroadcastReceiver的onReceive()方法。LocalBroadcastManager.getInstance(context)中对应的发送，解绑，注册方法



4. 粘性广播：

   （1）.Sticky广播是会滞留的广播，通过context.sendStickyBroadcast()方法进行发送，用此方法发送的广播会滞留一段时间，直到有对应的广播接收者注册时，该广播接收者就会收到该广播。

   （2）.需要用到 android.permission.BROADCAST_STICKY 权限

   （3）.sendStickyBroadcast只会保留最后一条广播，并且一直保存下去，这样即使有了广播接收者接收到了广播，当再有广播接收者被注册时仍然会接收到该广播。如果只想处理一遍该广播可以使用removeStickyBroadcast来处理。粘性广播的Receiver如果被销毁，那么下次重建时会自动接收到消息数据。(在 android 5.0/api 21中deprecated,不再推荐使用，相应的还有粘性有序广播，同样已经deprecated)



### 十七  数据库如何进行升级？SQLite增删改查的基础sql语句?

在SQLiteOpenHelper的构造函数中，包含了一个version的参数。这个参数即是数据库的版本。 所以，我们可以通过修改version来实现数据库的升级。 当version大于原数据库版本时，onUpgrade()会被触发，可以在该方法中编写数据库升级逻辑。降级的话是在onDownGrade()方法中被触发



常用的SQL增删改查：

​	•	增：INSERT INTO table_name (列1, 列2,…) VALUES (值1, 值2,….)

​	•	删：DELETE FROM 表名称 WHERE 列名称 = 值

​	•	改：UPDATE 表名称 SET 列名称 = 新值 WHERE 列名称 = 某值

​	•	查：SELECT 列名称（通配是*符号） FROM 表名称

ps:操作数据表是:ALTER TABLE。该语句用于在已有的表中添加、修改或删除列。

ALTER TABLE table_name ADD column_name datatype

ALTER TABLE table_name DROP COLUMN column_name

ALTER TABLE table_name_old RENAME TO table_name_new



### 十八  死锁是怎么导致的？如何定位死锁

某个任务在等待另一个任务，而后者又等待别的任务，这样一直下去，直到这个链条上的任务又在等待第一个任务释放锁。这得到了一个任务之间互相等待的连续循环，没有哪个线程能继续。这被称之为死锁。当以下四个条件同时满足时，就会产生死锁：

​	(1) 互斥条件。任务所使用的资源中至少有一个是不能共享的。

​	(2) 任务必须持有一个资源，同时等待获取另一个被别的任务占有的资源。

​	(3) 资源不能被强占。

​	(4) 必须有循环等待。一个任务正在等待另一个任务所持有的资源，后者又在等待别的任务所持有的资源，这样一直下去，直到有一个任务在等待第一个任务所持有的资源，使得大家都被锁住。

要解决死锁问题，必须打破上面四个条件的其中之一。在程序中，最容易打破的往往是第四个条件。

定位死锁最常用的工具就是利用jstack等工具获取线程栈，然后定位相互之间的依赖关系，进而找到死锁。



### 十九  final关键字的用法？

final 可以修饰类、变量和方法。修饰类代表这个类不可被继承。修饰变量代表此变量不可被改变。修饰方法表示此方法不可被重写 (override)。



### 二十  如何理解Java的多态？其中，重载和重写有什么区别？

**java中有三大特性，封装，继承和多态，其中封装和继承都是为多态做准备的。**



**封装**

封装就是将类的信息隐藏在类内部，不允许外部程序直接访问，而是通过该类的方法实现对隐藏信息的操作和访问。



**继承**

继承是类与类的一种关系，比较像集合中的从属于关系。比如说，狗属于动物。就可以看成狗类继承了动物类，那么狗类就是动物类的子类（派生类），动物类就是狗类的父类（基类）。在Java中是单继承的，也就是说一个子类只有一个父类。



**多态**

1.多态是同一个行为具有多个不同表现形式或形态的能力。多态就是同一个接口，使用不同的实例而执行不同操作，多态就是程序运行期间才确定，一个引用变量倒底会指向哪个类的实例对象，该引用变量发出的方法调用到底是哪个类中实现的方法。

2.多态存在的三个必要条件是：继承，重写，父类引用指向子类引用。

3.多态的三个实现方式是：重写，接口，抽象类和抽象方法。



**重载与重写**

![5](/Users/zwb/Documents/面试宝典/Interview/5.tiff)





### 二十一   谈一下JVM内存区域划分？哪部分是线程公有的，哪部分是私有的？



![6](/Users/zwb/Documents/面试宝典/Interview/6.tiff)

JVM 的内存区域可以分为两类：线程私有区域和线程共有的区域。 线程私有的区域：程序计数器、JVM 虚拟机栈、本地方法栈；线程共有的区域：堆、方法区、运行时常量池。

程序计数器，也有称作PC寄存器。每个线程都有一个私有的程序计数器，任何时间一个线程都只会有一个方法正在执行，也就是所谓的当前方法。程序计数器存放的就是这个当前方法的JVM指令地址。当CPU需要执行指令时，需要从程序计数器中得到当前需要执行的指令所在存储单元的地址，然后根据得到的地址获取到指令，在得到指令之后，程序计数器便自动加1或者根据转移指针得到下一条指令的地址，如此循环，直至执行完所有的指令。

JVM虚拟机栈。创建线程的时候会创建线程内的虚拟机栈，栈中存放着一个个的栈帧，对应着一个个方法的调用。JVM 虚拟机栈有两种操作，分别是压栈和出栈。栈帧中存放着局部变量表(Local Variables)、操作数栈(Operand Stack)、指向当前方法所属的类的运行时常量池的引用(Reference to runtime constant pool)、方法返回地址(Return Address)和一些额外的附加信息。

本地方法栈。本地方法栈与Java栈的作用和原理非常相似。区别只不过是Java栈是为执行Java方法服务的，而本地方法栈则是为执行本地方法（Native Method）服务的。在JVM规范中，并没有对本地方发展的具体实现方法以及数据结构作强制规定，虚拟机可以自由实现它。在HotSopt虚拟机中直接就把本地方法栈和Java栈合二为一。

堆。堆是内存管理的核心区域，用来存放对象实例。几乎所有创建的对象实例都会直接分配到堆上。所以堆也是垃圾回收的主要区域，垃圾收集器会对堆有着更细的划分，最常见的就是把堆划分为新生代和老年代。java堆允许处于不连续的物理内存空间中，只要逻辑连续即可。堆中如果没有空间完成实例分配无法扩展时将会抛出OutOfMemoryError异常。

方法区。方法区与堆一样所有线程所共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、及时编译器编译后的代码等数据。在Class文件中除了类的字段、方法、接口等描述信息外，还有一项信息是常量池，用来存储编译期间生成的字面量和符号引用。

其实除了程序计数器，其他的部分都会发生 OOM。

堆。 通常发生的 OOM 都会发生在堆中，最常见的可能导致 OOM 的原因就是内存泄漏。

JVM虚拟机栈和本地方法栈。 当我们写一个递归方法，这个递归方法没有循环终止条件，最终会导致  StackOverflow 的错误。当然，如果栈空间扩展失败，也是会发生 OOM 的。

方法区。方法区现在基本上不太会发生 OOM，但在早期内存中加载的类信息过多的情况下也是会发生 OOM 的。



### 二十二  TCP/IP

​	•	TCP（传输控制协议）是一种面向连接的通过失败重传机制确保数据在端到端之间可靠传输的协议。

​	•	IP是面向无连接无状态的么有额外的控制机制保证发送的包是否有序到达。

​	•	五层模型：应用层、传输层、网络层、链路层、物理层

​	•	总结一下：程序在发送消息时，应用层按既定的协议打包数据，随后由数据层加上双方端口号，网络层加上双方IP地址，链路层加上双方MAC地址，并且将数据拆分成数据帧，经过多个路由器和网关后到达目的机器。简而言之，就是按“端口--IP地址--MAC地址”这样的路径进行数据的封装和发送。

​	▪	三次握手：SYN和ACK置0和1，seq 和 ack [序号和应答号] x和y

​	▪	你听得见吗？

​	▪	我听得见，你听得见吗？

​	▪	我也听得见，我们说话吧。

​	•	四次握手：

​	▪	我们分手吧

​	▪	好等我收拾东西，收拾完我告诉你

​	▪	我收拾完了

​	▪	好再见

HTTPS

HTTPS在传输的过程中会涉及到三个密钥：

服务器端的公钥和私钥，用来进行非对称加密

客户端生成的随机密钥，用来进行对称加密

一个HTTPS请求实际上包含了两次HTTP传输，可以细分为8步。

1.客户端向服务器发起HTTPS请求，连接到服务器的443端口(客户端给出协议版本号、一个客户端随机数A（Client random）以及客户端支持的加密方式

)

2.服务器端有一个密钥对，即公钥和私钥，是用来进行非对称加密使用的，服务器端保存着私钥，不能将其泄露，公钥可以发送给任何人。(服务端确认双方使用的加密方式，并给出数字证书、一个服务器生成的随机数B（Server random）)

3.服务器将自己的公钥发送给客户端。

4.客户端收到服务器端的公钥之后，会对公钥进行检查，验证其合法性，如果发现发现公钥有问题，那么HTTPS传输就无法继续。严格的说，这里应该是验证服务器发送的数字证书的合法性，关于客户端如何验证数字证书的合法性，下文会进行说明。如果公钥合格，那么客户端会生成一个随机值，这个随机值就是用于进行对称加密的密钥，我们将该密钥称之为client key，即客户端密钥，这样在概念上和服务器端的密钥容易进行区分。然后用服务器的公钥对客户端密钥进行非对称加密，这样客户端密钥就变成密文了，至此，HTTPS中的第一次HTTP请求结束。

5.客户端会发起HTTPS中的第二个HTTP请求，将加密之后的客户端密钥发送给服务器。

6.服务器接收到客户端发来的密文之后，会用自己的私钥对其进行非对称解密，解密之后的明文就是客户端密钥，然后用客户端密钥对数据进行对称加密，这样数据就变成了密文。

7.然后服务器将加密后的密文发送给客户端。

8.客户端收到服务器发送来的密文，用客户端密钥对其进行对称解密，得到服务器发送的数据。这样HTTPS中的第二个HTTP请求结束，整个HTTPS传输完成。

![7](/Users/zwb/Documents/面试宝典/Interview/7.tiff)

![8](/Users/zwb/Documents/面试宝典/Interview/8.tiff)