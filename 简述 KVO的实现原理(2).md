# 简述 KVO的实现原理(2)

好的，我们来简述一下 KVO (Key-Value Observing) 的实现原理。

KVO 的核心在于 Objective-C **动态运行时 (Runtime)** 的特性，特别是 **`isa` 指针的混写 (isa-swizzling)** 和**类的动态创建**。

其基本原理可以概括为以下几步：

1.  **动态创建子类:**
    *   当你第一次对一个对象（比如 `MyObject` 的实例 `obj`）的某个属性（比如 `propertyName`）调用 `addObserver:forKeyPath:options:context:` 方法时，Objective-C 运行时会在**内部动态地**创建一个**新的子类**。
    *   这个子类的命名通常是基于原类名，加上一个前缀，例如 `NSKVONotifying_MyObject`。
    *   这个动态子类会**继承**自原始的 `MyObject` 类。

2.  **修改 `isa` 指针 (isa-swizzling):**
    *   运行时会**修改**被观察对象 `obj` 的 `isa` 指针，让它从指向原来的 `MyObject` 类**改变为指向新创建的** `NSKVONotifying_MyObject` 子类。
    *   从此以后，虽然你从外部看 `obj` 仍然是 `MyObject` 类型（`[obj class]` 会返回 `MyObject`，因为子类重写了 `class` 方法），但它的**实际类**已经是那个动态生成的子类了。

3.  **重写被观察属性的 Setter 方法:**
    *   在动态创建的子类 `NSKVONotifying_MyObject` 中，运行时会**重写 (override)** 被观察属性 `propertyName` 的 **setter 方法** (例如 `setPropertyName:`)。
    *   这个重写后的 setter 方法大致会执行以下操作：
        a.  调用 Foundation 框架的内部函数，通知观察者属性**即将改变** (`willChangeValueForKey:`)。
        b.  调用**原始父类**（即 `MyObject`）的该属性的 **setter 实现**，真正地去修改属性的值 (`[super setPropertyName:newValue]`)。
        c.  再次调用 Foundation 框架的内部函数，通知观察者属性**已经改变** (`didChangeValueForKey:`)。

4.  **触发观察者方法:**
    *   在 `didChangeValueForKey:` 方法内部（或其他相关通知机制中），系统会遍历所有注册到该对象该属性上的观察者，并调用它们的 `observeValueForKeyPath:ofObject:change:context:` 方法，将属性变更的详细信息传递过去。

**总结关键点:**

*   KVO 利用了 Objective-C 的**动态性**。
*   它通过在运行时**动态创建子类**来实现。
*   通过**修改对象的 `isa` 指针**，将被观察对象“偷偷”变成了这个动态子类的实例。
*   在动态子类中**重写被观察属性的 setter 方法**，在真正修改值的前后插入了通知逻辑 (`willChangeValueForKey:` 和 `didChangeValueForKey:`)。
*   最终通过 `didChangeValueForKey:` 触发观察者的 `observeValueForKeyPath:...` 方法。

这种基于 `isa-swizzling` 和动态子类的方式，使得 KVO 可以在**不修改原始类代码**的情况下，为任何 KVC (Key-Value Coding) 兼容的属性添加观察能力。