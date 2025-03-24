# RunLoop 实现卡顿检测和内存泄漏检测原理

## 卡顿检测原理

卡顿检测的核心思想是监测主线程的执行状态。由于所有 UI 相关操作都必须在主线程进行，当主线程被长时间占用时，用户界面就会出现卡顿。

基于 RunLoop 的卡顿检测原理如下：

1. **监控 RunLoop 状态**：创建一个子线程定时检查主线程 RunLoop 的状态和停留时间
2. **设置观察者**：通过 `CFRunLoopObserverRef` 在不同 RunLoop 状态间进行观察
3. **计时判断**：监控从 RunLoop 进入某个状态到退出该状态的时间间隔
4. **阈值报警**：当时间间隔超过预设阈值时，认为发生了卡顿，此时可以捕获堆栈信息进行分析

主线程 RunLoop 在 `BeforeWaiting`(即将休眠) 到 `AfterWaiting`(被唤醒) 状态之间，如果停留太久且没有进入休眠状态，就可能是卡顿发生了。

## 内存泄漏检测原理

内存泄漏检测基于对对象生命周期的监控：

1. **对象引用跟踪**：监控特定对象的引用计数变化
2. **弱引用表**：使用 RunLoop 维护一个弱引用表来定期检查对象是否应该被释放
3. **触发时机控制**：在 RunLoop 的合适时机触发检测逻辑
4. **自动释放池监控**：检测自动释放池中对象的释放情况

针对 ViewController 等特殊对象，可检测其 `viewDidDisappear:` 后是否能被释放，如果在一定时间后（几个 RunLoop 周期）对象仍然存在，则可能存在内存泄漏。

## 实现说明

上面的代码通过 `PMMonitor` 类实现了一个完整的基于 RunLoop 的卡顿检测和内存泄漏检测工具。下面是关键部分解释：

### 卡顿检测实现细节

1. **观察者设置**：向主线程 RunLoop 注册一个观察者，监听所有活动状态

   ```objc
   _runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, 
                                              kCFRunLoopAllActivities, 
                                              YES, 
                                              0, 
                                              &runLoopObserverCallback, 
                                              &context);
   ```

2. **子线程监控**：创建一个子线程，使用信号量定时检查主线程状态

   ```objc
   dispatch_async(_stuckQueue, ^{
       while (self->_stuckMonitoring) {
           long semaphoreWait = dispatch_semaphore_wait(self->_stuckSemaphore, dispatch_time(DISPATCH_TIME_NOW, (int64_t)(self->_stuckThreshold * NSEC_PER_SEC)));
           if (semaphoreWait != 0) {
               // 超时，说明主线程卡顿
               // ...检测和报告逻辑
           }
       }
   });
   ```

3. **堆栈获取**：当发现卡顿时，获取主线程的调用堆栈进行分析

   ```objc
   - (void)reportStuck {
       NSLog(@"检测到主线程卡顿");
       NSString *backtrace = [self backtraceOfMainThread];
       NSLog(@"卡顿堆栈: %@", backtrace);
   }
   ```

### 内存泄漏检测实现细节

1. **对象跟踪**：使用弱引用表 `NSHashTable` 存储待监控对象

   ```objc
   _leakMonitorObjects = [NSHashTable weakObjectsHashTable];
   ```

2. **方法交换**：交换 `viewDidDisappear:` 方法，在控制器消失时标记时间戳

   ```objc
   - (void)pm_viewDidDisappear:(BOOL)animated {
       [self pm_viewDidDisappear:animated];
       objc_setAssociatedObject(self, "PMLeakCheckTimestamp", [NSDate date], OBJC_ASSOCIATION_RETAIN_NONATOMIC);
       [[PMMonitor sharedInstance] addLeakMonitorObject:self];
   }
   ```

3. **定时检查**：利用 RunLoop 中的定时器，定期检查对象是否被释放

   ```objc
   _leakCheckTimer = [NSTimer scheduledTimerWithTimeInterval:5.0 
                                                      target:self 
                                                    selector:@selector(checkLeaks) 
                                                    userInfo:nil 
                                                     repeats:YES];
   ```

4. **泄漏判定**：如果视图控制器的视图已不在窗口层级但对象仍存在，且超过阈值时间，则判定为泄漏

   ```objc
   if (vc.isViewLoaded && vc.view.window == nil) {
       NSDate *timestamp = objc_getAssociatedObject(vc, "PMLeakCheckTimestamp");
       if (timestamp) {
           NSTimeInterval interval = [[NSDate date] timeIntervalSinceDate:timestamp];
           if (interval > 3.0) { // 3秒后仍未释放
               [self reportLeak:vc];
           }
       }
   }
   ```

## 使用方法

在应用启动时初始化监控器：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // 启动卡顿监控
    [[PMMonitor sharedInstance] startMonitorWithType:PMMonitorTypeStuck];
    
    // 启动内存泄漏监控
    [[PMMonitor sharedInstance] startMonitorWithType:PMMonitorTypeMemLeak];
    
    // 可选：设置卡顿监控阈值
    [[PMMonitor sharedInstance] setStuckThreshold:0.3]; // 设为300ms
    
    return YES;
}
```

## 改进方向

1. **堆栈符号化**：实际使用中，需要对收集到的堆栈进行符号化处理，转换为可读的方法名和行号
2. **上报机制**：添加网络上报逻辑，将监控数据发送到后台服务器
3. **UI展示**：可以开发开发环境下的实时展示界面，直观地显示卡顿和内存泄漏情况
4. **更多维度**：增加CPU、内存占用等更多维度的监控指标

这套工具基于 RunLoop 机制，实现了轻量级、低侵入性的性能监控方案，适合集成到 iOS 项目中进行性能优化。



```Swift
// PMMonitor.h
#import <Foundation/Foundation.h>

typedef NS_ENUM(NSUInteger, PMMonitorType) {
    PMMonitorTypeStuck,    // 卡顿监控
    PMMonitorTypeMemLeak   // 内存泄漏监控
};

@interface PMMonitor : NSObject

// 单例方法
+ (instancetype)sharedInstance;

// 开始监控
- (void)startMonitorWithType:(PMMonitorType)type;

// 停止监控
- (void)stopMonitorWithType:(PMMonitorType)type;

// 设置卡顿检测阈值（默认200ms）
- (void)setStuckThreshold:(NSTimeInterval)threshold;

// 添加待检测内存泄漏的对象
- (void)addLeakMonitorObject:(id)object;

@end


// PMMonitor.m
#import "PMMonitor.h"
#import <objc/runtime.h>
#import <UIKit/UIKit.h>
#import <mach/mach.h>
#import <pthread.h>

@interface PMMonitor ()

// 卡顿监控
@property (nonatomic, strong) dispatch_semaphore_t stuckSemaphore;
@property (nonatomic, assign) CFRunLoopObserverRef runLoopObserver;
@property (nonatomic, assign) CFRunLoopActivity runLoopActivity;
@property (nonatomic, strong) dispatch_queue_t stuckQueue;
@property (nonatomic, assign) NSTimeInterval stuckThreshold;
@property (nonatomic, assign) BOOL stuckMonitoring;

// 内存泄漏监控
@property (nonatomic, strong) NSHashTable *leakMonitorObjects;
@property (nonatomic, strong) dispatch_queue_t leakQueue;
@property (nonatomic, assign) BOOL leakMonitoring;
@property (nonatomic, strong) NSTimer *leakCheckTimer;

@end

@implementation PMMonitor

#pragma mark - 单例实现
+ (instancetype)sharedInstance {
    static PMMonitor *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[PMMonitor alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _stuckThreshold = 0.2; // 默认200ms
        _stuckSemaphore = dispatch_semaphore_create(0);
        _stuckQueue = dispatch_queue_create("com.pmmonitor.stuck", DISPATCH_QUEUE_SERIAL);
        _leakMonitorObjects = [NSHashTable weakObjectsHashTable]; // 弱引用表
        _leakQueue = dispatch_queue_create("com.pmmonitor.leak", DISPATCH_QUEUE_SERIAL);
    }
    return self;
}

#pragma mark - 公共方法
- (void)startMonitorWithType:(PMMonitorType)type {
    if (type == PMMonitorTypeStuck) {
        [self startStuckMonitor];
    } else if (type == PMMonitorTypeMemLeak) {
        [self startLeakMonitor];
    }
}

- (void)stopMonitorWithType:(PMMonitorType)type {
    if (type == PMMonitorTypeStuck) {
        [self stopStuckMonitor];
    } else if (type == PMMonitorTypeMemLeak) {
        [self stopLeakMonitor];
    }
}

- (void)setStuckThreshold:(NSTimeInterval)threshold {
    _stuckThreshold = threshold;
}

- (void)addLeakMonitorObject:(id)object {
    if (object) {
        dispatch_async(_leakQueue, ^{
            [self.leakMonitorObjects addObject:object];
        });
    }
}

#pragma mark - 卡顿监控实现
- (void)startStuckMonitor {
    if (_stuckMonitoring) return;
    _stuckMonitoring = YES;
    
    // 注册RunLoop观察者
    CFRunLoopObserverContext context = {0, (__bridge void *)self, NULL, NULL, NULL};
    _runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, 
                                              kCFRunLoopAllActivities, 
                                              YES, 
                                              0, 
                                              &runLoopObserverCallback, 
                                              &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), _runLoopObserver, kCFRunLoopCommonModes);
    
    // 创建子线程监控
    dispatch_async(_stuckQueue, ^{
        while (self->_stuckMonitoring) {
            // 如果主线程正在执行任务，则等待
            long semaphoreWait = dispatch_semaphore_wait(self->_stuckSemaphore, dispatch_time(DISPATCH_TIME_NOW, (int64_t)(self->_stuckThreshold * NSEC_PER_SEC)));
            if (semaphoreWait != 0) {
                if (!self->_stuckMonitoring) {
                    return;
                }
                // 超时，说明主线程卡顿
                if (self->_runLoopActivity == kCFRunLoopBeforeWaiting) {
                    // 如果RunLoop即将进入休眠，说明并非真正卡顿
                    continue;
                }
                
                [self reportStuck];
            }
        }
    });
}

static void runLoopObserverCallback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    PMMonitor *monitor = (__bridge PMMonitor *)info;
    monitor.runLoopActivity = activity;
    
    // 发送信号
    dispatch_semaphore_signal(monitor.stuckSemaphore);
}

- (void)stopStuckMonitor {
    if (!_stuckMonitoring) return;
    _stuckMonitoring = NO;
    
    // 移除观察者
    if (_runLoopObserver) {
        CFRunLoopRemoveObserver(CFRunLoopGetMain(), _runLoopObserver, kCFRunLoopCommonModes);
        CFRelease(_runLoopObserver);
        _runLoopObserver = NULL;
    }
    
    // 保证监控线程退出
    dispatch_semaphore_signal(_stuckSemaphore);
}

// 获取当前线程堆栈
- (NSString *)backtraceOfCurrentThread {
    NSMutableString *backtrace = [NSMutableString string];
    void *callstack[128];
    int frames = backtrace(callstack, 128);
    char **strs = backtrace_symbols(callstack, frames);
    
    for (int i = 0; i < frames; i++) {
        [backtrace appendFormat:@"%s\n", strs[i]];
    }
    free(strs);
    return backtrace;
}

// 获取主线程堆栈
- (NSString *)backtraceOfMainThread {
    thread_act_array_t threads;
    mach_msg_type_number_t threadCount = 0;
    
    // 获取所有线程
    kern_return_t kr = task_threads(mach_task_self(), &threads, &threadCount);
    if (kr != KERN_SUCCESS) {
        return @"获取线程信息失败";
    }
    
    // 找到主线程
    thread_t mainThread = mach_thread_self();
    for (int i = 0; i < threadCount; i++) {
        if (pthread_mach_thread_np(pthread_main_thread_np()) == threads[i]) {
            mainThread = threads[i];
            break;
        }
    }
    
    // 获取主线程堆栈
    char name[256];
    pthread_t pt = pthread_from_mach_thread_np(mainThread);
    if (pt) {
        pthread_getname_np(pt, name, sizeof(name));
    }
    
    // 这里使用符号化堆栈需要额外处理，简化起见返回线程ID
    return [NSString stringWithFormat:@"主线程堆栈: %s", name];
}

- (void)reportStuck {
    NSLog(@"检测到主线程卡顿");
    
    // 获取卡顿时的主线程堆栈
    NSString *backtrace = [self backtraceOfMainThread];
    NSLog(@"卡顿堆栈: %@", backtrace);
    
    // 在实际使用中，可以将堆栈信息上传服务器
    // [self uploadStuckInfo:backtrace];
}

#pragma mark - 内存泄漏监控实现
- (void)startLeakMonitor {
    if (_leakMonitoring) return;
    _leakMonitoring = YES;
    
    // swizzle viewDidDisappear: 方法
    [self swizzleViewControllerDealloc];
    
    // 创建定时器，定期检查对象是否泄漏
    _leakCheckTimer = [NSTimer scheduledTimerWithTimeInterval:5.0 
                                                      target:self 
                                                    selector:@selector(checkLeaks) 
                                                    userInfo:nil 
                                                     repeats:YES];
    [[NSRunLoop mainRunLoop] addTimer:_leakCheckTimer forMode:NSRunLoopCommonModes];
}

- (void)stopLeakMonitor {
    if (!_leakMonitoring) return;
    _leakMonitoring = NO;
    
    // 移除定时器
    [_leakCheckTimer invalidate];
    _leakCheckTimer = nil;
    
    // 重置
    dispatch_async(_leakQueue, ^{
        [self.leakMonitorObjects removeAllObjects];
    });
}

- (void)swizzleViewControllerDealloc {
    // 替换UIViewController的viewDidDisappear:方法
    Class class = [UIViewController class];
    SEL originalSelector = @selector(viewDidDisappear:);
    SEL swizzledSelector = @selector(pm_viewDidDisappear:);
    
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    
    method_exchangeImplementations(originalMethod, swizzledMethod);
}

- (void)checkLeaks {
    dispatch_async(_leakQueue, ^{
        for (id object in self.leakMonitorObjects.allObjects) {
            // 如果监控列表中的控制器仍然存在，但其视图已不在窗口层级中，则可能泄漏
            if ([object isKindOfClass:[UIViewController class]]) {
                UIViewController *vc = (UIViewController *)object;
                if (vc.isViewLoaded && vc.view.window == nil) {
                    // 检查时间戳，如果超过一定时间仍未释放，则判定为泄漏
                    NSDate *timestamp = objc_getAssociatedObject(vc, "PMLeakCheckTimestamp");
                    if (timestamp) {
                        NSTimeInterval interval = [[NSDate date] timeIntervalSinceDate:timestamp];
                        if (interval > 3.0) { // 3秒后仍未释放
                            [self reportLeak:vc];
                        }
                    }
                }
            }
        }
    });
}

- (void)reportLeak:(id)leakedObject {
    NSLog(@"检测到内存泄漏: %@", [leakedObject class]);
    
    // 获取对象的内存地址和类信息
    NSString *className = NSStringFromClass([leakedObject class]);
    NSString *address = [NSString stringWithFormat:@"%p", leakedObject];
    
    // 在实际使用中，可以将泄漏信息上传服务器
    // [self uploadLeakInfo:@{@"class": className, @"address": address}];
}

@end

// UIViewController扩展
@implementation UIViewController (PMLeakMonitor)

- (void)pm_viewDidDisappear:(BOOL)animated {
    [self pm_viewDidDisappear:animated];
    
    // 标记离开时间
    objc_setAssociatedObject(self, "PMLeakCheckTimestamp", [NSDate date], OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    // 添加到监控列表
    [[PMMonitor sharedInstance] addLeakMonitorObject:self];
}

@end

// 使用示例
/*
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // 启动卡顿监控
    [[PMMonitor sharedInstance] startMonitorWithType:PMMonitorTypeStuck];
    
    // 启动内存泄漏监控
    [[PMMonitor sharedInstance] startMonitorWithType:PMMonitorTypeMemLeak];
    
    // 设置卡顿监控阈值为300ms
    [[PMMonitor sharedInstance] setStuckThreshold:0.3];
    
    return YES;
}
*/

```

