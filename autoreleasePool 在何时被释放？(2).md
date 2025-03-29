# autoreleasePool 在何时被释放？(2)

`@autoreleasepool` (自动释放池) 的释放（或者更准确地说，是**排干 - drain**）时机取决于它是如何被创建和管理的：

1.  **基于 RunLoop 的自动释放池 (系统管理):**
    *   **场景:** 这是最常见的情况，发生在应用程序的主线程或任何配置了 RunLoop 的线程上。
    *   **时机:** 系统会在 RunLoop 的**每个事件循环 (event loop iteration)** 中自动创建和排干自动释放池。具体来说：
        *   在处理一个事件（如用户触摸、定时器触发、网络数据到达等）**之前**，系统会创建一个自动释放池。
        *   在该事件的所有处理代码（包括你的应用程序代码、系统框架的回调等）执行**完毕后**，系统会排干（drain）这个自动释放池。排干时，池中所有被标记为 autorelease 的对象会被发送 `release` 消息。
        *   然后 RunLoop 进入休眠状态，等待下一个事件。当下个事件到来时，重复此过程。
    *   **Implication:** 这意味着在一个事件处理周期内创建的所有 autoreleased 对象，通常会在这个周期结束时被释放（如果它们的引用计数因此降为0）。这确保了内存不会在事件之间无限增长。

2.  **手动创建的自动释放池 (`@autoreleasepool { ... }` 块):**
    *   **场景:** 当你在代码中显式使用 `@autoreleasepool` 块时。这常见于：
        
        *   在一个循环中创建大量临时对象，需要及时释放内存，防止内存峰值过高。
        *   在没有 RunLoop 的辅助线程（如 GCD 的后台队列分发的任务）中执行代码，需要自己管理 autoreleased 对象。
    *   **时机:** 手动创建的自动释放池会在执行流程**离开 `@autoreleasepool` 块的作用域时**（即到达右花括号 `}` 时）被排干。
    *   **示例:**
        
        ```objectivec
        // 在没有 RunLoop 的后台线程中
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            for (int i = 0; i < 10000; ++i) {
                @autoreleasepool { // 每次循环创建一个池
                    // 创建大量临时 autoreleased 对象
                    NSString *tempString = [NSString stringWithFormat:@"Iteration %d", i];
                    NSURL *tempURL = [NSURL URLWithString:tempString];
                    // ... 使用 tempString, tempURL ...
        
                } // 在这里，本次循环创建的 tempString, tempURL 等会被 release
                  // 避免内存峰值过高
            }
        
            // --- 或者 ---
        
            @autoreleasepool { // 包裹整个任务
                // ... 执行一些可能会产生 autoreleased 对象的操作 ...
                MyObject *obj = [MyObject temporaryObject]; // 假设返回 autoreleased 对象
        
            } // 在这里，obj 会被 release
        });
        ```
    
3.  **NSThread 创建的线程:**
    *   如果你使用 `NSThread` 创建并启动一个线程，你**必须**在该线程的入口方法内部**手动创建**一个 `@autoreleasepool` 来包裹你的代码，尤其是在循环或长时间运行的任务中。否则，在该线程上创建的 autoreleased 对象将没有池来管理，会导致内存泄漏。这个手动创建的池的释放时机遵循规则 #2。

**总结:**

*   对于主线程和有 RunLoop 的线程，系统在**每个事件循环结束时**自动排干 Autorelease Pool。
*   对于手动创建的 `@autoreleasepool` 块，池在**代码执行离开该块作用域时**被排干。
*   没有 RunLoop 的辅助线程（如普通 GCD 队列任务）需要**开发者手动创建和管理** `@autoreleasepool`。

理解 Autorelease Pool 的释放时机对于编写内存高效的 Objective-C 代码（尤其是在 ARC 环境下理解其背后的机制）非常重要。