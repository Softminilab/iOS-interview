# Block 相关问题(2)

1. 一个int变量被 __block 修饰与否的区别？block的变量截获
2. 解决循环引用时为什么要用__strong、__weak修饰
3. block在修改NSMutableArray，需不需要添加__block
4. block可以用strong修饰吗

---

## Block是将函数及其执行上下文封装起来的对象。

---



**1. 一个int变量被 \_\_block 修饰与否的区别？block的变量截获**

理解这个问题的关键在于理解Block如何捕获（capture）其外部作用域的变量，以及`__block`修饰符的作用。

*   **变量截获 (Variable Capture):**
    *   默认情况下，当Block被定义时，它会“捕获”其引用的外部局部变量。捕获的方式取决于变量的类型和存储位置。
    *   对于**非静态局部变量**（存储在栈上）：
        *   **基本数据类型 (如 int, float, C结构体等):** Block会捕获这些变量的**值**。也就是说，在Block内部创建了一个这些变量的**副本(copy)**。这个副本在Block创建时就被固定下来了，并且在Block内部是**只读(const)**的。之后外部变量值的任何改变，都不会影响Block内部的这个副本；同样，你也不能在Block内部直接修改这个副本的值（会导致编译错误）。
        *   **对象类型 (指针):** Block会捕获对象指针的**值**，并且根据Block的类型（栈Block或堆Block）和ARC/MRC环境，可能会**持有(retain)**这个对象。你不能在Block内部修改这个指针本身让它指向另一个对象（指针是const），但你可以通过这个指针去修改它所指向的那个对象（如果对象是可变的）。

*   **\_\_block 修饰符的作用:**
    *   `__block` 告诉编译器，对于被它修饰的变量，不要按默认方式捕获。
    *   编译器会将`__block`变量包装在一个特殊的结构体中。这个结构体实例存储在堆上（如果Block被拷贝到堆上）或者栈上。
    *   Block捕获的是指向这个结构体的**指针**。
    *   通过这个结构体指针，Block内部和Block外部的代码都可以访问和**修改**同一个共享的变量存储。

*   **int 变量的区别:**
    *   **没有 `__block` 修饰的 `int`:**
        ```objectivec
        int age = 10;
        void (^myBlock)(void) = ^{
            // 捕获的是 age 的值 (10)
            NSLog(@"Inside block (no __block), age = %d", age);
            // age = 11; // 编译错误！Cannot assign to variable 'age' with const-qualified type 'const int'
        };
        age = 20; // 修改外部 age
        myBlock(); // 输出: Inside block (no __block), age = 10
        NSLog(@"Outside block, age = %d", age); // 输出: Outside block, age = 20
        ```
        **原理:** Block捕获了`age`在定义时的值`10`。这个值在Block内部是常量副本。外部`age`修改为`20`与Block内部的副本无关。

    *   **有 `__block` 修饰的 `int`:**
        ```objectivec
        __block int age = 10;
        void (^myBlock)(void) = ^{
            // 捕获的是指向包含 age 的结构体的指针
            NSLog(@"Inside block (__block), age = %d", age);
            age = 11; // 可以修改！
            NSLog(@"Inside block (__block) after modification, age = %d", age);
        };
        age = 20; // 在 Block 执行前修改外部 (共享的) age
        myBlock();
        // 输出:
        // Inside block (__block), age = 20  (读取到外部修改后的值)
        // Inside block (__block) after modification, age = 11 (内部修改了共享值)
        
        NSLog(@"Outside block, age = %d", age); // 输出: Outside block, age = 11 (外部也能看到Block内部的修改)
        ```
        **原理:** `__block`使得`age`变量被包装起来。Block内外的代码都通过指针访问这个包装后的共享存储。因此，任何一方的修改对另一方都可见，并且Block内部可以修改它。

---

**2. 解决循环引用时为什么要用__strong、__weak修饰**

这是经典的Block导致循环引用的场景及解决方法，通常发生在Block捕获了`self`，而`self`又强引用了这个Block的情况下。

*   **循环引用产生:**
    1.  对象A（比如一个ViewController `self`）有一个Block属性（通常用`copy`修饰，是强引用）。
    2.  这个Block的代码体内部直接或间接引用了对象A（比如使用了`self.propertyName`或实例变量`_ivarName`）。
    3.  当Block被拷贝到堆上时，为了保证其内部引用的对象在Block执行期间有效，它会强引用（retain）这些对象。
    4.  结果：`self` -> `Block` -> `self`，形成了一个强引用循环。两者都无法被释放，导致内存泄漏。

*   **解决方法 (__weak):**
    *   为了打破这个循环，我们需要让Block对`self`的引用变成弱引用(`__weak`)。弱引用不会增加对象的引用计数，也不会阻止对象被释放。
    *   **做法:** 在Block定义之前，创建一个`self`的弱引用指针。
        ```objectivec
        __weak typeof(self) weakSelf = self; // 创建 self 的弱引用
        self.myBlock = ^{
            // Block 内部通过 weakSelf 访问 self 的属性或方法
            // 此时 Block 只持有对 self 的弱引用，打破了循环
            [weakSelf doSomething];
        };
        ```
    *   **原理:** 现在引用链是 `self` ->(strong) `Block` ->(weak) `self`。当外部不再有强引用指向`self`时，`self`可以被正常释放。`self`释放后，它持有的Block也会被释放（如果Block没有被其他对象强引用）。

*   **引入 __strong 的原因:**
    *   仅仅使用`__weak`有时会遇到一个问题：Block的执行通常是异步的。可能在Block开始执行时`weakSelf`还指向有效的`self`对象，但在Block执行过程中（比如执行到一半），`self`对象可能因为其他原因被释放了，导致`weakSelf`变为`nil`。
    *   如果Block的后续代码依赖于`self`对象必须存在才能正确执行（比如需要连续访问`self`的多个属性或方法），那么中途`weakSelf`变`nil`可能会导致逻辑不完整或异常。
    *   `__strong`用于解决这个问题，确保**在Block单次执行的作用域内**，如果`self`在Block开始时还存在，那么它在整个Block执行期间都不会被释放。
    *   **做法 (Weak-Strong Dance):**
        ```objectivec
        __weak typeof(self) weakSelf = self;
        self.myBlock = ^{
            // 在 Block 执行开始时，尝试从 weakSelf 创建一个 strongSelf
            __strong typeof(weakSelf) strongSelf = weakSelf; // 或 __strong typeof(self) strongSelf = weakSelf;
        
            // 关键检查：如果 strongSelf 为 nil，说明 self 已经被释放了，直接返回
            if (!strongSelf) {
                return;
            }
        
            // 在这个 Block 的剩余执行时间内，strongSelf 会持有 self
            // 可以安全地使用 strongSelf，不用担心它在中途变 nil
            [strongSelf doSomething];
            strongSelf.someProperty = @"newValue";
            // ... 其他操作 ...
        
            // 当 Block 执行完毕，strongSelf 变量超出作用域，其对 self 的强引用自动解除
        };
        ```
    *   **原理:**
        1.  `weakSelf`打破循环引用。
        2.  进入Block时，立刻用`__strong`从`weakSelf`创建一个局部强引用`strongSelf`。
        3.  如果`weakSelf`此时已为`nil`（即`self`已释放），`strongSelf`也为`nil`，通过检查提前退出，避免后续操作。
        4.  如果`weakSelf`不为`nil`，`strongSelf`就获得了对`self`的一个强引用。这个强引用只存在于Block的本次执行作用域内。只要`strongSelf`存在，`self`就不会被释放。
        5.  Block执行结束，`strongSelf`销毁，对`self`的临时强引用解除。循环引用并未重新建立。

---

**3. block在修改NSMutableArray，需不需要添加__block**

**通常不需要。**

*   **回顾变量捕获:** 对于对象类型（如`NSMutableArray *`），Block默认捕获的是**指针的值**，并强引用该指针指向的对象（假设Block被拷贝到堆上）。
*   **修改对象 vs 修改指针:**
    *   **修改对象 (Mutating the object):** 你可以通过Block捕获到的指针，调用该指针指向的`NSMutableArray`对象的方法来修改其内容（如`addObject:`, `removeObjectAtIndex:`等）。这是允许的，因为你并没有改变指针本身，只是通过指针操作它指向的内存中的对象。
    *   **修改指针 (Assigning the pointer):** 如果你试图在Block内部让这个指针变量指向一个**全新的** `NSMutableArray` 对象（即赋值操作 `myArray = ...;`），那么默认情况下是不允许的，因为捕获的指针是`const`的。

*   **示例:**
    ```objectivec
    NSMutableArray *myArray = [NSMutableArray arrayWithObjects:@"A", @"B", nil];
    
    void (^modifyArrayBlock)(void) = ^{
        // 不需要 __block 就可以修改数组内容
        [myArray addObject:@"C"];
        NSLog(@"Inside block, array: %@", myArray);
    
        // myArray = [NSMutableArray array]; // 编译错误！Variable is not assignable (missing __block type specifier)
    };
    
    modifyArrayBlock(); // 输出: Inside block, array: (A, B, C)
    NSLog(@"Outside block, array: %@", myArray); // 输出: Outside block, array: (A, B, C)
    
    // 如果确实需要在 Block 内部改变指针指向，则需要 __block
    __block NSMutableArray *blockArray = [NSMutableArray arrayWithObjects:@"X", nil];
    void (^changeArrayBlock)(void) = ^{
        [blockArray addObject:@"Y"]; // 可以修改内容
        blockArray = [NSMutableArray arrayWithObjects:@"New", @"Array", nil]; // 也可以重新赋值指针
        NSLog(@"Inside block (change), array is now: %@", blockArray);
    };
    
    changeArrayBlock(); // 输出: Inside block (change), array is now: (New, Array)
    NSLog(@"Outside block, blockArray is now: %@", blockArray); // 输出: Outside block, blockArray is now: (New, Array)
    
    ```

*   **结论:** 如果你只是想在Block内部**修改`NSMutableArray`的内容**（添加、删除元素等），你**不需要**使用`__block`。只有当你需要在Block内部**给这个`NSMutableArray`指针变量赋一个新的数组对象**时，才需要`__block`修饰符。

---

**4. block可以用strong修饰吗**

**可以，但在ARC下，对于Block类型的属性，更推荐、更标准、更安全的是使用 `copy`。**

*   **Block的存储类型:** Block有三种类型：
    *   `_NSConcreteGlobalBlock` (全局Block): 定义在全局作用域，或不捕获任何自动变量的Block。它存储在数据段，生命周期从程序开始到结束，对其`copy`是空操作。
    *   `_NSConcreteStackBlock` (栈Block): 定义在函数或方法内部，并且捕获了自动变量。它存储在栈上，一旦其定义的函数/方法返回，栈上的Block就会被销毁。
    *   `_NSConcreteMallocBlock` (堆Block): 栈Block通过`copy`操作（或在ARC下某些情况自动）被复制到堆上。堆Block的生命周期由引用计数管理，就像普通Objective-C对象一样。

*   **属性修饰符 (`strong` vs `copy`):**
    *   **`copy`:** 当你把一个Block赋值给`copy`修饰的属性时，系统会确保执行一个`copy`操作。
        *   如果赋值的是一个栈Block，`copy`会将其复制到堆上，属性持有这个堆Block。这是关键！避免了持有栈Block导致野指针的问题。
        *   如果赋值的是一个堆Block，`copy`通常等同于`retain`（增加引用计数）。
        *   如果赋值的是一个全局Block，`copy`是空操作。
        *   **优点:** 保证了属性持有的一定是一个在堆上（或全局区）的、生命周期可控的Block。这是MRC时代的标准做法，也是ARC下最清晰、最能表达意图的做法。
    *   **`strong`:** 当你把一个Block赋值给`strong`修饰的属性时，系统会执行`retain`操作（增加引用计数）。
        *   **在ARC下:** ARC做了很多优化。当你把一个**栈Block**赋值给一个`strong`指针类型的变量或属性时，**ARC通常会自动执行`copy`操作**，将栈Block拷贝到堆上。所以，在很多情况下，使用`strong`也能正常工作。
        *   **潜在风险/不清晰:**
            1.  虽然ARC通常会`copy`，但这是一种隐式行为。使用`copy`则明确表达了“我需要一个堆上的副本”的意图。
            2.  依赖隐式行为可能在某些非常罕见的边界情况或编译器优化下产生非预期结果（尽管现代ARC下已很少见）。
            3.  从代码可读性和维护性角度看，`copy`更清晰地说明了你在处理一个Block，并且需要管理其从栈到堆的转换。

*   **结论:**
    *   在**MRC**下，Block属性**必须**使用`copy`，否则如果赋值的是栈Block，会导致悬垂指针。
    *   在**ARC**下，虽然使用`strong`通常也能工作（因为ARC的自动`copy`），但**最佳实践、官方推荐、以及为了代码清晰性和绝对安全，仍然是使用 `copy` 来修饰Block类型的属性**。

    ```objectivec
    // 推荐的方式 (ARC & MRC)
    @property (nonatomic, copy) void (^myBlockCopy)(void);
    
    // 在 ARC 下通常也能工作，但不推荐
    @property (nonatomic, strong) void (^myBlockStrong)(void);
    ```

希望这些解释和示例能帮助你更好地理解Block的相关机制！如果还有其他问题，请继续提出。
