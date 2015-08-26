# WJRunLoopSummary
#RunLoop个人小结

学习iOS开发一般都是从UI开始的，从只知道从IB拖控件，到知道怎么在方法里写代码，然后会显示什么样的视图，产生什么样的事件，等等。其实程序从启动开始，一直都是按照苹果封装好的代码运行着，暴露的一些属性和方法作为接口，是让我们在给定的方法里写代码实现自定义功能，做出各种各样的应用。这些方法的调用顺序最为关键，熟悉了程序运转和方法调用的顺序，才可以更好地操控程序和代码，尽量避免Xcode不报错又实现不了功能的BUG。从Xcode的线程函数调用栈可以看到一些方法调用顺序。

##从程序启动开始到view显示：
start->(加载framework，动态静态链接库，启动图片，Info.plist，pch等)->main函数->UIApplicationMain函数：

- 初始化UIApplication单例对象
  - 初始化AppDelegate对象，并设为UIApplication对象的代理
  - 检查Info.plist设置的xib文件是否有效，如果有则解冻Nib文件并设置outlets，创建显示key window、rootViewController、与rootViewController关联的根view（没有关联则看rootViewController同名的xib），否则launch之后由程序员手动加载。
  - 建立一个主事件循环，其中包含UIApplication的Runloop来开始处理事件。

###UIApplication：
1、通过window管理视图；
2、发送Runloop封装好的control消息给target；
3、处理URL，应用图标警告，联网状态，状态栏，远程事件等。
###AppDelegate：
管理UIApplication生命周期和应用的五种状态(notRunning/inactive/active/background/suspend)。
###Key Window：
1、显示view；
2、管理rootViewcontroller生命周期；
3、发送UIApplication传来的事件消息给view。
###rootViewController：
1、管理view（view生命周期；view的数据源/代理；view与superView之间事件响应nextResponder的“备胎”）;
2、界面跳转与传值；
3、状态栏，屏幕旋转。
###view：
1、通过作为CALayer的代理，管理layer的渲染（顺序大概是先更新约束，再layout再display）和动画（默认layer的属性可动画，view默认禁止，在UIView的block分类方法里才打开动画）。layer是RGBA纹理，通过和mask位图（含alpha属性）关联将合成后的layer纹理填充在像素点内，GPU每1/60秒将计算出的纹理display在像素点中。
2、布局子控件（屏幕旋转或者子视图布局变动时，view会重新布局）。
3、事件响应：event和guesture。
插播控制器生命周期
###runloop:
1、（要让马儿跑）通过do-while死循环让程序持续运行：接收用户输入，调度处理事件时间。
2、（要让马儿少吃草）通过mach_msg()让runloop没事时进入trap状态，节省CPU资源。

关于程序启动原理以及各个控件的资料，已经有太多资料介绍，平时我们也经常接触经常用到，但关于Runloop的资料，官方文档总是太过简练，网上资源说法也不太统一，只能从CFRunLoopRef开源代码着手，试着学习总结下。（NSRunloop是对CFRunloopRef的面向对象封装，但是不是线程安全）。

##Runloop

###1、与线程和自动释放池相关：
Runloop的寄生于线程：一个线程只能有唯一对应的runloop；但这个根runloop里可以嵌套子runloops；
自动释放池寄生于Runloop：程序启动后，主线程注册了两个Observer监听runloop的进出与睡觉。一个最高优先级OB监测Entry状态；一个最低优先级OB监听BeforeWaiting状态和Exit状态。
线程(创建)-->runloop将进入-->最高优先级OB创建释放池-->runloop将睡-->最低优先级OB销毁旧池创建新池-->runloop将退出-->最低优先级OB销毁新池-->线程(销毁)

###2、CFRunLoopRef构造：
####数据结构：

// runloop数据结构
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode名字, 
    CFMutableSetRef _sources0;    // Set<CFRunLoopSourceRef>
    CFMutableSetRef _sources1;    // Set<CFRunLoopSourceRef>
    CFMutableArrayRef _observers; // Array<CFRunLoopObserverRef>
    CFMutableArrayRef _timers;    // Array<CFRunLoopTimerRef>
    ...
};
// mode数据结构
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set<CFStringRef>
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set<CFRunLoopModeRef>
    ...
};

####创建与退出：mode切换和item依赖

a 主线程的runloop自动创建，子线程的runloop默认不创建（在子线程中调用NSRunLoop *runloop = [NSRunLoop currentRunLoop];
获取RunLoop对象的时候，就会创建RunLoop）；

b runloop退出的条件：app退出；线程关闭；设置最大时间到期；modeItem为空；

c 同一时间一个runloop只能在一个mode，切换mode只能退出runloop，再重进指定mode（隔离modeItems使之互不干扰）；

d 一个item可以加到不同mode；一个mode被标记到commonModes里（这样runloop不用切换mode）。

####启动Runloop：

// 用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}

// 用指定的Mode启动，允许设置RunLoop最大时间（假无限循环），执行完毕是否退出
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}

###CFRunLoopModeRef：
####数据结构（见上）；
####创建添加：runloop自动创建对应的mode；mode只能添加不能删除

// 添加mode
CFRunLoopAddCommonMode(CFRunLoopRef runloop, CFStringRef modeName);

####类型：

1. kCFRunLoopDefaultMode: 默认 mode，通常主线程在这个 Mode 下运行。

2. UITrackingRunLoopMode: 追踪mode，保证Scrollview滑动顺畅不受其他 mode 影响。

3. UIInitializationRunLoopMode: 启动程序后的过渡mode，启动完成后就不再使用。

4: GSEventReceiveRunLoopMode: Graphic相关事件的mode，通常用不到。

5: kCFRunLoopCommonModes: 占位mode，作为标记DefaultMode和CommonMode用。


####modeItems：

// 添加移除item的函数（参数：添加/移除哪个item到哪个runloop的哪个mode下）
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);

CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);

CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);

CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);

CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);

CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);


###A-- CFRunLoopSourceRef：事件来源

按照官方文档CFRunLoopSourceRef为3类，但数据结构只有两类（？？？）
Port-Based Sources:与内核端口相关
Custom Input Sources:与自定义source相关
Cocoa Perform Selector Sources:与PerformSEL方法相关）

####数据结构（source0/source1）：

// source0 (manual): order(优先级)，callout(回调函数)
CFRunLoopSource {order =..., {callout =... }}

// source1 (mach port)：order(优先级)，port:(端口), callout(回调函数)
CFRunLoopSource {order = ..., {port = ..., callout =...}

source0：event事件，只含有回调，需要标记待处理（signal），然后手动将runloop唤醒（wakeup）；
source1 ：包含一个 mach_port 和一个回调，被用于通过内核和其他线程发送的消息，能主动唤醒runloop。

###B-- CFRunLoopTimerRef：系统内“定时闹钟”

NSTimer和performSEL方法实际上是对CFRunloopTimerRef的封装；runloop启动时设置的最大超时时间实际上是GCD的dispatch_source_t类型。

####数据结构:

// Timer：interval:(闹钟间隔), tolerance:(延期时间容忍度)，callout(回调函数)
CFRunLoopTimer {firing =..., interval = ...,tolerance = ...,next fire date = ...,callout = ...}

####创建与生效；

//NSTimer：
  // 创建一个定时器（需要手动加到runloop的mode中）
 + (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;

  // 默认已经添加到主线程的runLoop的DefaultMode中 
 + (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;


// performSEL方法
// 内部会创建一个Timer到当前线程的runloop中（如果当前线程没runloop则方法无效；performSelector:onThread: 方法放到指定线程runloop中）
- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay

####相关类型（GCD的timer与CADisplayLink)
GCD的timer：
dispatch_source_t 类型，可以精确的参数，不用以来runloop和mode，性能消耗更小。

dispatch_source_set_timer(dispatch_source_t source, // 定时器对象
                              dispatch_time_t start, // 定时器开始执行的时间
                              uint64_t interval, // 定时器的间隔时间
                              uint64_t leeway // 定时器的精度
                              );
                              
CADisplayLink ：
Timer的tolerance表示最大延期时间，如果因为阻塞错过了这个时间精度，这个时间点的回调也会跳过去，不会延后执行。
CADisplayLink 是一个和屏幕刷新率一致的定时器，如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似，只是没有tolerance容忍时间），造成界面卡顿的感觉。

###C--CFRunLoopObserverRef：监听runloop状态，接收回调信息（常见于自动释放池创建销毁）
####数据结构:

// Observer：order(优先级)，ativity(监听状态)，callout(回调函数)
CFRunLoopObserver {order = ..., activities = ..., callout = ...}

####创建与添加；

// 第一个参数用于分配该observer对象的内存空间
// 第二个参数用以设置该observer监听什么状态
// 第三个参数用于标识该observer是在第一次进入run loop时执行还是每次进入run loop处理时均执行
// 第四个参数用于设置该observer的优先级,一般为0
// 第五个参数用于设置该observer的回调函数
// 第六个参数observer的运行状态   
CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
      // 执行代码
}

####监听的状态；

typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};


##3、Runloop内部逻辑：关键在两个判断点（是否睡觉，是否退出）
###代码实现：

// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {

    // 0.1 根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    // 0.2 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;

    // 1.1 通知 Observers: RunLoop 即将进入 loop。---（OB会创建释放池）
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);

    // 1.2 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {

        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {

            // 2.1 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            // 2.2 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            // 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);

            // 2.3 RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            // 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);

            // 2.4 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }

            // 3.1 如果没有待处理消息，通知 Observers: RunLoop 的线程即将进入休眠(sleep)。--- (OB会销毁释放池并建立新释放池)
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }

            // 3.2. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            // -  一个基于 port 的Source1 的事件。
            // -  一个 Timer 到时间了
            // -  RunLoop 启动时设置的最大超时时间到了
            // -  被手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }

            // 3.3. 被唤醒，通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);

            // 4.0 处理消息。
            handle_msg:

            // 4.1 如果消息是Timer类型，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 

            // 4.2 如果消息是dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 

            // 4.3 如果消息是Source1类型，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }

            // 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);


            // 5.1 如果处理事件完毕，启动Runloop时设置参数为一次性执行,设置while参数退出Runloop
            if (sourceHandledThisLoop && stopAfterHandle) {
                retVal = kCFRunLoopRunHandledSource;
            // 5.2 如果启动Runloop时设置的最大运转时间到期，设置while参数退出Runloop
            } else if (timeout) {
                retVal = kCFRunLoopRunTimedOut;
            // 5.3 如果启动Runloop被外部调用强制停止，设置while参数退出Runloop
            } else if (__CFRunLoopIsStopped(runloop)) {
                retVal = kCFRunLoopRunStopped;
            // 5.4 如果启动Runloop的modeItems为空，设置while参数退出Runloop
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                retVal = kCFRunLoopRunFinished;
            }

            // 5.5 如果没超时，mode里没空，loop也没被停止，那继续loop，回到第2步循环。
        } while (retVal == 0);
    }

    // 6. 如果第6步判断后loop退出，通知 Observers: RunLoop 退出。--- (OB会销毁新释放池)
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}

###函数作用栈显示：

{
    // 1.1 通知Observers，即将进入RunLoop
    // 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {

        // 2.1 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        // 2.2 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
         // 执行Block
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

        // 2.3 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        // 执行Block
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

        // 3.1 通知Observers，即将进入休眠
        // 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);

        // 3.2 sleep to wait msg.
        mach_msg() -> mach_msg_trap();

        // 3.3 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);

        // 4.1 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);

        // 4.2 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);

        // 4.3 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);

        // 5. 退出判断函数调用栈无显示
    } while (...);

    // 6. 通知Observers，即将退出RunLoop
    // 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}

一步一步写具体的实现逻辑过于繁琐不便理解，按Runloop状态大致分为：

1- Entry：通知OB（创建pool）；
2- 执行阶段：按顺序通知OB并执行timer，source0；若有source1执行source1；
3- 休眠阶段：利用mach_msg判断进入休眠，通知OB（pool的销毁重建）；被消息唤醒通知OB；
4- 执行阶段：按消息类型处理事件；
5- 判断退出条件：如果符合退出条件（一次性执行，超时，强制停止，modeItem为空）则退出，否则回到第2阶段；
6- Exit：通知OB（销毁pool）。

##4、Runloop本质：mach port和mach_msg()。

Mach是XNU的内核，进程、线程和虚拟内存等对象通过端口发消息进行通信，Runloop通过mach_msg()函数发送消息，如果没有port 消息，内核会将线程置于等待状态 mach_msg_trap() 。如果有消息，判断消息类型处理事件，并通过modeItem的callback回调(处理事件的具体执行是在DoBlock里还是在回调里目前我还不太明白？？？)。

Runloop有两个关键判断点，一个是通过msg决定Runloop是否等待，一个是通过判断退出条件来决定Runloop是否循环。


##5、如何处理事件：

###界面刷新：
当UI改变（ Frame变化、 UIView/CALayer 的继承结构变化等）时，或手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理。
苹果注册了一个用来监听BeforeWaiting和Exit的Observer，在它的回调函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

###事件响应：
当一个硬件事件(触摸/锁屏/摇晃/加速等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收， 随后由mach port 转发给需要的App进程。
苹果注册了一个 Source1 (基于 mach port 的) 来接收系统事件，通过回调函数触发Sourece0（所以UIEvent实际上是基于Source0的），调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。
_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。

###手势识别：
如果上一步的 _UIApplicationHandleEventQueue() 识别到是一个guesture手势，会调用Cancel方法将当前的touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。
苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，其回调函数为 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。
当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

###GCD任务：
当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调里执行这个 block。Runloop只处理主线程的block，dispatch 到其他线程仍然是由 libDispatch 处理的。

###timer：（见上modeItem部分）

###网络请求：
关于网络请求的接口:最底层是CFSocket层，然后是CFNetwork将其封装，然后是NSURLConnection对CFNetwork进行面向对象的封装，NSURLSession 是 iOS7 中新增的接口，也用到NSURLConnection的loader线程。所以还是以NSURLConnection为例。
当开始网络传输时，NSURLConnection 创建了两个新线程：com.apple.NSURLConnectionLoader 和 com.apple.CFSocket.private。其中 CFSocket 线程是处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。
![image](https://github.com/WinJayQ/WJRunLoopSummary/raw/master/wj.png)

           RunLoop_network.png

##6、应用：

###滑动与图片刷新；
当tableview的cell上有需要从网络获取的图片的时候，滚动tableView，异步线程会去加载图片，加载完成后主线程就会设置cell的图片，但是会造成卡顿。可以让设置图片的任务在CFRunLoopDefaultMode下进行，当滚动tableView的时候，RunLoop是在 UITrackingRunLoopMode 下进行，不去设置图片，而是当停止的时候，再去设置图片。

- (void)viewDidLoad {
  [super viewDidLoad];
  // 只在NSDefaultRunLoopMode下执行(刷新图片)
  [self.myImageView performSelector:@selector(setImage:) withObject:[UIImage imageNamed:@""] afterDelay:ti inModes:@[NSDefaultRunLoopMode]];    
}

###常驻子线程，保持子线程一直处理事件
为了保证线程长期运转，可以在子线程中加入RunLoop，并且给Runloop设置item，防止Runloop自动退出。

+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
- (void)start {
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}

这是一篇综合官方文档、绘制像素到屏幕上、View-Layer 协作、sunnyxx关于runloop的线下视频、深入理解RunLoop后加上自己的个人总结，各个资料有些说法都有差异，自己有整理有验证，也还存有疑惑，放上来希望得到指正。

####转自楚天舒的简述-->http://www.jianshu.com/p/37ab0397fec7
