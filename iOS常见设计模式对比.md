# iOS常见设计模式对比

## 创建型模式

### 单例模式 (Singleton)

**适用场景**：需要全局唯一访问点的对象 **特点**：

- 确保类只有一个实例存在
- 提供全局访问点
- 延迟初始化

**Swift实现示例**：

```swift
class NetworkManager {
    static let shared = NetworkManager()
    private init() {}
    
    func request() {
        // 网络请求实现
    }
}
```

### 工厂模式 (Factory)

**适用场景**：创建对象时不指定具体类，根据条件动态创建不同子类 **特点**：

- 将对象的创建与使用分离
- 隐藏具体类的实例化过程
- 便于添加新产品类型

**实现示例**：

```swift
protocol Button {
    func render()
}

class IOSButton: Button {
    func render() { print("渲染iOS风格按钮") }
}

class AndroidButton: Button {
    func render() { print("渲染Android风格按钮") }
}

class ButtonFactory {
    static func createButton(platform: String) -> Button {
        switch platform {
        case "iOS": return IOSButton()
        case "Android": return AndroidButton()
        default: return IOSButton()
        }
    }
}
```

### 构建者模式 (Builder)

**适用场景**：创建复杂对象，特别是有多个配置选项时 **特点**：

- 分步骤构建复杂对象
- 同样的构建过程可以创建不同的表示
- 隐藏内部表示

**实现示例**：

```swift
class AlertBuilder {
    private var title: String = ""
    private var message: String = ""
    private var buttons: [String] = []
    
    func setTitle(_ title: String) -> AlertBuilder {
        self.title = title
        return self
    }
    
    func setMessage(_ message: String) -> AlertBuilder {
        self.message = message
        return self
    }
    
    func addButton(_ title: String) -> AlertBuilder {
        self.buttons.append(title)
        return self
    }
    
    func build() -> UIAlertController {
        let alert = UIAlertController(title: title, message: message, preferredStyle: .alert)
        for buttonTitle in buttons {
            alert.addAction(UIAlertAction(title: buttonTitle, style: .default))
        }
        return alert
    }
}
```

## 结构型模式

### 适配器模式 (Adapter)

**适用场景**：使不兼容的接口能够一起工作 **特点**：

- 将一个类的接口转换成客户期望的另一个接口
- 让原本不兼容的类能够合作

**实现示例**：

```swift
protocol ModernAPI {
    func newRequest()
}

class LegacyService {
    func oldRequest() {
        print("执行旧API请求")
    }
}

class APIAdapter: ModernAPI {
    private let legacyService: LegacyService
    
    init(legacyService: LegacyService) {
        self.legacyService = legacyService
    }
    
    func newRequest() {
        legacyService.oldRequest()
    }
}
```

### 装饰器模式 (Decorator)

**适用场景**：动态地向对象添加职责，比子类更灵活 **特点**：

- 不改变原类的情况下扩展功能
- 遵循开闭原则
- 可以层层叠加功能

**实现示例**：

```swift
protocol Coffee {
    func cost() -> Double
    func description() -> String
}

class SimpleCoffee: Coffee {
    func cost() -> Double { return 5.0 }
    func description() -> String { return "简单咖啡" }
}

class MilkDecorator: Coffee {
    private let coffee: Coffee
    
    init(coffee: Coffee) {
        self.coffee = coffee
    }
    
    func cost() -> Double {
        return coffee.cost() + 1.0
    }
    
    func description() -> String {
        return coffee.description() + ", 加牛奶"
    }
}
```

### 组合模式 (Composite)

**适用场景**：将对象组合成树形结构表示"部分-整体"层次结构 **特点**：

- 统一对待单个对象和组合对象
- 能够递归组合

**实现示例**：

```swift
protocol FileSystemItem {
    var name: String { get }
    func size() -> Int
}

class File: FileSystemItem {
    var name: String
    private var fileSize: Int
    
    init(name: String, size: Int) {
        self.name = name
        self.fileSize = size
    }
    
    func size() -> Int {
        return fileSize
    }
}

class Directory: FileSystemItem {
    var name: String
    private var contents: [FileSystemItem] = []
    
    init(name: String) {
        self.name = name
    }
    
    func add(_ item: FileSystemItem) {
        contents.append(item)
    }
    
    func size() -> Int {
        return contents.reduce(0) { $0 + $1.size() }
    }
}
```

## 行为型模式

### 观察者模式 (Observer)

**适用场景**：对象状态变化时通知其他对象 **特点**：

- 定义了一对多的依赖关系
- 状态变化自动通知所有依赖对象
- 松耦合设计

**实现示例**：

```swift
protocol Observer: AnyObject {
    func update(with data: String)
}

class Subject {
    private var observers: [Observer] = []
    
    func attach(_ observer: Observer) {
        observers.append(observer)
    }
    
    func detach(_ observer: Observer) {
        if let index = observers.firstIndex(where: { $0 === observer }) {
            observers.remove(at: index)
        }
    }
    
    func notify(with data: String) {
        observers.forEach { $0.update(with: data) }
    }
}
```

### 策略模式 (Strategy)

**适用场景**：定义一系列算法，使它们可以互相替换 **特点**：

- 将算法封装在独立的类中
- 运行时选择不同算法
- 避免使用条件语句

**实现示例**：

```swift
protocol SortStrategy {
    func sort<T: Comparable>(_ items: [T]) -> [T]
}

class QuickSort: SortStrategy {
    func sort<T: Comparable>(_ items: [T]) -> [T] {
        // 快速排序实现
        return items.sorted()
    }
}

class MergeSort: SortStrategy {
    func sort<T: Comparable>(_ items: [T]) -> [T] {
        // 归并排序实现
        return items.sorted()
    }
}

class SortContext {
    private var strategy: SortStrategy
    
    init(strategy: SortStrategy) {
        self.strategy = strategy
    }
    
    func setStrategy(_ strategy: SortStrategy) {
        self.strategy = strategy
    }
    
    func executeStrategy<T: Comparable>(_ items: [T]) -> [T] {
        return strategy.sort(items)
    }
}
```

### 命令模式 (Command)

**适用场景**：将请求封装成对象，支持撤销/重做操作 **特点**：

- 将请求发送者和接收者解耦
- 可以参数化和队列化请求
- 支持可撤销操作

**实现示例**：

```swift
protocol Command {
    func execute()
    func undo()
}

class Light {
    func turnOn() { print("灯打开") }
    func turnOff() { print("灯关闭") }
}

class LightOnCommand: Command {
    private let light: Light
    
    init(light: Light) {
        self.light = light
    }
    
    func execute() {
        light.turnOn()
    }
    
    func undo() {
        light.turnOff()
    }
}

class RemoteControl {
    private var command: Command?
    
    func setCommand(_ command: Command) {
        self.command = command
    }
    
    func pressButton() {
        command?.execute()
    }
    
    func pressUndoButton() {
        command?.undo()
    }
}
```

## 总结对比

| 设计模式 | 主要用途           | iOS中常见应用                                 |
| -------- | ------------------ | --------------------------------------------- |
| 单例     | 全局唯一实例       | URLSession.shared, FileManager.default        |
| 工厂     | 对象创建抽象化     | UILabel/UIButton创建定制控件                  |
| 构建者   | 分步构建复杂对象   | NSAttributedString构建, UIAlertController配置 |
| 适配器   | 兼容不同接口       | 第三方SDK集成, 新旧API兼容                    |
| 装饰器   | 动态扩展功能       | UIView的category扩展, 下拉刷新组件            |
| 组合     | 树形结构管理       | UIView层次结构, 复杂表单设计                  |
| 观察者   | 状态变化通知       | NotificationCenter, KVO, 数据绑定             |
| 策略     | 算法族动态切换     | 多种排序方式, 不同网络请求策略                |
| 命令     | 请求封装, 操作队列 | Target-Action模式, 撤销重做功能               |

选择合适的设计模式应根据具体业务场景和问题复杂度，避免过度设计。



- **工厂方法 (Factory Method):** 专注于创建**单一类型**的产品对象，将创建哪种具体产品的决定**延迟到子类**。
- **抽象工厂 (Abstract Factory):** 专注于创建**一族（多个）相关的或相互依赖的产品对象**，提供一个接口来创建这个产品族中的所有对象，而**无需指定它们的具体类**。