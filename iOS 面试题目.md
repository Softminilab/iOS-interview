## iOS 面试题目



#### Swift语言特性

Swift是现代iOS开发的首选语言，其特性如泛型、协议、异步/等待（async/await）等需要深入理解。示例问题包括：

* 解释Swift类型系统如何处理泛型和类型擦除。

  * Swift的类型系统通过泛型允许编写灵活、可重用的代码，类型参数在编译时确定，生成特定类型的专用代码。类型擦除则通过协议或`Any`隐藏具体类型信息，运行时仅通过共同接口操作。例如，使用 `Array<Any>` 会擦除类型信息，而泛型如 `Array<Int>` 在运行时保留类型信息。

* 讨论使用结构体（structs）与类（classes）的性能影响。

  * 结构体：值类型，存储在 `栈` 上，赋值时复制，适合小数据，复制开销可能较大。  
  * 类：引用类型，存储在 `堆` 上，共享引用，适合大数据或需共享的场景。
    性能上，结构体访问更快，类涉及动态内存管理，可能较慢。建议小数据用结构体，大数据用类。

* Swift如何管理值类型和引用类型的内存？  

  * 值类型：如结构体，存储在 `栈` 上，赋值时复制，无引用计数。  
  * 引用类型：如类，存储在 `堆` 上，使用ARC管理引用计数，0时释放。
    值类型内存管理简单，引用类型需注意循环引用

  

####  Objective-C

尽管Swift逐渐取代Objective-C，但了解Objective-C的运行时、分类和内存管理仍是重要技能。示例问题包括：

- Objective-C运行时是什么，与其他语言有何不同？

  - Objective-C运行时提供动态方法调用、运行时类型信息等功能。不同点：支持动态方法解析，消息传递机制可拦截或转发，灵活性高，如运行时添加方法

- 如何实现分类（category），及其限制是什么？

  - 分类通过@interface ClassName (CategoryName)扩展类，示例：  

    ```objectivec
    @interface NSString (MyCategory)  
    - (void)newMethod;  
    @end  
    ```

    限制：不能添加实例变量，方法名冲突可能导致未定义行为，不能覆盖现有方法。  

    

- 解释方法交换（method swizzling）的概念和应用场景。

  - 方法交换动态替换方法实现，常用于日志、调试或框架行为修改。
    应用场景：添加全局日志，修改框架方法。
    示例：用method_exchangeImplementations交换方法，确保在load方法中执行。  

  

#### UIKit

UIKit是iOS界面开发的传统框架，涉及视图控制器、动画和布局优化。示例问题包括：

- 如何处理视图控制器之间的自定义过渡动画？

  - 通过实现UIViewControllerTransitioningDelegate，定义UIAnimationController处理呈现和消失动画。

    ```swift
    class CustomTransitionDelegate: NSObject, UIViewControllerTransitioningDelegate {  
        func animationController(forPresented presented: UIViewController, presenting: UIViewController, source: UIViewController) -> UIViewControllerAnimationController? {  
            return CustomPresentAnimationController()  
        }  
    }  
    ```

- layoutSubviews与drawRect的区别是什么？

  - layoutSubviews：调整子视图布局，bounds变化时调用。  
  - drawRect：自定义绘制内容，性能开销大，尽量减少使用。
    用途：前者布局，后者渲染；调用顺序：layoutSubviews先于drawRect。

- 如何优化UITableView的滚动性能？

  - 使用可重用单元格dequeueReusableCellWithIdentifier。  
  - 减少单元格子视图数量，优化Auto Layout。  
  - 批量更新数据用performBatchUpdates。  
  - 使用Instruments分析性能瓶颈。

  

#### Core Data

Core Data是iOS数据持久化的核心框架，涉及数据建模、关系处理和性能优化。示例问题包括：

- 如何在Core Data中处理实体之间的关系？
- 什么是获取请求（fetch request）与谓词（predicate）的区别？
  - 获取请求：定义从Core Data检索数据的规则，包括实体、排序、分批等。  
  - 谓词：条件过滤返回的对象，如NSPredicate(format: "age > 20")。
    获取请求可包含谓词以筛选结果。
- 如何优化Core Data处理大型数据集的性能？
  - 使用fetchBatchSize限制每次获取的对象数。  
  - 启用故障（faulting），按需加载数据。  
  - 为常用谓词属性建立索引。  
  - 使用背景上下文处理繁重操作，避免阻塞主线程。



#### 网络编程

网络编程是iOS开发的重要部分，包括HTTP请求、错误处理和安全通信。示例问题包括：

- 如何使用URLSession处理HTTP请求和响应？
- 处理网络失败和重试的策略有哪些？
  - 检查错误和HTTP状态码，通知用户失败。  
  - 实现指数退避重试，逐步增加重试间隔。  
  - 缓存响应，提供离线数据。  
  - 使用第三方库如Alamofire支持自动重试。
- 如何在iOS应用中实现安全网络通信？
  - 使用HTTPS加密通信。  
  - 实现证书固定，验证服务器证书。  
  - 使用令牌认证，确保请求安全。  
  - 加密敏感数据传输，防止窃听。



#### 并发性

并发性涉及GCD、NSOperationQueue和线程安全，是高性能iOS开发的关键。示例问题包括：

- 解释GCD的工作原理，如何用于后台任务？

  - GCD（Grand Central Dispatch）通过调度队列管理并发任务。  

    队列类型：串行队列顺序执行，并发队列并行执行。  

    后台任务：用dispatch_async在全局队列执行，示例：

    ```swift
    let backgroundQueue = DispatchQueue.global()  
    backgroundQueue.async {  
        // 后台任务  
    }  
    ```

- 如何使用DispatchGroup同步多个异步任务？

  - 创建DispatchGroup，跟踪任务。  
  - enter开始任务，leave结束，notify或wait等待完成。
    示例：

  ```swift
  let group = DispatchGroup()  
  for i in 1...5 {  
      group.enter()  
      DispatchQueue.global().async {  
          // 任务  
          group.leave()  
      }  
  }  
  group.wait() 
  ```

- 如何避免多线程环境中的死锁？

  - 一致锁顺序，避免嵌套锁。  
  - 使用条件变量等待，不持锁等待。  
  - 调试工具监控线程状态，检测潜在死锁。  
  - 最小化共享资源访问，减少竞争

* GCD与NSOperationQueue的区别是什么？  

  * GCD：基于块，低级API，适合简单任务。  
  * NSOperationQueue：基于对象，高级API，支持操作依赖和取消，适合复杂任务。
    GCD更轻量，NSOperationQueue更灵活。

  

#### 内存管理

内存管理涉及ARC、内存泄漏检测和性能优化，是iOS开发的重要技能。示例问题包括：

- ARC是什么，如何工作？
  - ARC（自动引用计数）自动管理对象生命周期，通过引用计数决定释放。  
    - 赋值增加计数，释放减少，0时销毁。  
    - 避免手动retain/release，减少内存泄漏。
- 如何检测和修复iOS应用中的内存泄漏？
  - 使用Instruments的Leaks工具检测。  
  - 检查代码，寻找循环引用，用弱引用解决。  
  - 确保deinit方法被调用，释放资源。  
  - 定期清理缓存，减少内存占用。
- weak与unowned引用的区别是什么？
  - weak：可为nil，引用对象释放后自动置nil，适合可选关系。  
  - unowned：假设对象永不释放，崩溃风险高，适合确定对象存活的场景。
- 如何管理大型数据结构或图片的内存？  
  - 使用懒加载，按需加载数据。  
  - 压缩图片，减少内存占用。  
  - 启用故障机制，Core Data按需加载。  
  - 监听低内存警告，释放缓存。



#### 设计模式

设计模式如MVC、MVVM、VIPER用于组织代码结构。示例问题包括：

- MVC模式在iOS开发中的应用是什么？
  - MVC（模型-视图-控制器）分离职责：  
    - 模型：数据和逻辑。  
    - 视图：UI组件。  
    - 控制器：管理模型和视图交互。
      iOS中，视图控制器常为控制器，模型为自定义类，视图为UIKit组件。
- 解释MVVM, MVC, MVP, VIPER模式及其优势。(未完)
- 何时使用单例模式（Singleton），提供一个示例？
  - 使用场景：全局访问共享资源，如设置管理。
- 解释观察者模式及其在iOS框架中的应用。  
  - 观察者模式：对象间一对多依赖，状态变化通知观察者。
    iOS应用：如NSNotificationCenter广播通知，KVO监控属性变化。
    示例：用通知中心监听事件，解耦组件。  
- iOS开发中常用的其他设计模式有哪些？  (未完)
  - 工厂模式：创建对象，如UICollectionViewCell工厂。  
  - 适配器模式：适配不同接口，如第三方库适配。  
  - 策略模式：定义算法族，如排序策略切换。  
  - 装饰器模式：动态添加功能，如视图装饰。
- 设计模式有哪些？
- 常见设计模式对比

#### 应用生命周期

应用生命周期涉及状态管理、深链接和资源优化。示例问题包括：

- iOS应用的不同状态有哪些，如何处理状态转换？
- 如何处理深链接以导航到特定屏幕？
- 如何在应用后台时管理资源以减少内存使用？



#### 安全

安全涉及Keychain使用、HTTPS和证书固定。示例问题包括：

- 如何使用Keychain存储敏感数据？
  - 使用KeychainServices或KeychainAccess库存储数据，设置唯一标识符，加密存储密码或密钥，检索时用相同标识符，确保数据安全受设备保护。  
- HTTPS如何确保安全通信？
  - HTTPS通过SSL/TLS加密数据，验证服务器证书防止中间人攻击，确保传输数据不可读和篡改，iOS默认支持HTTPS通信。
- 如何防止应用被逆向工程？
  -    使用代码混淆，加密敏感数据，检测调试环境，定期更新修补漏洞，遵循安全编码实践，但需注意无绝对防逆向方法。  
- 如何确保应用符合GDPR或CCPA隐私法规？
  - 透明告知数据收集用途，获取用户明确同意，收集最小必要数据，保护数据安全，提供访问/删除数据功能，记录处理活动以符合法规。  
- 越狱检测是什么，为什么可能需要实现？
  - 越狱检测检查设备是否被修改以访问受保护区域，需实现以保护知识产权，维护安全环境，符合服务条款，但需注意检测可能被绕过。



#### 性能

性能优化涉及使用Instruments分析、启动时间优化和GPU性能。示例问题包括：

- 如何使用Instruments分析应用性能？
- 如何优化应用的启动时间？
  - 减少启动时执行代码，延迟加载非必需资源，使用预编译头加速开发，优化图片加载，分析启动瓶颈以改进
- 如何减少应用的二进制文件大小？
  - 启用Bitcode，移除调试符号，优化图片压缩，管理依赖避免包含多余库，清理未用代码减少体积。 



✔️ 使用 Runtime 就行埋单统计
✔️ 使用 RunLoop 对项目进行卡顿检查，内存泄漏检测
✔️ 使用LLVM埋点，对app进行二进制重排
✔️ 熟练使用shell来处理符号表冲突问题
✔️ 熟练使用shell来处理符号表冲突问题
✔️ 熟练使用数据库 FMDB,Realm,SQLite,Core data
✔️ NSURLProtocol
✔️ 熟练掌握Runtime,Runlop底层原理以及底层源码分析
✔️ 熟练掌握 KVC，KVO 使用及其底层原理
✔️ 熟练掌握多线程原理以及 NSThread，NSOperation 使用
✔️ 熟练掌握 GCD高级使用以及 GCD 源码分析
✔️ 熟练使用设计模式
✔️ 熟练使用流行的架构模式