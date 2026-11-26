# iOS 内存与性能调优知识体系（新手友好版）

> 覆盖：内存与性能的典型业务场景、内存布局、ARC、strong/weak/unowned、循环引用、autoreleasepool、值类型与写时复制、内存压力、Memory Graph / Allocations / Leaks / Zombies、性能指标、Instruments 模板、启动优化、卡顿优化、内存优化、CPU/耗电、包体积、性能回归测试、从零实操 SOP、报错排查与优化优先级。
> 适用版本：Xcode 27.2（2026-10-29 发布，随附 Swift 6.4.2）、iOS 27 SDK；Xcode 27 仅支持 Apple 芯片 Mac，需要 macOS 26.6 或更高版本，并支持 iOS 17 及以上设备调试。菜单路径以 Xcode 27 为准；旧版本入口可能不同，找不到时用 Xcode 搜索或 `Help → Search`。
> 文档边界：以“内存 → 性能”为主线，讲对象生命周期、泄漏排查、Instruments 测量与常见优化；语言选型、Objective-C Runtime、消息转发、Swift 派发与 C/C++ 互操作见姊妹篇《iOS 开发语言与运行时知识体系》。签名、证书、ipa、上架流程见《iOS 签名与出包知识体系》。
> 阅读方式：每个概念先讲“是什么（带比喻）”，再讲“为什么需要它”，最后讲“怎么用/怎么排查”。关键词首次出现标注中英文。
> 信息核验日期：2026-11-21。版本、菜单路径等动态信息以官方页面为准（文末附链接）。

---

## 目录

- 0. 先用一组比喻建立全局
- 一、内存管理：谁在什么时候被释放
- 二、性能调优：先测量，再动手
- 三、从零到跑通：内存与性能实操 SOP
- 四、常见报错与排查
- 五、业务与面试视角
- 附录 A：实操入口地图
- 附录 B：排障“动手顺序”速查
- 官方与参考链接

---

## 0. 先用一组比喻建立全局

把内存与性能调优想象成“管理一间仓库”：

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 自动引用计数 | ARC（Automatic Reference Counting） | 自动记账员 | 引用计数归零就释放对象 |
| 强引用 | strong | 抓着不放的手 | 持有对象，计数 +1 |
| 弱引用 | weak | 借条 | 不持有，对象释放后自动变 nil |
| 无主引用 | unowned | 默认对方一直在 | 不持有，释放后访问会崩溃 |
| 循环引用 | Retain Cycle / Strong Reference Cycle | 两个人互相抓着不放 | 计数永远不归零，内存下不来 |
| 自动释放池 | Autorelease Pool | 临时托盘 | 批量延迟释放临时对象 |
| 写时复制 | Copy-on-Write（COW） | 借书证 | 复制时共享，修改时才真正复制 |
| 内存图 | Memory Graph | 关系图 | 看谁在引用谁 |
| 分配 | Allocations | 仓库盘点 | 看内存分配和增长 |
| 泄漏 | Leaks | 漏水检测 | 找不可达但仍占内存的对象 |
| 卡顿 | Hitch | 上菜慢 | 一帧没画完 |
| 启动时间 | Launch Time | 开店准备 | 点图标到第一帧 |
| 性能分析工具 | Instruments | 体检仪器 | 采样、记录、定位性能问题 |

串起来一句话：**ARC 管对象生死，weak/unowned 断环，Memory Graph 找持有关系，Instruments 量出哪里慢。**

### 0.1 这套体系在什么业务场景里用

内存和性能调优不是“高级工程师才做的优化题”，而是很多业务功能能不能用、好不好用的底线。下面这些场景，新手进项目后很快就会遇到：

| 业务场景 | 典型现象 | 背后要用的知识 | 对应章节 |
|---|---|---|---|
| 信息流、电商、社交列表 | 越滑越卡，图片一多内存飙升 | Cell 复用（列表单元格重复使用）、图片降采样（大图缩到显示尺寸）、离屏渲染（先画到离屏缓冲区再合成）、autoreleasepool | 1.5、2.6、2.7 |
| 详情页、弹窗、页面反复进出 | 退出页面后内存不降，`deinit` 不打印 | 循环引用、delegate / 闭包 / Timer、Memory Graph | 1.4、1.8、4.2 |
| 启动与首屏 | 点图标后白屏久，首屏内容出不来 | 启动阶段任务、动态库、主线程 I/O、App Launch（启动模板） | 2.5 |
| 直播、音视频、相机 | 内存峰值高，切后台回来崩溃 | buffer 生命周期、autoreleasepool、内存警告、Jetsam（系统内存不足时杀进程） | 1.5、1.7、2.7、4.5 |
| 地图、长图、PDF、大文件 | 打开大文件就 OOM（Out Of Memory，内存耗尽），内存曲线尖峰 | COW、分块 / 降采样、峰值控制 | 1.6、2.7 |
| 游戏、Flutter、React Native、Unity 混合工程 | 原生内存（native）和托管内存（managed）混在一起，卡顿难定位 | 内存布局、跨语言对象、纹理 / 资源、Instruments | 1.1、2.3、2.7 |
| 线上版本与多人协作 | 偶发 OOM / 卡顿 / 耗电，改完又退化 | MetricKit（苹果的性能指标框架）、Organizer（Xcode 的归档与线上指标窗口）、性能基线、回归测试 | 2.2、2.10、3.5 |

游戏、Flutter、RN 这类混合工程的跨语言接入细节，见姊妹篇 1.7；本文只保留内存与性能的通用排查线。

一个新手最常接到的任务长这样：产品说“列表越滑越卡，进出详情页几次后内存不降”。你不需要一上来把所有 Instruments 模板学完，按这条线走就够了：

1. 用 Memory Graph 看页面退出后谁还持有它（1.8、3.3）。
2. 用 Leaks / Allocations（泄漏 / 分配模板）判断是泄漏、缓存还是峰值（2.3、3.4）。
3. 用 Time Profiler（CPU 采样模板）看主线程谁在耗 CPU（2.6、3.4）。
4. 改完用同样的操作复测，记录基线（2.10、3.6）。

所以这套体系的目标不是“背工具”，而是让你能回答三个问题：**对象什么时候释放？内存涨是泄漏还是峰值？卡顿是主线程、渲染还是 I/O？** 能回答这三个问题，你就能把大多数内存 / 性能问题定位到可改的代码。

---

## 一、内存管理：谁在什么时候被释放

### 1.1 先看内存布局

一个 iOS 进程的内存，粗略可以分成四块：

```
┌─────────────────────────────┐
│ 代码区 Code                 │ 机器指令，只读
├─────────────────────────────┤
│ 全局区 Global / Static      │ 单例、静态变量，跟随 App 生命周期
├─────────────────────────────┤
│ 栈 Stack                    │ 函数局部变量，自动进出，速度快
├─────────────────────────────┤
│ 堆 Heap                     │ 对象，ARC 管理，泄漏通常发生在这里
└─────────────────────────────┘
```

值类型（`struct`、`enum`）通常直接存放在变量所在的位置：局部变量在栈上，作为对象属性时就在堆上的对象里。类实例（`class`）放在堆上，用引用计数管理生命周期。ARC 只管类实例，不管结构体和枚举。

### 1.2 ARC 是什么：自动记账员

ARC（Automatic Reference Counting，自动引用计数）不是垃圾回收器（Garbage Collector）。它不会定期扫描整个堆，而是由编译器在合适的位置插入“引用计数 +1 / -1”的操作。每当一个强引用指向对象，计数加一；强引用断开，计数减一；计数归零，对象先执行 `deinit`，再释放内存。Objective-C 和 Swift 都使用 ARC；手动 `retain` / `release` 只会出现在关闭 ARC 的老代码里。

```swift
final class Session {
    deinit { print("Session deinit") }
}

var session: Session? = Session()   // 计数 1
var backup = session                // 计数 2
session = nil                       // 计数 1
backup = nil                        // 计数 0，打印 deinit
```

`deinit` 是类实例的析构函数，适合放清理逻辑：移除观察者、取消定时器、关闭文件。不要依赖 `deinit` 的执行顺序，也不要在里面做耗时操作。Objective-C 里对应的析构方法叫 `dealloc`；ARC 下不要手动调用 `[super dealloc]`。

### 1.3 strong、weak、unowned 怎么选

| 修饰符 | 是否增加引用计数 | 对象释放后 | 类型要求 | 适用场景 |
|---|---|---|---|---|
| `strong`（默认） | 是 | 保持对象存活 | 任意 | 拥有关系，例如父对象持有子对象 |
| `weak` | 否 | 自动变成 `nil` | 必须是 `var` 的可选类型，且是类类型 | 可能先于自己释放的对象，例如 delegate |
| `unowned` | 否 | 访问会触发运行时陷阱（崩溃） | 非可选，且是类类型 | 生命周期相同或更长的对象，例如客户与订单 |

默认用 `strong`。只有当你明确要“断开环”或“不拥有对方”时，才换成 `weak` 或 `unowned`。

Objective-C 代码里还会看到这些属性修饰符，别和 Swift 的关键词混在一起：

| 修饰符 | 含义 | 常用场景 |
|---|---|---|
| `strong` / `retain` | 强引用，计数 +1 | 对象属性 |
| `copy` | 赋值时复制一份 | `NSString`、`NSArray`、Block |
| `assign` | 不改变引用计数 | 基本类型（`NSInteger`、`BOOL`）；对象类型慎用 |
| `weak` | 弱引用，释放后置 `nil` | delegate、避免循环引用 |
| `unsafe_unretained` | 弱引用但不置 `nil` | 老代码；新手别用 |
| `nonatomic` | 不做原子锁 | App 内绝大多数属性 |
| `atomic` | 保证单次读写的原子性 | 不等于线程安全；有性能和死锁风险 |

`weak` 是安全的选择：对象没了，引用变 `nil`，你最多多做一次可选绑定。`unowned` 更快、语法更干净，但它假设“我访问你的时候你一定还活着”。这个假设一旦不成立，就是崩溃。新手优先用 `weak`，等你能明确证明生命周期关系时再用 `unowned`。

### 1.4 循环引用：最常见的五种现场

循环引用（Retain Cycle）是两个或多个对象互相强引用，导致引用计数永远不归零。下面五种写法最容易踩：

| 场景 | 错误写法 | 正确做法 |
|---|---|---|
| 代理（Delegate） | `var delegate: MyDelegate?` | `weak var delegate: MyDelegate?` |
| 闭包捕获 self | 闭包里直接用 `self` | `[weak self]`（确认生命周期后再考虑 `unowned`） |
| 定时器 Timer | Timer 强引用 target/闭包，target 又持有 Timer | `[weak self]`，并在退出时 `invalidate()` |
| 父子对象 | 父强引用子，子也强引用父 | 子对父用 `weak` |
| 通知/回调 | 观察者闭包强引用 self，token 又被 self 持有 | `[weak self]`，或及时移除观察者 |

闭包这一段最容易误会。闭包捕获 `self` 不一定会泄漏，只有当 `self` 也持有这个闭包（直接或间接）时，才形成环。代理场景还有一个前提：协议要继承 `AnyObject`（Swift 里写 `protocol MyDelegate: AnyObject`），`weak var delegate` 才能编译通过。

```swift
final class DetailViewController: UIViewController {
    var onFinish: (() -> Void)?

    override func viewDidLoad() {
        super.viewDidLoad()
        onFinish = { [weak self] in
            self?.dismiss(animated: true)
        }
    }

    deinit { print("DetailViewController deinit") }
}
```

Timer 的坑更隐蔽，因为 RunLoop（事件循环，负责分发触摸、定时器等事件）会持有 Timer，Timer 又持有闭包或 target：

```swift
final class Ticker {
    private var timer: Timer?

    func start() {
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            self?.tick()
        }
    }

    func stop() {
        timer?.invalidate()
        timer = nil
    }

    private func tick() { }
}
```

注意 `stop()` 里要先 `invalidate()` 再置空。只把 `timer` 设为 `nil`，RunLoop 仍然持有它，计时器会继续跑。

### 1.5 autoreleasepool：临时对象的托盘

Objective-C 里，很多方法返回的是“自动释放对象”（autoreleased object）：对象先被放进当前的自动释放池，等池子排空时才释放。主线程的 RunLoop 每次事件循环会排空一次池子，所以日常代码不用管。

但如果你在一个循环里创建大量临时对象，池子要等到循环结束才排空，内存峰值就会很高。这时可以手动加一个自动释放池（Autorelease Pool）：

```swift
for _ in 0..<10_000 {
    autoreleasepool {
        let data = NSMutableData(length: 1024)   // 模拟大量临时对象
        _ = data?.length
    }
}
```

每轮循环结束，池子排空，临时对象及时释放。Objective-C 里写 `@autoreleasepool { ... }`，Swift 里写 `autoreleasepool { ... }`，作用一样。

不要给每个小函数都套一层。自动释放池本身有开销，只在“循环里大量创建临时对象”或“后台线程没有自动池”时使用。

### 1.6 值类型与写时复制

Swift 的结构体、枚举是值类型，赋值时复制。数组、字典、集合、字符串这些标准库类型，内部用了写时复制（Copy-on-Write，COW）：赋值时只复制一个引用，真正修改时才复制底层存储。

```swift
var a = Array(repeating: 0, count: 1000)
var b = a          // 此时没有复制底层数组
b[0] = 1           // 真正修改，底层存储在这里复制
```

COW 让“传值”看起来很便宜，但第一次修改仍有复制成本。大量修改大数组时，注意不要在循环里反复触发复制。

结构体和类怎么选，可以用一张表判断：

| 选择 | 适合的情况 |
|---|---|
| `struct` | 数据小、以值语义为主、不需要继承、不需要引用身份 |
| `class` | 需要共享同一份状态、需要继承、需要 Objective-C 互操作、需要 `deinit` |

### 1.7 内存压力：系统什么时候会杀 App

Xcode 的 Debug 导航器里有内存计量条。绿色是正常，黄色表示接近警告线，红色表示随时可能被系统终止。这里的“终止”不是崩溃，而是系统在内存紧张时杀掉进程，日志里通常能看到 Jetsam（系统内存不足时的终止机制）相关记录。

模拟器的内存计量条几乎一直是绿色，因为 macOS 不会像 iOS 真机那样发内存警告，也不会执行低内存终止。**不要用模拟器的绿色条判断内存安全**，内存问题尽量在真机上验证。

收到内存警告时，应该释放可以重建的缓存和临时数据。`NSCache` 会在系统内存紧张时自动淘汰对象，适合做图片/数据缓存；普通 `Dictionary` 不会自动清理。

```swift
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    imageCache.removeAllObjects()   // imageCache 是你的图片缓存（NSCache）
}
```

App 进入后台后也可能被系统回收。不要把“用户切回来时状态还在”当成理所当然，重要状态要持久化或支持恢复。

### 1.8 怎么找泄漏：一条可执行的排查线

先加 `deinit` 打印，这是成本最低的验证：

```swift
deinit { print("\(Self.self) deinit") }
```

打开页面、返回、再打开、再返回。如果 `deinit` 一直不打印，说明对象还被谁持有。接下来按顺序做：

1. **Memory Graph。** 运行 App，点击调试栏的 `Debug Memory Graph` 按钮。左侧会按类型列出对象，选中可疑对象，右侧能看到谁在引用它。顺着箭头找环，通常一眼就能看到“A 持有 B，B 又持有 A”。
2. **打开 Malloc Stack。** `Product → Scheme → Edit Scheme → Run → Diagnostics`，勾选 `Malloc Stack`（记录对象分配调用栈的诊断选项）。再生成内存图，选中对象就能看到分配时的调用栈，定位到具体代码。
3. **Allocations 看增长。** `Product → Profile`（`⌘I`），选 `Allocations`，点 `Mark Generation`（标记世代，给当前内存分配打一个分隔点），重复一次进入/退出页面的操作，再点一次。看这两代之间哪些对象在增长。
4. **Leaks 看不可达对象。** 同样在 Instruments 里选 `Leaks`。它能找出“没有任何强引用、但仍占着内存”的对象。注意：如果环还挂在一个活着的根对象上（比如一个没释放的 ViewController），Leaks 不一定报，Memory Graph 更直观。
5. **修完复测。** 不要只看一次红条消失，要重复操作 3~5 次，确认内存曲线能回到基线。

### 1.9 几个常见误区

**“用了 ARC 就不会泄漏。”** ARC 只负责引用计数归零时释放，环是自己造的，ARC 解不开。

**“`weak` 一定安全。”** `weak` 是可选类型，用之前要解包；大量 `weak` 引用也有运行时维护成本，不要滥用。

**“`unowned` 更快，所以都用 `unowned`。”** 它省掉的是可选检查，代价是生命周期判断错误时直接崩溃。新手默认 `weak`。

**“`deinit` 不打印就是泄漏。”** 也可能只是还有别的持有者，或者对象被自动释放池延迟释放。用 Memory Graph 确认，不要只凭打印下结论。

---

## 二、性能调优：先测量，再动手

### 2.1 性能优化的正确姿势

性能优化通常是一个循环：收集问题 → 测量定位 → 只改一个点 → 实现 → 观察是否变好。这个顺序很重要，因为新手最常见的错误是“凭感觉改”：把 `for` 循环改成 `while`、给所有视图加 `shouldRasterize`、把 OC 重写成 C++，结果热点根本不在那里。

三条原则：

1. **在真机上测。** 模拟器用 Mac 的 CPU 和内存，跑分和真机差距很大，GPU、内存压力、发热都不一样。
2. **一次只改一个变量。** 同时改五处，变快了也不知道是哪一处起的作用。
3. **保留改前和改后的数据。** Instruments 的 trace、Xcode Organizer 的指标、XCTest 的耗时，都可以作为对比基线。

### 2.2 八个核心指标

| 指标 | 英文 | 用户感受 | 常用工具 |
|---|---|---|---|
| 启动时间 | Launch Time | 点图标后多久看到第一帧 | Organizer、App Launch、Signpost |
| 卡顿 / 挂起 | Hitch / Hang | 滑动掉帧、点击没反应 | Animation Hitches、Time Profiler |
| 内存 | Memory | 切后台会不会被杀、会不会越用越卡 | Memory Graph、Allocations、Leaks |
| CPU | CPU Usage | 发热、耗电、掉帧 | Time Profiler、CPU Profiler |
| 耗电 | Energy | 手机烫、掉电快 | Energy Log |
| 磁盘写入 | Disk Writes | 启动慢、设备寿命 | File Activity |
| 网络 | Network | 流量、等待 | Network 模板 |
| 包体积 | App Size | 下载快慢、占空间 | App Store Connect、Xcode Archive |

### 2.3 Instruments 模板怎么选

打开方式：`Xcode → Open Developer Tool → Instruments`，或者在 Xcode 里 `Product → Profile`（`⌘I`）。

| 模板 | 用来查什么 | 什么时候选 |
|---|---|---|
| Time Profiler | CPU 采样，找出最耗时的调用栈 | 卡顿、启动慢、CPU 高 |
| CPU Profiler | CPU 性能分析，区分核心/非核心事件 | 需要更细的 CPU 归因 |
| Allocations | 堆内存分配、增长、世代 | 内存持续上涨 |
| Leaks | 不可达的泄漏对象 | 怀疑内存泄漏 |
| Zombies（僵尸对象） | 已释放对象被再次访问 | `EXC_BAD_ACCESS` |
| Animation Hitches | 滚动/动画卡顿 | 列表滑动掉帧 |
| Energy Log | 耗电 | 发热、掉电快 |
| File Activity | 文件读写 | 启动/列表卡在磁盘 |
| Network | 网络请求 | 请求多、等待长 |
| App Launch | 启动阶段拆解 | 冷启动优化 |
| System Trace | 线程调度、锁、系统调用 | 多线程卡顿、优先级反转 |

新手先掌握三个：**Time Profiler 找 CPU 热点，Allocations 看内存增长，Leaks 找泄漏。** 其余按症状选。

### 2.4 第一次用 Instruments 的完整操作

1. 用真机连接 Mac，在 Xcode 顶部选择你的设备。
2. `Product → Scheme → Edit Scheme → Profile`，确认 `Build Configuration` 是 `Release`。Profile 默认用 Release，但改过 scheme 的话要检查。
3. `Product → Profile`（`⌘I`），选择模板，点左上角红色 `Record`。
4. 在 App 里重复你要调查的操作 10~30 秒。时间太短采样不够，太长数据难读。
5. 点 `Stop`，在时间轴上框选可疑区间。
6. 打开右下角的 `Call Tree`（调用树）选项，按需勾选：
   - `Separate by Thread`：把不同线程分开，先看主线程。
   - `Invert Call Tree`：按“最耗时的方法”聚合，而不是按调用顺序展开。
   - `Hide System Libraries`：隐藏系统库，只看自己的代码。
   - `Flatten Recursion`：把递归调用合并成一行。
7. 双击耗时最高的方法，跳到源码。看不到源码时，检查 dSYM（调试符号文件，把崩溃地址还原成代码行）和 Build Settings 里的调试符号。
8. 改完后，用同样的操作再录一次，对比改前改后的耗时和调用次数。

### 2.5 启动优化

启动时间指“用户点图标”到“App 画出第一帧”的时间。它分成两段：

**pre-main（main 函数之前）**：dyld（动态链接器，负责加载动态库）加载动态库、绑定符号、执行 `+load`、跑静态初始化。优化手段是减少动态库数量、合并静态库、去掉不必要的 `+load`、把全局对象的构造推迟到真正使用时。

**post-main（main 函数之后）**：`application(_:didFinishLaunchingWithOptions:)`、首页构建、网络请求、数据库打开、图片解码。优化手段是懒加载、异步化、减少首屏视图层级、避免在主线程做同步 I/O。

测量方式有三种：

- `Window → Organizer → Metrics → Launch Time`：看线上用户的真实启动数据，需要 App 已经通过 App Store 分发。
- Instruments 的 `App Launch` 模板：本地拆解启动阶段。
- `OSSignposter`（苹果的统一日志/打点 API）打点：在关键阶段埋点，然后在 Instruments 里看区间。

```swift
import os

let signposter = OSSignposter(subsystem: "com.example.app", category: "launch")
let state = signposter.beginInterval("LoadHome")
// 这里放你怀疑耗时的启动逻辑
signposter.endInterval("LoadHome", state)
```

启动阶段还有一条硬约束：启动屏幕（Launch Screen）要尽快出现，主线程不能做重活。iOS 的 watchdog（看门狗）会终止启动时间过长的 App，用户看到的就是“点开闪退”。

### 2.6 卡顿优化

屏幕刷新是有节奏的：60Hz 屏幕一帧约 16.7ms，120Hz ProMotion（高刷新率）屏幕一帧约 8.3ms。主线程没在这一帧内完成布局、绘制、提交，就会掉帧，用户感觉“不跟手”。

常见原因：

- 主线程做同步网络请求、读大文件、解析大 JSON。
- 图片解码、缩放、圆角裁剪集中在滚动时做。
- 视图层级太深，Auto Layout（自动布局）约束过多。
- 文本排版、富文本计算量大。
- 离屏渲染（Offscreen Rendering）：圆角 + 裁剪、阴影、遮罩等组合可能触发额外的渲染通道。

排查顺序：

1. 用 `Animation Hitches` 模板复现滚动，先看卡顿发生在哪一帧。
2. 用 `Time Profiler` 看主线程热点，确认是 CPU 计算、I/O 还是锁等待。
3. 用 `Main Thread Checker` 确认没有后台线程碰 UI。
4. 在 `Debug → View Debugging` 里捕获视图层级，再打开渲染选项中的 `Color Offscreen-Rendered` 检查离屏渲染（不同 Xcode 版本入口略有差异）。

优化手段：把网络/解析/解码挪到后台，图片预解码或降采样，减少视图层级，缓存布局结果，避免在 `cellForRowAt` 里做重计算。SwiftUI 的 `body` 会被频繁调用，不要在 `body` 里做网络请求或大计算。

### 2.7 内存优化

先看峰值，不要只看平均值。图片是 iOS 内存的大头：一张 4000×3000 的图片解码后可能占几十 MB。常见做法：

- 用 `UIImage` 的降采样（downsampling）按显示尺寸解码，而不是加载原图再缩放。
- 列表图片用缓存（`NSCache`），并在内存警告时清理。
- 大文件用流式读取，不要一次性读进内存。
- 长循环里用 `autoreleasepool` 控制临时对象峰值。
- 页面退出时断开代理、取消请求、移除观察者。

排查时用 Memory Graph 看“谁持有谁”，用 Allocations 的世代对比看“这次操作新增了什么”，用 Leaks 看“有没有完全不可达的对象”。

### 2.8 CPU 与耗电

CPU 高不一定卡，但一定费电。常见耗电源：定时器太频繁、定位精度过高、网络轮询、后台任务、频繁磁盘写入、大量图片解码。

优化手段：

- 定时器设 `tolerance`，允许系统合并唤醒。
- 定位按需降精度，用完就停。
- 网络请求合并、缓存、退避重试。
- 后台任务用系统提供的 API，不要常驻线程空转。
- 用 `Energy Log` 看哪个模块耗电，用 `Time Profiler` 看哪个函数占 CPU。

### 2.9 包体积

包体积影响下载转化和安装体验。常用手段：

- 图片用 Asset Catalog 管理，按设备输出合适尺寸；能用矢量（PDF/符号）就用矢量。
- 删除未使用的图片、字体、音频、旧代码。
- 大资源用 On-Demand Resources（按需资源）延迟下载。
- 动态库数量控制住，能静态链接的静态链接。
- 用 App Thinning（应用瘦身）让用户只下载自己设备需要的切片。

Bitcode（苹果的中间代码格式）从 Xcode 14 起已废弃，App Store 不再接受 Xcode 14 提交的 bitcode，Xcode 也不再默认构建 bitcode。看到 `Building with bitcode is deprecated` 的警告，把 `Build Settings → Enable Bitcode` 设为 `No` 即可，不需要再为 bitcode 做适配。报错处理见姊妹篇《iOS 开发语言与运行时知识体系》6.4 节。

### 2.10 性能回归测试

一次优化不算完，要防止以后改回去。XCTest（Xcode 自带的测试框架）自带性能测试：

```swift
func testParserPerformance() {
    let parser = Parser()          // 换成你的类型
    measure {
        _ = parser.parse(sampleData)   // sampleData 换成你的测试数据
    }
}
```

`measure` 会跑多次取平均，并在测试结果里记录基线。超过基线阈值时，CI 可以报警。

线上数据用 MetricKit（苹果的性能指标框架）收集。它按天返回启动时间、卡顿、内存、耗电等指标的直方图，适合看真实用户分布，而不是实验室里的单次采样。

### 2.11 症状到工具的一张表

| 症状 | 先看什么 | 再看什么 |
|---|---|---|
| 启动慢 | Organizer → Launch Time | App Launch、Signpost |
| 滑动掉帧 | Animation Hitches | Time Profiler、View Debugging |
| 点击没反应 | Time Profiler 主线程 | Main Thread Checker |
| 内存越用越多 | Memory Graph | Allocations 世代、Leaks |
| 偶发崩溃 | Crash Log、Zombies | Memory Graph、Address Sanitizer |
| 手机发热 | Energy Log | Time Profiler、Network |
| 包太大 | App Store Connect 体积报告 | Asset Catalog、On-Demand Resources |

### 2.12 性能优化优先级

新手容易一上来就抠微优化。更划算的顺序通常是：

1. **先消灭主线程阻塞。** 同步网络、同步读文件、大 JSON 解析，收益最大。
2. **再处理图片。** 降采样、预解码、缓存，内存和卡顿一起改善。
3. **再减少视图层级和重复计算。** 列表复用、布局缓存、避免 `body` 里做重活。
4. **最后才是算法和语言级优化。** 热点明确后再考虑 C++、内联、静态派发。

---

## 三、从零到跑通：内存与性能实操 SOP

> 这一章是“执行线”，前面的章节是“知识点”。按顺序做一遍，你会把循环引用、Memory Graph、Instruments 都亲手碰一次。

### 3.0 全景步骤总览（收藏这张）

```
第 0 步 准备环境，新建或复用 RuntimeLab 工程
   → 第 1 步 制造并修复循环引用
   → 第 2 步 用 Memory Graph 找环
   → 第 3 步 用 Instruments 测 CPU / 内存
   → 第 4 步 用 Organizer 看线上指标（可选）
   → 第 5 步 记录基线
```

### 3.1 第 0 步：准备环境

如果已经按姊妹篇建好 RuntimeLab，可以直接复用它；没有就新建一个 iOS App 工程（Storyboard + Swift）。

需要一台 Apple 芯片的 Mac（Xcode 27 仅支持 Apple 芯片；Intel Mac 请用 Xcode 26 及更早版本，菜单路径可能不同）、macOS 26.6 或更高版本、Xcode 27.2。Xcode 可以从 Mac App Store 安装，也可以到 `developer.apple.com/download` 下载。首次启动会要求安装附加组件和同意许可，磁盘至少留 40GB。模拟器运行时在 `Xcode → Settings → Components` 里安装（旧版本可能叫 `Platforms`）。下载 Xcode、模拟器运行时和依赖需要稳定网络；离线环境要提前准备好完整安装包。本文示例不依赖第三方库；如果你的工程用 CocoaPods 或 Swift Package Manager，先把依赖装好再开始。

验证安装：

```bash
xcodebuild -version      # 应显示 Xcode 27.2
swift --version          # 应显示 Swift 6.4.2
```

如果命令找不到，打开 `Xcode → Settings → Locations`，确认 `Command Line Tools` 选中了 Xcode 27.2。

没有付费开发者账号也能做本文的实验：用普通 Apple ID 在真机上跑免费签名即可。真机调试需要在 iPhone 上打开“开发者模式”（`设置 → 隐私与安全性 → 开发者模式`），并在 Xcode 的 `Window → Devices and Simulators` 里信任设备。真机系统要在 iOS 17 或更高版本（Xcode 27 的设备调试限制）；设备太旧时，可以先用模拟器跑通流程，但性能与内存数据要在符合要求的真机上测。签名细节见《iOS 签名与出包知识体系》。

内存与性能实验不需要 `.mm` 混编，用任意 UIKit 页面即可。

### 3.2 第 1 步：制造并修复循环引用

新建两个 Swift 类：

```swift
final class A {
    var b: B?
    deinit { print("A deinit") }
}

final class B {
    var a: A?
    deinit { print("B deinit") }
}
```

在 `viewDidLoad` 里创建 `a` 和 `b`，互相赋值，然后把外部变量置空。运行后你会发现两个 `deinit` 都不打印。把 `B` 的 `var a: A?` 改成 `weak var a: A?`，再运行，两个 `deinit` 都会打印。

这就是循环引用最小复现。真实项目里，把 `A` 和 `B` 换成 ViewController 和 delegate、或者 ViewController 和闭包，原理一样。

### 3.3 第 2 步：用 Memory Graph 找环

运行 App，进入有循环引用的页面，点击调试栏的 `Debug Memory Graph` 按钮。左侧搜索 `A` 或 `B`，选中对象，右侧看引用关系。你会看到 `A → B` 和 `B → A` 两条实线箭头，这就是环。

如果看不到对象，回到 `Edit Scheme → Run → Diagnostics`，勾选 `Malloc Stack`，重新运行。选中对象后，右侧还能看到分配时的调用栈，直接跳到创建它的代码。

### 3.4 第 3 步：用 Instruments 测 CPU 和内存

`Product → Profile`（`⌘I`），选 `Time Profiler`，点 `Record`，在 App 里反复进入/退出页面 10 秒，点 `Stop`。在 `Call Tree` 里勾选 `Separate by Thread`、`Invert Call Tree`、`Hide System Libraries`，看主线程最耗时的方法。

再选 `Allocations`，点 `Mark Generation`，做一次进入/退出，再点一次 `Mark Generation`。看两次标记之间哪些对象在增长。如果 `A`、`B` 一直在增长，说明环还在。

最后选 `Leaks`，看有没有红色条。Leaks 的红色条不一定都是你的问题，结合 Memory Graph 判断。

### 3.5 第 4 步：用 Organizer 看线上指标（可选）

`Window → Organizer → Metrics`，选择你的 App，可以看 `Launch Time`、`Hang Rate`、`Memory`、`Disk Writes`、`Energy` 等指标。这一步需要 App 已经通过 App Store 分发并有真实用户数据；本地实验没有数据是正常的，跳过即可。

### 3.6 第 5 步：记录基线，收尾

把改前/改后的 Instruments trace 保存下来，把 XCTest 的 `measure` 基线记下来，把 Memory Graph 导出（`File → Export Memory Graph`）留档。下次再改这块代码，先跑一遍基线，确认没有退化。

**SOP 常见失败**（卡住时先查这张）：

| 步骤 | 常见现象 | 先查什么 |
|---|---|---|
| 第 0 步 | 工程跑不起来 | 模拟器选择、签名、Bundle Identifier |
| 第 1 步 | `deinit` 不打印 | delegate / 闭包 / Timer 是否强引用；先看 1.4 节 |
| 第 2 步 | Memory Graph 里搜不到对象 | 先运行 App 再点按钮；打开 Malloc Stack 重新运行 |
| 第 3 步 | Instruments 没数据 | 用真机 + Release；确认选中了 App 进程 |
| 第 4 步 | Organizer 没数据 | 需要 App Store 分发，本地实验可跳过 |
| 第 5 步 | 改完又退化 | 用同样操作复测，保留改前/改后 trace |

---

## 四、常见报错与排查

> 这些报错分布在内存、线程和系统终止三个阶段。先按报错原文对号入座，再按顺序排查。

### 4.1 报错一：EXC_BAD_ACCESS

```
Thread 1: EXC_BAD_ACCESS (code=1, address=0x0)
```

或者：

```
-[Cat speak]: message sent to deallocated instance 0x600000c12345
```

这是访问了已经释放的内存，或者访问了无效地址。常见原因：对象被提前释放、`unowned` 引用指向的对象已经没了、野指针、越界。

**怎么办**：

1. 在 `Edit Scheme → Run → Diagnostics` 勾选 `Zombie Objects`（僵尸对象），复现崩溃。它会把已释放对象保留下来，控制台会告诉你“谁给谁发了消息”。
2. 用 `Debug Memory Graph` 看对象的引用关系，确认是不是被提前释放。
3. 检查 `weak`/`unowned` 的使用。`unowned` 在对象释放后访问会直接崩，改成 `weak` 试试。
4. 检查异步回调：网络/定时器回调回来时，页面可能已经销毁。用 `[weak self]` 并解包。
5. 如果只在 Release 崩，检查未初始化变量、越界、多线程竞争；必要时开 `Address Sanitizer`（内存检查工具）和 `Thread Sanitizer`（线程竞争检查工具）。

**预防**：代理用 `weak`；闭包用捕获列表；不要用 `unowned` 除非生命周期确定；异步回调先判断 `self` 是否还在。

### 4.2 报错二：页面退出后 deinit 不打印，内存持续上涨

这不是一条报错，而是一个症状。你反复进入/退出同一个页面，内存只涨不降，`deinit` 始终不打印。

**怎么办**：

1. 先加 `deinit { print(...) }`，确认对象确实没释放。
2. `Debug Memory Graph`，选中页面对象，看谁在引用它。
3. 按 1.4 节的五类现场逐一排查：delegate、闭包、Timer、父子对象、通知观察者。
4. 把可疑的强引用改成 `weak`，再跑一次。
5. 如果是闭包，检查 `[weak self]` 是否只写在最外层，内层闭包是否又强捕获了 `self`。

**预防**：代理默认 `weak`；闭包默认 `[weak self]`；Timer 在 `viewWillDisappear` 或 `deinit` 里 `invalidate()`；通知观察者在合适时机移除。

### 4.3 报错三：Leaks 有红条，但代码看起来没问题

Leaks 的红色条表示“检测到泄漏”，但不一定是你代码的错。系统框架、第三方库、一次性缓存、启动阶段的单例，都可能被标红。反过来，循环引用如果还挂在活着的对象上，Leaks 也可能不报。

**怎么办**：

1. 不要只看红条，点进去看泄漏对象的类型和分配栈。
2. 用 `Memory Graph` 确认引用环是否真的存在。
3. 用 `Allocations` 的 `Mark Generation`，重复同一操作 3~5 次，看对象数量是否持续增长。只增长一次可能是缓存，持续增长才像泄漏。
4. 对比改前/改后，不要追求“零红条”。

**预防**：把 Leaks 当作线索，不当作判决；以 Memory Graph 和重复操作的曲线为准。

### 4.4 报错四：UI API called on a background thread

```
Main Thread Checker: UI API called on a background thread: -[UIView setFrame:]
```

UIKit 不是线程安全的，UI 更新必须在主线程。这个报错通常来自网络回调、数据库回调、后台队列里直接改了 UI。

**怎么办**：

1. 把 UI 更新包进 `DispatchQueue.main.async { ... }`。
2. Swift 并发里给 UI 方法/类加 `@MainActor`。
3. 检查第三方 SDK 的回调线程，不要假设它一定在主线程。
4. 在 `Edit Scheme → Run → Diagnostics` 勾选 `Main Thread Checker`（主线程检查器），让它持续帮你抓。

**预防**：UI 相关代码统一走主线程；在团队规范里写明“回调不保证线程，UI 前先切主线程”。

### 4.5 报错五：App 被系统杀死（Jetsam）

```
EXC_CRASH (SIGKILL)
...
JetsamEvent
```

这不是普通的崩溃，而是系统在内存紧张时杀掉了进程。用户看到的是“突然闪退”，但崩溃日志里没有你的调用栈。

**怎么办**：

1. 用 `Allocations` 看峰值内存，确认是不是图片/视频/大数组导致。
2. 用 `Memory Graph` 找不必要的大对象和泄漏。
3. 图片降采样，列表只加载可见区域，缓存用 `NSCache`。
4. 在 `didReceiveMemoryWarning` 里释放可重建的缓存。
5. 后台任务及时结束，不要常驻大内存。

**预防**：把内存峰值纳入性能测试；大图/大文件操作前后打点；在真机上做低内存测试。

---

## 五、业务与面试视角

### 5.1 岗位 JD 为什么要求这些

内存和性能是 iOS 岗位的高频要求，常见表述包括：

- “熟悉 ARC、内存管理，能定位循环引用和内存泄漏”——对应第一章。
- “有性能优化经验，熟悉 Instruments 的 Time Profiler / Allocations / Leaks”——对应第二章。
- “能优化启动速度、卡顿、包体积、耗电”——对应 2.5~2.9。
- “能建立性能基线，做回归测试”——对应 2.10。

面试时不需要背所有 Instruments 按钮，但要能讲清楚：一次卡顿你怎么复现、怎么定位、怎么验证改好了。

### 5.2 面试口述钩子

**问：weak 和 unowned 的区别？**
答：都不增加引用计数。weak 在对象释放后自动变 nil，是可选类型；unowned 假设对象还活着，释放后访问会崩溃。默认用 weak。

**问：循环引用怎么排查？**
答：先加 deinit 打印，确认没释放；再用 Xcode 的 Debug Memory Graph 看引用箭头；最后用 Allocations 的世代对比确认对象是否持续增长。

**问：怎么定位一个卡顿？**
答：真机 + Release，先用 Animation Hitches 复现，再用 Time Profiler 看主线程热点，确认是 CPU、I/O 还是锁；改完用同样的操作复测。

**问：内存泄漏和内存上涨是一回事吗？**
答：不完全是。泄漏是不可达但没释放；上涨可能是缓存、临时对象峰值或泄漏。用 Memory Graph 看持有关系，用 Allocations 的世代对比看是否持续增长。

### 5.3 和其他文档怎么衔接

- 语言选型、Runtime、消息转发、Swift 派发、C/C++ 互操作：见《iOS 开发语言与运行时知识体系》。
- 签名、证书、描述文件、ipa、上架流程：见《iOS 签名与出包知识体系》。
- 游戏引擎、IL2CPP、dSYM：见《iOS 游戏工程与出包知识体系》。
- React Native 的 Bridge、JSI、TurboModule：见《iOS 与 RN 桥接知识体系》。

---

## 附录 A：实操入口地图

| 要做的事 | 位置（界面/命令） | 操作说明 |
|---|---|---|
| 启动性能分析 | `Product → Profile`（`⌘I`） | 选 Time Profiler / Allocations / Leaks |
| 打开 Instruments | `Xcode → Open Developer Tool → Instruments` | 也可以从 Profile 进入 |
| 看内存图 | 调试栏 `Debug Memory Graph` 按钮 | 需要 App 正在运行或暂停在断点 |
| 打开 Malloc Stack | `Edit Scheme → Run → Diagnostics` | 勾选 `Malloc Stack` |
| 导出内存图 | `File → Export Memory Graph` | 生成 `.memgraph` 文件，可分享 |
| 看线上指标 | `Window → Organizer → Metrics` | Launch Time、Hang Rate、Memory 等 |
| 看崩溃日志 | `Window → Devices and Simulators → 选设备 → View Device Logs` | 真机崩溃在这里看 |
| 清理构建 | `Product → Clean Build Folder`（`⇧⌘K`） | 改依赖后先清一次 |
| 看内存计量条 | Debug 导航器 → Memory Gauge | 黄色警告、红色危险 |
| 编辑 Scheme | `Product → Scheme → Edit Scheme`（`⌘<`） | Run/Profile 的配置都在这里 |

---

## 附录 B：排障“动手顺序”速查

```
① 先判断是崩溃、卡死还是内存上涨
   ├─ EXC_BAD_ACCESS           → 开 Zombie → Memory Graph → 查 weak/unowned
   ├─ UI API on background     → 切主线程 / @MainActor
   ├─ deinit 不打印            → Memory Graph 找环
   ├─ 内存持续上涨             → Allocations 世代对比
   └─ Jetsam                   → 看峰值内存 → 查大图/泄漏

② 再看性能
   ├─ 卡顿                     → Animation Hitches → Time Profiler
   ├─ 启动慢                   → App Launch → Signpost
   ├─ 发热                     → Energy Log → Time Profiler
   └─ 包大                     → App Thinning / Asset Catalog

③ 最后复测
   └─ 同样的操作再录一次，对比改前/改后，记录基线
```

**新手学习验收清单**（做完打勾）：

- [ ] 能解释 ARC、strong/weak/unowned 的区别。
- [ ] 能复现一个循环引用，并用 Memory Graph 找到它。
- [ ] 能用 Allocations 的世代对比判断内存是否持续增长。
- [ ] 能用 Time Profiler 找到主线程最耗时的方法。
- [ ] 能说清楚启动、卡顿、内存、耗电各自先看哪个工具。
- [ ] 能建立一次性能基线，并在改动后复测。

---

## 官方与参考链接

- [Xcode 27 Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)（Xcode 27 包含 Swift 6.4 与 iOS 27 SDK）
- [Xcode 27.2 发布记录](https://developer.apple.com/news/releases/?id=10292026a)（2026-10-29）
- [Announcing Swift 6.4.2](https://forums.swift.org/t/announcing-swift-6-4-2/90337)（Xcode 27.2 包含 Swift 6.4.2）
- [About iOS 27 Updates](https://support.apple.com/en-us/149076)（最新 iOS 27.2.1）
- [The Swift Programming Language: Automatic Reference Counting](https://docs.swift.org/latest/documentation/the-swift-programming-language/automaticreferencecounting/)
- [Gathering information about memory use](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use)
- [Apple 中文文档：正在收集内存使用信息](https://developer.apple.com/cn/documentation/xcode/gathering-information-about-memory-use)
- [Improving your app’s performance](https://developer.apple.com/documentation/xcode/improving-your-app-s-performance)
- [Improving app responsiveness（Animation Hitches）](https://developer.apple.com/documentation/xcode/improving-app-responsiveness)
- [Apple 中文文档：提升 App 响应性](https://developer.apple.com/cn/documentation/xcode/improving-app-responsiveness)
- [Reducing your app’s launch time](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)
- [Analyzing the performance of your shipping app（Organizer）](https://developer.apple.com/documentation/xcode/analyzing-the-performance-of-your-shipping-app)
- [Diagnosing memory, thread, and crash issues early（Main Thread Checker / Sanitizers）](https://developer.apple.com/documentation/xcode/diagnosing-memory-thread-and-crash-issues-early)
- [Profiling apps using Instruments](https://developer.apple.com/tutorials/instruments)
- [Apple 中文视频：Instruments 入门](https://developer.apple.com/cn/videos/play/wwdc2019/411/)
- [MetricKit](https://developer.apple.com/documentation/metrickit)
- [Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html)
- [Identifying high memory use with jetsam event reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)

> **可靠性说明**：版本号、SDK、菜单路径、Bitcode 政策等“当前事实”以 Apple 官方文档和发布记录为准，核验日期为 2026-11-21；Xcode 菜单在不同大版本间可能微调，找不到时用 Xcode 右上角搜索或 `Help → Search` 搜关键字。循环引用场景、性能优化优先级、面试表述属于通用实践，不同团队和项目会有差异。文中的代码是最小示例，按 Xcode 27.2 / Swift 6.4.2 语法编写，未在真机逐条运行；实际行为以编译器和运行时为准。语言、Runtime、消息转发、C/C++ 互操作的细节以姊妹篇《iOS 开发语言与运行时知识体系》为准；签名、上架、游戏引擎、React Native 的内容请以对应姊妹篇为准。
