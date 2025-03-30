# @protocol和category中如何使用@property(2)

好的，我们来分别看看在 `@protocol` (协议) 和 `category` (分类) 中如何以及为何使用 `@property`，它们的含义和限制是不同的。

**1. 在 `@protocol` 中使用 `@property`**

*   **目的/含义:**
    *   在协议中使用 `@property` 是为了**声明该协议要求遵循它的类必须提供指定属性的访问器方法（getter，以及可选的 setter）**。
    *   它定义了一个**契约 (Contract)**：任何遵守该协议的类都需要能够响应与该属性相关的 getter（对于 `readonly` 和 `readwrite` 属性）和 setter（对于 `readwrite` 属性）。
    *   协议本身**不会**为这个属性自动合成实例变量 (ivar) 或实现访问器方法。它仅仅是**声明了一个要求**。

*   **语法:**
    ```objectivec
    @protocol MyDataSourceProtocol <NSObject>
    
    // 要求遵循者提供一个只读的 name 属性 (至少需要实现 -name 方法)
    @property (nonatomic, strong, readonly) NSString *name;
    
    // 要求遵循者提供一个可读写的 data 属性 (需要实现 -data 和 -setData: 方法)
    @property (nonatomic, copy) NSData *data;
    
    // 可选的属性要求 (使用 @optional)
    @optional
    @property (nonatomic, weak) id<MyDelegateProtocol> delegate;
    
    @required // @required 是默认的，可以省略
    @property (nonatomic, assign) NSInteger itemCount;
    
    @end
    ```
    *   你可以像在类接口中一样使用各种属性修饰符 (`nonatomic`, `strong`, `weak`, `copy`, `readonly`, `readwrite`, `assign` 等) 来精确定义所需的语义和内存管理策略。

*   **遵循者类的责任:**
    *   遵守该协议的类**必须**满足协议中 `@required` (默认) 的 `@property` 声明。
    *   这通常意味着遵循类需要在其 `.h` 文件中也声明一个具有**相同名称和兼容类型**的 `@property`，并在 `.m` 文件中 `@synthesize`（旧式）或依赖自动合成（现代）来生成 ivar 和访问器方法。
    *   或者，遵循类可以不声明 `@property`，但**必须手动实现**协议要求的 getter 和/或 setter 方法。
    *   对于 `@optional` 的属性，遵循类可以选择性地提供。

*   **示例:**
    ```objectivec
    // MyDataObject.h
    #import "MyDataSourceProtocol.h"
    
    @interface MyDataObject : NSObject <MyDataSourceProtocol>
    
    // 编译器会自动检查 MyDataObject 是否满足 MyDataSourceProtocol 的要求
    
    // 实现协议要求的属性 (通过自动合成)
    @property (nonatomic, strong, readonly) NSString *name; // 满足 readonly name 要求
    @property (nonatomic, copy) NSData *data;             // 满足 readwrite data 要求
    @property (nonatomic, assign) NSInteger itemCount;    // 满足 readwrite itemCount 要求
    
    // (可以选择性地实现 @optional 的 delegate 属性)
    @property (nonatomic, weak) id<MyDelegateProtocol> delegate;
    
    - (instancetype)initWithName:(NSString *)name;
    
    @end
    
    // MyDataObject.m
    @implementation MyDataObject
    
    // 如果需要自定义ivar名称或旧项目，可以使用 @synthesize
    // @synthesize name = _name;
    // @synthesize data = _data;
    // @synthesize itemCount = _itemCount;
    // @synthesize delegate = _delegate;
    
    - (instancetype)initWithName:(NSString *)name {
        self = [super init];
        if (self) {
            // 对于 readonly 属性，通常在内部赋值给实例变量
            // 现代 Objective-C 会自动合成 _name ivar
            _name = [name copy];
        }
        return self;
    }
    
    // data, itemCount, delegate 的 getter/setter 会被自动合成
    
    @end
    ```

**2. 在 `category` 中使用 `@property`**

*   **目的/含义:**
    *   在分类的接口 (`.h`) 中使用 `@property` 是为了**声明添加到现有类中的访问器方法（getter 和可选的 setter）**。它提供了一种方便的方式来定义与分类添加的功能相关的“属性式”接口。
    *   **关键限制:** 分类**不能**为类添加新的实例变量 (ivar)。因此，分类中的 `@property` **不会自动合成实例变量**。
    *   这意味着你**必须**在分类的实现 (`.m`) 文件中**手动提供** `@property` 所声明的 getter 和 setter 方法的实现。

*   **如何存储数据? (关联对象 - Associated Objects):**
    *   既然不能添加 ivar，那么分类中属性的数据通常存储在哪里？标准做法是使用 Objective-C 运行时的**关联对象 (Associated Objects)** 机制。
    *   关联对象允许你在运行时将任意键值对附加到任何对象实例上。

*   **语法与实现:**
    ```objectivec
    // NSObject+MyCategory.h
    #import <Foundation/Foundation.h>
    
    @interface NSObject (MyCategory)
    
    // 在分类中声明一个属性
    // 注意：这里只是声明了 -(myCustomProperty) 和 -(setMyCustomProperty:) 方法
    // 不会自动生成 _myCustomProperty 实例变量
    @property (nonatomic, strong) NSString *myCustomProperty;
    
    @end
    
    // NSObject+MyCategory.m
    #import "NSObject+MyCategory.h"
    #import <objc/runtime.h> // 需要导入运行时头文件
    
    // 定义一个静态的 key，用于关联对象。地址必须唯一。
    static const char MyCustomPropertyKey = '\0'; // 或者 static void *MyCustomPropertyKey = &MyCustomPropertyKey;
    
    @implementation NSObject (MyCategory)
    
    // 手动实现 getter 方法
    - (NSString *)myCustomProperty {
        // 使用 objc_getAssociatedObject 获取关联对象
        return objc_getAssociatedObject(self, &MyCustomPropertyKey);
        // 或者 return objc_getAssociatedObject(self, MyCustomPropertyKey); if using void* key
    }
    
    // 手动实现 setter 方法
    - (void)setMyCustomProperty:(NSString *)myCustomProperty {
        // 使用 objc_setAssociatedObject 设置关联对象
        // 参数：源对象, key, 值, 关联策略 (对应 @property 的内存管理策略)
        objc_setAssociatedObject(self,
                                 &MyCustomPropertyKey, // key 的地址
                                 myCustomProperty,     // 要关联的值
                                 OBJC_ASSOCIATION_RETAIN_NONATOMIC); // 对应 strong, nonatomic
                                 // 其他策略如: OBJC_ASSOCIATION_COPY_NONATOMIC, OBJC_ASSOCIATION_ASSIGN, etc.
    }
    
    @end
    ```

**总结对比:**

| 特性               | `@property` in `@protocol`                        | `@property` in `category`                                    |
| :----------------- | :------------------------------------------------ | :----------------------------------------------------------- |
| **主要目的**       | 声明**要求** (Contract)，遵循者必须提供访问器方法 | 声明添加到现有类中的**访问器方法**                           |
| **ivar 自动合成?** | **否** (协议本身不提供)                           | **否** (分类不能添加 ivar)                                   |
| **方法自动合成?**  | **否** (协议本身不提供)                           | **否**                                                       |
| **谁负责实现?**    | **遵循协议的类** (通过 ` @property` 或手动实现)   | **分类本身** (必须手动实现 getter/setter)                    |
| **数据存储**       | 由遵循类的 ivar 提供                              | 通常使用**关联对象 (Associated Objects)** 在运行时附加数据   |
| **常见用途**       | 定义代理、数据源等接口中所需的数据属性            | 为现有类添加方便的、状态化的扩展属性 (如给 UIView 加一个标识符) |

因此，虽然语法上都是 `@property`，但在协议和分类中使用它，其背后的含义、限制和实现方式有着本质的区别。