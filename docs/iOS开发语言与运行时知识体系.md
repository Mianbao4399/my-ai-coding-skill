# iOS 开发语言与运行时知识体系（新手友好版）

> 覆盖：Objective-C / Swift / C++ 的选型与性能取舍、Objective-C++（.mm）混编、纯 C 与其他语言（JavaScript、Dart、C#、Rust 等）的接入边界、Objective-C Runtime（isa、objc_msgSend、方法缓存、消息转发三道防线、Method Swizzling、KVO、关联对象）、Swift 派发机制、Swift 6 并发、Swift 与 UI 框架（UIKit/SwiftUI）的协作机制、从零实操 SOP、报错排查与速查表。
> 适用版本：Xcode 27.2（2026-10-29 发布，随附 Swift 6.4.2）、iOS 27 SDK；Xcode 27 仅支持 Apple 芯片 Mac，需要 macOS 26.6 或更高版本，并支持 iOS 17 及以上设备调试。菜单路径以 Xcode 27 为准；旧版本入口可能不同，找不到时用 Xcode 搜索或 `Help → Search`。
> 文档边界：以“语言 → 运行时”为主线，讲开发者日常要用的原理与操作；内存管理、循环引用、Instruments、性能调优见姊妹篇《iOS 内存与性能调优知识体系》。签名、证书、ipa、上架流程见《iOS 签名与出包知识体系》；游戏引擎与 IL2CPP 见《iOS 游戏工程与出包知识体系》；React Native 桥接见《iOS 与 RN 桥接知识体系》。
> 阅读方式：每个概念先讲“是什么（带比喻）”，再讲“为什么需要它”，最后讲“怎么用/怎么排查”。关键词首次出现标注中英文。
> 信息核验日期：2026-11-21。版本、菜单路径等动态信息以官方页面为准（文末附链接）。

---

## 目录

- 0. 先用一组比喻建立全局
- 一、语言选型：Objective-C、Swift、C++ 与其他语言
- 二、Objective-C Runtime：消息是怎么送到方法上的
- 三、消息转发：方法找不到时的三道防线
- 四、Swift 的派发机制与互操作
- 五、从零到跑通：语言与运行时实操 SOP
- 六、常见报错与排查
- 七、进阶与业务视角
- 附录 A：实操入口地图
- 附录 B：排障“动手顺序”速查
- 官方与参考链接

---

## 0. 先用一组比喻建立全局

把一次 iOS 开发想象成开一家餐厅：

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| Objective-C | Objective-C | 老牌快递系统 | 靠运行时把“消息”送到对象手里 |
| Swift | Swift | 新一代智能厨具 | 编译期帮你查类型和安全，写起来更省心 |
| C++ | C++ | 后厨的重型设备 | 把计算性能压到硬件极限 |
| Objective-C++ | Objective-C++ | 双语翻译 | 让 C++ 设备接进 Objective-C 的餐厅 |
| C 语言 | C | 地基 | 系统 API 和上层语言的底座 |
| 其他语言 | Other Languages | 外援 | 通过桥接、引擎或 WebView 接入 |
| 运行时 | Runtime | 分拣中心 | 运行时决定一条消息最终由谁处理 |
| 消息发送 | Messaging | 寄快递 | Objective-C 每次调用方法的默认运输方式 |
| 方法缓存 | Method Cache | 常用电话快捷键 | 让重复的方法查找变快 |
| 消息转发 | Message Forwarding | 三层转接台 | 找不到方法时的补救通道 |

> ARC、循环引用、Instruments 的比喻在姊妹篇《iOS 内存与性能调优知识体系》第 0 节。

串起来一句话：**语言决定你怎么写，Runtime 决定消息怎么送到；对象什么时候死、哪里慢，见姊妹篇。**

---

## 一、语言选型：Objective-C、Swift、C++ 与其他语言

### 1.1 四种主力“语言身份”是什么

iOS 工程里最常用的是四种源文件。它们不是四选一，而是各管一段：

| 语言 / 模式 | 常见后缀 | 谁在编译 | 典型用途 | 一句话定位 |
|---|---|---|---|---|
| Objective-C | `.h` / `.m` | Clang | 老项目、UIKit、第三方 SDK、需要动态能力 | 消息在运行时才决定去哪 |
| Objective-C++ | `.h` / `.mm` | Clang（Objective-C++ 模式） | 桥接 C++ 库、游戏引擎、音视频核心 | 同一个文件里混写两种语法 |
| C++ | `.hpp` / `.cpp` | Clang | 游戏渲染、音视频编解码、跨平台核心 | 把性能压到硬件附近 |
| Swift | `.swift` | Swift 编译器 | 新项目、SwiftUI / UIKit 业务层 | 编译期把类型和安全查清楚 |

纯 C（`.c`）以及 JavaScript、Dart、C# 等语言也能进 iOS 工程，只是接入方式不同，统一放在 1.7 节。

新手最容易误解的一点是“Objective-C++ 是一门新语言”。它不是。它只是 Clang 的一种编译模式：当文件后缀是 `.mm` 时，编译器允许你在同一个文件里写 Objective-C 和 C++。

> **新手常见疑问：那我把 `.m` 改成 `.mm`，代码会变快吗？**
> 不会。`.mm` 只改变“这个文件能不能写 C++”，不会自动把 Objective-C 的消息派发变成 C++ 的静态调用。性能来自你写的代码结构，不来自后缀。

### 1.2 为什么 iOS 会同时存在这些语言

Objective-C 的历史可以追溯到 NeXT 时代，它继承了 Smalltalk 的消息模型：方法调用不是“跳到固定地址”，而是“把消息交给运行时，由运行时查表决定”。这套机制让 Objective-C 天生适合做 UI 框架、插件、代理和运行时增强。

#### 1.2.1 先把“消息发送”讲透

C++ 的普通方法调用，编译器在编译/链接期就能确定“调哪个函数”，生成的机器指令直接跳到那个地址；虚函数会多一次查表，但可选的实现集合在编译期已经确定。Objective-C 连“最终用哪个实现”都不写死。你写：

```objc
[cat speak];
```

编译器大致生成：

```objc
objc_msgSend(cat, @selector(speak));
```

调用点只告诉运行时两件事：接收者是谁（cat）、要它做什么（speak），再加上这次调用带的参数。至于 `speak` 最终由哪个函数实现，编译期不写死，运行时再查。查的过程是：顺着对象的 `isa` 找到类，查方法缓存，再查方法列表，找不到就沿父类往上找。完整流程在第二章。

| 项目 | C++ | Objective-C |
|---|---|---|
| 调用点知道什么 | 函数地址 | 接收者 + 方法名（`SEL`，方法名的运行时标识） |
| 什么时候绑定 | 编译/链接期 | 运行时 |
| 运行时能不能换实现 | 很难 | 可以：Swizzling、动态解析、转发 |
| 代价 | 调用快、可内联 | 多一次查表；缓存命中后接近普通调用 |

这张表的关键不是“谁快谁慢”，而是“谁在什么时候做决定”。Objective-C 把“做决定”推迟到了运行时，这就是动态绑定（Dynamic Binding，也叫晚绑定 Late Binding）。

#### 1.2.2 UI 框架为什么需要这种“晚决定”

UIKit 是苹果编译好、随系统发布的框架。它编译的时候，你的 `ViewController`、你的按钮回调方法还不存在，它不可能在编译期把地址写死。但它又必须在用户点击按钮时调用你的代码。target-action 就是答案：

```objc
[button addTarget:self
           action:@selector(buttonTapped:)
 forControlEvents:UIControlEventTouchUpInside];
```

UIKit 会把 target、action 和触发事件记下来：target 是谁，action 是哪个方法名，事件是点击还是别的。`UIControlEventTouchUpInside` 就是“手指在按钮范围内抬起”的点击事件。用户点击时，UIKit 再给 target 发 `buttonTapped:` 消息。你的类叫什么、方法地址在哪，UIKit 不需要提前知道。Storyboard/XIB（两种界面文件格式）里的连线（IBOutlet/IBAction）也是同一回事：界面文件里存的是类名和 selector 名，运行时才实例化类、绑定方法。

代理（Delegate）也是同一套思路。UITableView 的 delegate 协议里很多方法是可选的，不强制全部实现。调用前它先问一句：

```objc
if ([self.delegate respondsToSelector:@selector(tableView:didSelectRowAtIndexPath:)]) {
    [self.delegate tableView:self didSelectRowAtIndexPath:indexPath];
}
```

如果代理没实现，就跳过；实现了，就发消息。C++ 用纯虚函数（pure virtual function，只声明不实现的接口方法）做接口时，所有方法通常都得实现；Objective-C 的协议 + 可选方法（`@optional`）+ `respondsToSelector:` 更适合“框架定义回调，业务按需实现”的 UI 场景。

响应链（Responder Chain）同理：触摸事件从 UIView 往上传递，谁先能处理谁处理，框架不需要在编译期知道最终由哪个对象响应。KVO（Key-Value Observing，键值观察）存的是观察关系，通知和 `performSelector:` 存的是 selector 或 block，运行时再回调。

#### 1.2.3 插件和运行时增强为什么也靠它

插件的关键是“宿主编译时不知道插件是谁”。Objective-C 可以在运行时按类名找类、检查协议、发消息：

```objc
Class pluginClass = NSClassFromString(@"MyPlugin");
if ([pluginClass conformsToProtocol:@protocol(Plugin)]) {
    id<Plugin> plugin = [[pluginClass alloc] init];
    [plugin start];
}
```

宿主只依赖 `Plugin` 协议，不依赖插件的具体类；插件只要实现协议，编译成独立的库或模块，就能被宿主发现并调用。iOS 上不能随意下载可执行代码做插件（App Store 审核限制），所以这里的“插件”更多指工程内模块、framework（库包）、企业内部分发，或者 macOS 上的 Bundle（插件包）；能力本身仍然是运行时那套。

运行时增强更是直接建立在“方法地址可以变”上：Method Swizzling（方法交换）交换两个方法的 `IMP`（方法实现的函数指针），动态方法解析在运行时补方法，消息转发把消息交给别的对象，关联对象给 Category（类别，给已有类加方法）补“属性”，KVO 在运行时换一个动态子类。这些都不需要改调用点的代码，所以能做全局日志、埋点、兼容处理、AOP（Aspect-Oriented Programming，面向切面编程）。代价是调试更难、行为更隐式，第三章会讲边界。

#### 1.2.4 代价与边界

动态不是免费的。消息发送比静态调用多一次查表，编译器也不容易内联；动态替换让调用链更难跟踪；Swizzling 和转发用错会把问题放大到全局。所以现代 Swift 优先静态派发，只有需要 KVO、Swizzling、Objective-C 互操作时才用 `@objc dynamic` 接回消息派发（见第四章）。

四个场景的共同点可以压成一张表：

| 场景 | 调用方编译时知道什么 | 运行时靠什么接上 |
|---|---|---|
| UI 框架 | target + selector | `objc_msgSend` 查表 |
| 代理 | 协议声明 | `respondsToSelector:` + 消息 |
| 插件 | 协议 / 类名 | `NSClassFromString` + `conformsToProtocol:` |
| 运行时增强 | 原调用点 | Swizzling、动态解析、转发 |

把这段收成一句：Objective-C 不是“因为老才这样”，而是它选择用一点运行时开销，换来了“框架可以在不认识你的类的情况下调用你”的能力。UI 框架、代理、插件、运行时增强，都是这个选择的直接受益者。

#### 1.2.5 C/C++ 和 Swift 为什么也在

理解了 Objective-C 的选择，再看 C/C++ 和 Swift 的位置就清楚了。C 更贴近硬件，是系统 API 和跨平台核心的底座；C++ 在它之上提供零成本抽象、模板和 STL。游戏引擎、音视频编解码、图形渲染这些场景，C++ 仍然是主力。

Swift 在 2014 年发布，目标是“现代语法 + 编译期安全 + 接近 C 的性能”。它把 Optional、值类型、协议、泛型、并发检查做进了语言层，苹果这些年的新框架也优先支持 Swift。

所以一个真实的 iOS App 经常长这样：

```
Swift / Objective-C 业务层
        │
        ├── Objective-C++（.mm）或 C 桥接头
        │        │
        │        └── C / C++ 核心（算法 / 引擎 / 跨平台库）
        │
        └── 系统框架（UIKit / Foundation / SwiftUI）
```

### 1.3 性能对比要怎么看

“C++ 最快、Swift 第二、Objective-C 最慢”这个排序只在特定条件下成立。更实用的理解是按维度拆开：

| 维度 | C++ | Swift | Objective-C |
|---|---|---|---|
| 方法调用开销 | 直接调用，编译器可内联 | 以静态派发为主，需要动态时才走虚表/消息 | 默认走 `objc_msgSend` 消息派发 |
| 内存控制 | 手动或 RAII，最贴近硬件 | ARC 管理类实例，值类型复制 | ARC 管理对象 |
| 动态能力 | 弱，主要靠虚函数 | 中等，`@objc dynamic` 可接回运行时 | 强，运行时增删方法、交换实现、转发 |
| 类型安全 | 自己负责 | 强，Optional + 编译期检查 | 中等，动态类型多 |
| 开发效率 | 偏低，编译和调试成本高 | 高 | 中等 |
| 适合的位置 | 底层、跨平台、计算密集 | 业务、UI、新项目 | 遗留项目、动态增强、老 SDK |

有三件事需要说清楚，否则很容易被“语言鄙视链”带偏：

第一，跨语言调用有成本。C++ 再快，如果每个像素都要从 Swift 传一次数组、做一次 `std::string` 和 `NSString` 的互转，优势会被拷贝和桥接吃掉。C++ 应该处理“一整块计算”，而不是被切成一万次小调用。

第二，Swift 不是“全静态”。类的方法默认可以通过虚表派发，协议要求通过见证表（Witness Table）派发，标记了 `@objc dynamic` 的方法仍然走 `objc_msgSend`。只有结构体、枚举、`final` 方法、以及编译器能确定不会重写的方法，才稳定地走静态派发。

第三，Objective-C 的“慢”主要慢在没命中缓存的消息查找。第一次调用要查方法列表和继承链，之后命中方法缓存，开销就接近普通函数调用。真正需要优化时，先用 Instruments 量，不要凭感觉把 OC 代码重写成 C++。

### 1.4 选型建议

| 场景 | 建议 | 理由 |
|---|---|---|
| 全新 App | Swift 为主 | 类型安全、语法现代、苹果新框架优先支持 |
| 老 Objective-C 项目 | 保持 OC，增量混 Swift | 全量重写风险高，按模块迁移更稳 |
| 计算密集模块 | C++ 核心 + Swift/OC 外壳 | 性能收益明确，边界清晰 |
| 跨 iOS / Android 核心 | C++ | 一次编写，两端复用 |
| 需要 Method Swizzling / AOP | Objective-C 或 Swift + `@objc dynamic` | 这些能力依赖 Objective-C 运行时 |
| 集成第三方 SDK | 按 SDK 语言走 | OC SDK 用桥接头，C++ SDK 用 `.mm` 包一层 |
| 已有 C 库 / 底层接口 | 纯 C 或 C++ 包一层 | C ABI 稳定，跨语言调用方便 |
| Hybrid 页面 / 跨平台 UI | JavaScript、Dart、C# 等 | 通过 WebView、Flutter、Unity 等运行时接入，见 1.7 |
| 脚本化 / 热更新 | Lua、Python、JavaScript | 可嵌入，但下发脚本有审核风险，见 1.7 |
| 游戏 / 图形渲染 | 引擎语言为主，C++ 做底层 | 详见《iOS 游戏工程与出包知识体系》 |

一条实用原则：**业务层优先 Swift，底层和性能热点用 C/C++，动态增强留给 Objective-C，跨平台 UI 用对应框架语言；它们用 `.mm`、桥接头或 C ABI 连接。**

### 1.5 Objective-C++：把 C++ 装进 iOS 工程

Objective-C++ 的关键标识只有一个：文件后缀 `.mm`。Clang 看到 `.mm`，就会按 Objective-C++ 模式处理，允许你在里面同时写 `@interface`、`NSLog` 和 `std::vector`、`namespace`。

它有一个必须记住的限制：**Objective-C 的类体系和 C++ 的类体系彼此独立。** C++ 类不能继承 `NSObject`，Objective-C 类也不能继承 C++ 类。两者要合作，通常用“包装”而不是“继承”：写一个 Objective-C 类，把 C++ 对象作为成员变量或指针放进去，对外只暴露 Objective-C 的方法。

一个最小可运行的桥接例子，分四个文件：

```cpp
// FooEngine.h —— 纯 C++ 头文件，不要在这里 import Foundation
#pragma once
#include <string>

class FooEngine {
public:
    std::string greet(const std::string &name) const;
};
```

```cpp
// FooEngine.cpp —— 普通 C++ 实现
#include "FooEngine.h"

std::string FooEngine::greet(const std::string &name) const {
    return "Hello, " + name;
}
```

```objc
// FooEngineBridge.h —— 对外只暴露 Objective-C 接口
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN
@interface FooEngineBridge : NSObject
- (NSString *)greet:(NSString *)name;
@end
NS_ASSUME_NONNULL_END
```

```objc
// FooEngineBridge.mm —— 关键：后缀必须是 .mm
#import "FooEngineBridge.h"
#include "FooEngine.h"

@implementation FooEngineBridge {
    FooEngine _engine;   // C++ 对象可以作为 Objective-C 类的成员
}

- (NSString *)greet:(NSString *)name {
    std::string result = _engine.greet(name.UTF8String);
    NSString *output = [NSString stringWithUTF8String:result.c_str()];
    return output ?: @"";
}
@end
```

在 Xcode 里落地这几步：

1. `File → New → File`，选 `C++ File`，创建 `FooEngine.cpp`（删掉模板自带的 `main`，换成上面的实现）；再用 `Header File` 模板创建 `FooEngine.h`，把类声明放进去。
2. `File → New → File`，选 `Cocoa Touch Class`。这里的 Cocoa Touch 指 iOS 的框架层（UIKit、Foundation 等），这个模板就是 Xcode 的“新建 iOS 类”向导：你选父类和语言，它帮你生成 `.h`/`.m`（或 `.swift`）骨架。它不是一种语言，也不是某个具体的类。Class 填 `FooEngineBridge`，Subclass of 选 `NSObject`，Language 选 `Objective-C`。如果工程还没有桥接头，Xcode 会问要不要创建，选 `Create Bridging Header`。
3. 选中 `FooEngineBridge.m`，打开右侧 `File Inspector`（`⌥⌘1`），把 `Identity and Type → Name` 的后缀改成 `.mm`。
4. 到 `Build Settings → Swift Compiler - General → Objective-C Bridging Header` 确认路径指向 `YourApp-Bridging-Header.h`。
5. `Build`（`⌘B`）。C++ 文件必须勾选目标成员（Target Membership，决定文件是否参与当前 target 的编译），否则链接时会报 `Undefined symbol`。

两个高频坑：

- **在 `.m` 或 Swift 能看到的公共头文件里 `#include` C++ 头**。`.m` 不认识 C++ 语法，会报一堆 `Expected ';' after method prototype`。公共头只放 Objective-C 声明，C++ 头只在 `.mm` 里引用。
- **改了后缀但没改文件类型**。如果 Xcode 仍按 Objective-C 编译，检查 `File Inspector → Type` 是否显示 `Objective-C++ Source`。

### 1.6 Swift 6 并发只记三句话

Swift 6 语言模式把“数据竞争”从运行时问题提前到了编译期检查。新手先记住三句话就够用：

1. **跨线程共享可变状态是事故源。** 能不在线程间共享，就不共享。
2. **`actor` 是隔离箱。** 把可变状态放进 actor，外部只能通过 `await` 访问，编译器帮你排队。
3. **`Sendable` 是通行证。** 能安全跨线程传递的类型才标 `Sendable`；不确定就不要硬标。

Swift 6 模式可以按 target 逐步开启，不需要为了本文把整个老项目一次性切过去。第 4.4 节有一个最小 actor 例子。本文不展开 `async/await` 语法和 SwiftUI 并发，那属于另一条学习线。

### 1.7 纯 C 和其他语言：还能用什么

1.1 到 1.5 讲的是四种主力语言。实际工程里还会遇到纯 C、汇编、Metal 着色器、JavaScript、Dart、C#、Rust 等。它们进 iOS 工程的方式不一样，先记住三种接入类型：

| 接入类型 | 语言 / 技术 | 做法 |
|---|---|---|
| 直接编译 | C、C++、Objective-C、Objective-C++、Swift、汇编、Metal | 源文件加进 target，Xcode 直接编译 |
| 编译成库再接入 | Rust、Go、Kotlin/Native | 先产出静态库（static library）或 framework（库包），再通过 C ABI / 桥接头调用 |
| 运行时 / 引擎接入 | JavaScript、Dart、C#、Lua、Python | 嵌入 JS 引擎、Flutter/Unity 引擎或语言解释器 |

C ABI（Application Binary Interface，应用二进制接口）可以理解成“不同语言编译出来的函数，按什么规则传参数、返回值、对齐内存”。只要都暴露 C 兼容接口，C、C++、Swift、Rust 编译出来的库就能互相调用。

#### 1.7.1 C 语言：iOS 的地基

C 不仅能用，而且是 iOS 的底层基础。Objective-C 本身就是 C 的超集：`.m` 文件里可以直接写 C 代码；`.c` 文件也能和 Objective-C、Swift、C++ 一起链接进同一个 App。

Darwin（iOS 的底层系统）、POSIX（可移植操作系统接口）、GCD（Grand Central Dispatch，苹果的并发队列）、Core Foundation、SQLite 这些接口本身就是 C API。你平时写的 Swift/Objective-C 代码，底层经常在调用它们。

| C API / 库 | 用途 |
|---|---|
| Darwin / POSIX | 文件、线程、socket、内存、锁 |
| GCD（`dispatch_*`） | 队列、并发、定时器 |
| Core Foundation | 底层字符串、集合、RunLoop |
| SQLite / zlib / libcurl | 数据库、压缩、网络 |

C 的接入很直接：把 `.c` / `.h` 加进 target，Objective-C 直接 `#import`，Swift 通过桥接头（Bridging Header）或 module map（把 C/ObjC 头声明成 Swift 可 import 的模块）看到函数声明。1.7.4 会走一遍最小实操。

C 没有 ARC，也没有类和对象。`malloc/free`、`memcpy`、指针生命周期都要自己管；如果要把 Objective-C 对象传给 C 函数，通常用 `void *` 承接，再用 `__bridge`（在 ObjC 对象指针和 C 指针之间转换的桥接标记）转换。跨语言边界传对象时，先想清楚“谁持有、谁释放”。

性能上，C 和 C++ 同属“直接编译、手动管理内存”的一侧：C 更贴近硬件，C++ 多了零成本抽象和标准库。

#### 1.7.2 其他语言一览

| 语言 / 技术 | 接入方式 | 典型用途 | 边界与注意 |
|---|---|---|---|
| C++ | `.cpp` / `.hpp`，或 `.mm` 包装 | 引擎、音视频、算法 | 类体系与 Objective-C 独立，见 1.5 |
| 汇编（Assembly） | `.s` / `.S` 文件 | arm64 底层优化、运行时 | 可读性差，非必要不手写 |
| Metal Shading Language（Metal 的 GPU 着色器语言） | `.metal` 文件 | GPU 渲染、并行计算 | 只跑在 GPU，不是通用业务语言 |
| JavaScript | JavaScriptCore（苹果自带的 JS 引擎，`JSContext` 是执行上下文）或 WKWebView（系统 WebView 组件） | Hybrid 页面、动态脚本 | 下发脚本改功能有审核风险 |
| Dart | Flutter 引擎 | 跨平台 UI | Dart 代码在构建时 AOT（Ahead-of-Time，提前编译）成原生代码，由 Flutter 引擎运行，不能被 Swift 直接调用 |
| C# | Unity + IL2CPP（Unity 把 C# 转成 C++ 的工具链） | 游戏逻辑 | 依赖 Unity 工具链，不是通用 iOS 语言 |
| Kotlin/Native | 产出 framework | Kotlin Multiplatform（KMP，跨平台共享代码方案） | 工具链和版本管理成本较高 |
| Rust | 静态库 / framework + C ABI | 高性能核心、复用库 | Xcode 没有一等支持，需要自己写头文件或 module map |
| Go | gomobile（Go 官方的移动端绑定工具）产出 framework | 共享逻辑、工具 | 包体和工具链成本较高，不算主流 |
| Lua / Python | 嵌入解释器 | 游戏脚本、工具、规则配置 | 审核和包体风险高，先确认是否真的需要 |

这里保留 C++ 作为对照，其余是四种主力语言之外的选项。C++ 的细节在 1.5，Objective-C 和 Swift 的派发/互操作在第二章、第四章。

#### 1.7.3 接入方式怎么选

- 直接编译：C/C++/ObjC/ObjC++/Swift/汇编/Metal。新建文件时选对应模板，确认 Target Membership；C/C++ 头文件按 1.7.4 的方式暴露给 Swift。
- 编译成库：C/C++ 静态库、Rust/Go/Kotlin/Native。先在各自工具链里产出真机 arm64 和模拟器 arm64 的库，再把头文件和库文件拖进 Xcode。调用边界尽量用 C ABI，避免把 C++ 模板暴露给 Swift。
- 运行时/引擎：JavaScript/Dart/C#/Lua/Python。JavaScript 用 `JSContext` 或 `WKWebView`；Flutter/Unity 把引擎作为 framework 集成；Lua/Python 嵌入解释器。它们不是“Xcode 源文件”，而是随 App 一起打包的运行时。

#### 1.7.4 最小实操：让 Swift 调用 C 函数

1. `File → New → File`，选 `Header File`，命名 `hello.h`，声明 `const char *runtime_lab_hello(void);`。
2. `File → New → File`，选 `C File`，命名 `hello.c`，删掉模板自带的 `main`，实现这个函数。
3. 打开桥接头（没有就新建一个 `YourApp-Bridging-Header.h`，本文示例叫 `RuntimeLab-Bridging-Header.h`；然后在 `Build Settings → Swift Compiler - General → Objective-C Bridging Header` 填它的路径），加入 `#import "hello.h"`。
4. 在任意 Swift 文件（本文示例是 `ViewController.swift` 的 `viewDidLoad`）里调用 `print(String(cString: runtime_lab_hello()))`，运行。

```c
// hello.h
#ifndef hello_h
#define hello_h

const char *runtime_lab_hello(void);

#endif
```

```c
// hello.c
#include "hello.h"

const char *runtime_lab_hello(void) {
    return "hello from C";
}
```

```objc
// RuntimeLab-Bridging-Header.h
#import "hello.h"
```

```swift
// ViewController.swift
print(String(cString: runtime_lab_hello()))
```

控制台输出 `hello from C`。`runtime_lab_hello()` 返回的是 C 字符串指针，Swift 用 `String(cString:)` 复制成 Swift 字符串；如果函数可能返回 `NULL`，要先判断再转换。

#### 1.7.5 边界与常见坑

- 审核：App Store Review Guidelines 2.5.2 不允许下载可执行代码来增加或改变 App 功能。嵌入解释器/脚本引擎通常可以，但“线上下发脚本改核心逻辑”有审核风险，能正常发版就正常发版。
- 内存：C 没有 ARC；`malloc/free` 要成对，C++ 用 RAII（Resource Acquisition Is Initialization，用对象生命周期管理资源）或智能指针；跨 ObjC 边界传对象注意 `__bridge` 和生命周期。
- 头文件：公共 Objective-C 头里不要 `#include` C++ 头；C++ 只在 `.mm` 里出现。C++ 调 C 函数时，C 头文件通常用 `#ifdef __cplusplus` + `extern "C"` 包住声明，避免 C++ 名称改编导致 `Undefined symbol`。Swift 要调 C++，优先用 `.mm` 包一层，或者使用 Swift 的 C++ 互操作。
- 调用开销：跨语言调用有固定成本，别把“一个数组的每个元素”都过一次桥；一次传整块数据，在 C/C++ 里循环。
- 架构切片：真机 arm64，Apple 芯片模拟器 arm64；老版本 Xcode 在 Intel Mac 上的模拟器是 x86_64。静态库缺切片会报 `Undefined symbol`，动态库没嵌入会报 `dyld: Library not loaded`（见 6.3）。
- 静态库裁剪：如果插件类（见 1.2.3）放在静态库里且没有被直接引用，链接器可能把它裁掉，`NSClassFromString` 会返回 `nil`。检查 `Other Linker Flags`（链接器额外参数；常见做法是加 `-ObjC`），并以工程实际配置为准。
- 包体与安全：多引入一个语言运行时，就多一份包体和攻击面；第三方库要确认许可证、维护状态和符号冲突风险。

选型顺序可以记成：业务 Swift，老项目 Objective-C，底层 C/C++，动态能力 Objective-C，跨平台 UI 用对应框架语言，脚本只做可控的辅助功能。

---

## 二、Objective-C Runtime：消息是怎么送到方法上的

### 2.1 是什么：给对象“寄快递”

Objective-C 的运行时（Runtime）是一套 C 函数和数据结构，负责对象的创建、方法查找、消息发送、动态加载。你写的这一行：

```objc
[cat speak];
```

编译器不会把它编译成“直接跳到 `speak` 函数地址”，而是编译成一次消息发送：

```objc
objc_msgSend(cat, @selector(speak));
```

用比喻说：C++ 调方法像打固定电话，号码在编译期就写死了；Objective-C 调方法像寄快递，收件人写的是“cat”，包裹上写的是“speak”，至于最后由谁签收、怎么处理，要到分拣中心（Runtime）才知道。

这就是为什么 Objective-C 能在运行时做很多静态语言做不到的事：给类动态加方法、交换两个方法的实现、把消息转给另一个对象、甚至让一个对象“假装”自己拥有它没有的方法。

### 2.2 对象和类的内部结构

每个 Objective-C 对象的内存布局，第一个成员是一个指向类对象的指针，叫 `isa`。类对象里存着父类指针、方法列表和方法缓存。简化后的结构是这样：

```
cat 对象（堆上）
┌──────────────────┐
│ isa ─────────────┼──► Cat 类对象
│ 成员变量 ...      │
└──────────────────┘

Cat 类对象
┌──────────────────┐
│ isa ─────────────┼──► Cat 的元类（存类方法）
│ superclass ──────┼──► Animal 类对象 ──► NSObject
│ method list      │     speak  → 0x1000A
│                  │     eat    → 0x1000B
│ cache            │     speak  → 0x1000A（最近用过）
└──────────────────┘
```

几个名词先对齐：

- `isa`：对象的“所属类”指针。对象靠它找到自己的类，再找到方法。
- `SEL`（Selector，选择器）：方法名的编号。你可以把它理解成“方法名在运行时的身份证号”。
- `IMP`（Implementation，实现）：真正执行的那个 C 函数指针。
- 方法列表（Method List）：`SEL → IMP` 的映射表。
- 方法缓存（Method Cache）：最近调用过的方法，查过一次就记下来，下次直接命中。

### 2.3 一次消息发送的完整查找顺序

`objc_msgSend` 收到消息后，按下面的顺序找：

1. **查缓存。** 先看接收者所属类的方法缓存里有没有这个 `SEL`。有就直接跳过去，这是最快路径。
2. **查当前类的方法列表。** 缓存没有，就在 `Cat` 类自己的方法列表里找。
3. **沿继承链向上。** 当前类没有，就通过 `superclass` 指针去父类找，一层层直到 `NSObject`。
4. **找到就调用。** 把接收者对象和参数一起传给 `IMP`，同时传入两个隐藏参数：`self`（接收者）和 `_cmd`（当前方法的选择器）。
5. **找不到就进入补救流程。** 也就是下一章的三道防线，全部失败才抛 `unrecognized selector`。

缓存为什么重要？方法缓存是每个类一份的：命中缓存后，消息发送只比普通函数调用慢一点点。程序跑热以后，绝大多数消息都能命中缓存。所以“OC 方法调用特别慢”这个说法，只对“冷启动、大量首次调用、频繁动态解析”这些场景成立。

### 2.4 怎么亲眼看到这些结构

最直接的动手方式，是拿到方法的 `IMP` 然后当函数指针调用。新建一个 Objective-C 类 `Cat`，实现 `-speak`，然后写：

```objc
#import <objc/runtime.h>

Method method = class_getInstanceMethod([Cat class], @selector(speak));
IMP imp = method_getImplementation(method);
((void (*)(id, SEL))imp)(cat, @selector(speak));
```

这段代码做了三件事：从类里取出方法、拿到函数指针、按“接收者 + 选择器”的函数签名直接调用。它绕过了 `objc_msgSend` 的查找，适合观察，不适合日常业务代码。

在 LLDB（Xcode 自带的调试器）里也可以直接看：

```
(lldb) po cat
(lldb) image lookup -n "-[Cat speak]"
```

第二条命令会告诉你 `speak` 的实现落在哪个地址、哪个源文件。想看汇编级的消息发送，可以在 `[cat speak]` 这一行下断点，然后打开 `Debug → Debug Workflow → Always Show Disassembly`，单步进去就能看到 `objc_msgSend`。

### 2.5 几个容易混淆的点

**Category 为什么不能加实例变量？** 类在运行时注册后，对象的内存布局就固定了，给已有类加实例变量会破坏所有已创建对象的内存布局。Category 可以加方法，不能加 ivar；想“加属性”，通常用关联对象（见 3.8）。

**`respondsToSelector:` 为什么对转发的方法返回 `NO`？** 因为它只查真实的继承体系，不查转发链。一个对象即使能把消息转发出去，`respondsToSelector:` 仍然可能说“不会”。需要对外表现一致时，要自己重写这个方法（见 3.5）。

**Swift 的方法也走 `objc_msgSend` 吗？** 默认不走。Swift 有自己的派发方式，只有标记了 `@objc dynamic`、或者需要暴露给 Objective-C 的方法，才会接回消息派发（见第四章）。

### 2.6 Runtime 常用 API 速查

Runtime 的函数很多，先记住下面这些最常用的；具体用法在前面的小节里。

| API | 作用 | 新手注意 |
|---|---|---|
| `objc_msgSend(receiver, selector)` | 发送消息 | 编译器生成，不要手写调用 |
| `class_getInstanceMethod` | 取实例方法 | 找不到返回 `NULL` |
| `method_getImplementation` | 取方法实现 `IMP` | 函数签名要和类型编码一致 |
| `class_addMethod` | 动态加方法 | 类型编码写错会参数错乱 |
| `method_exchangeImplementations` | 交换两个方法实现 | 用 `dispatch_once`，注意继承 |
| `class_copyPropertyList` | 取属性列表 | 用完要 `free` |
| `object_getClass` | 取对象的类 | 和 `[obj class]` 在 KVO 场景可能不同 |
| `objc_setAssociatedObject` | 设置关联对象 | 生命周期跟随宿主 |
| `objc_getAssociatedObject` | 读取关联对象 | 注意内存策略 |
| `NSInvocation` | 封装一次消息调用 | 完整转发的核心对象 |

---

## 三、消息转发：方法找不到时的三道防线

### 3.1 是什么：找不到人时的三层转接

如果整条继承链都没有实现某个方法，静态语言通常会直接编译失败或运行时崩溃。Objective-C 不会立刻崩，它在报错前给接收者三次补救机会：

| 顺序 | 防线 | 入口方法 | 开销 | 能做什么 |
|---|---|---|---|---|
| 第一道 | 动态方法解析（Dynamic Method Resolution） | `resolveInstanceMethod:` / `resolveClassMethod:` | 最低 | 运行时给类加一个方法 |
| 第二道 | 快速转发（Fast Forwarding） | `forwardingTargetForSelector:` | 低 | 把消息原样转给另一个对象 |
| 第三道 | 完整转发（Normal Forwarding） | `methodSignatureForSelector:` + `forwardInvocation:` | 最高 | 拿到 `NSInvocation`，改参数、改返回值、转发给多个对象 |

这三道防线只会在“方法没找到”时触发。正常调用根本不会进这里，所以它们不会拖慢日常代码。

### 3.2 第一道：动态方法解析

运行时先问类：“你能不能在运行的时候补一个方法出来？”对应的入口是 `resolveInstanceMethod:`（实例方法）和 `resolveClassMethod:`（类方法）。你返回 `YES` 并调用 `class_addMethod`，运行时就会把方法加进去，然后重新走一遍查找。

```objc
#import <objc/runtime.h>

static void DynamicHello(id self, SEL _cmd) {
    NSLog(@"hello from a dynamically added method");
}

@implementation Cat
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(dynamicHello)) {
        class_addMethod([self class], sel, (IMP)DynamicHello, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
@end
```

`class_addMethod` 的第四个参数是类型编码（Type Encoding）：`"v@:"` 表示“返回 void，参数是 id 和 SEL”。类型编码写错，运行时会按错误的签名调用函数，轻则参数错乱，重则崩溃。

这一道适合 `@dynamic` 属性、Core Data 这类“方法在运行时才生成”的场景。它的代价是方法会被真正加进类里，之后走正常缓存，不会每次调用都重新解析。

### 3.3 第二道：快速转发

如果动态解析没接住，运行时会问：“要不要把这条消息先转给别人？”实现 `forwardingTargetForSelector:`，返回另一个对象即可：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.helper respondsToSelector:aSelector]) {
        return self.helper;   // 把消息原样交给 helper
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

返回的对象会成为新的接收者，运行时从它身上重新开始查找。因为它不需要创建 `NSInvocation`，开销比第三道低很多，适合“我内部组合了另一个对象，大部分方法直接借给它”的场景。

两个边界：返回 `self` 会造成无限循环；返回 `nil` 或 `super` 的结果，就会继续走第三道。

### 3.4 第三道：完整转发

最后一道最灵活，也最贵。运行时先问“这个方法长什么样”，再给你一个封装好的 `NSInvocation`：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (!signature && [self.helper respondsToSelector:aSelector]) {
        signature = [self.helper methodSignatureForSelector:aSelector];
    }
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if ([self.helper respondsToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:self.helper];
    } else {
        [super forwardInvocation:anInvocation];
    }
}
```

`NSInvocation` 里装着接收者、选择器、所有参数和返回值。你可以在 `invokeWithTarget:` 之前改参数，也可以在调用之后读返回值。全局日志、权限校验、无埋点、Mock 框架，很多都是在这个位置做文章。

`methodSignatureForSelector:` 必须返回一个非空签名。返回 `nil`，运行时会直接走 `doesNotRecognizeSelector:`，也就是你熟悉的崩溃：

```
-[Cat speak]: unrecognized selector sent to instance 0x600000c12345
```

### 3.5 转发与继承的边界

转发能让一个对象“看起来”会某个方法，但它并不是真的继承。`respondsToSelector:` 和 `isKindOfClass:` 只看真实继承体系，不看转发链。如果你希望转发对调用方完全透明，需要自己重写这些方法，让它们把转发目标也算进去。

同样，`methodSignatureForSelector:` 要能准确描述被转发的方法。签名不对，`NSInvocation` 打包参数时就会出错。

有一句话值得记住：**转发是高级技巧，不是继承的替代品。** 能用组合、协议、子类解决的，优先用那些；只有在确实需要动态代理、AOP、兼容未知消息时，才动用转发。

### 3.6 这套机制能做什么

| 场景 | 常用防线 | 说明 |
|---|---|---|
| 模拟多继承 | 快速转发 / 完整转发 | 把不同能力分给内部几个对象，对外表现成一个对象 |
| 动态代理 | 完整转发 + `NSProxy` | `NSProxy` 和 `NSObject` 一样是根类，专门用来做代理 |
| AOP（Aspect-Oriented Programming，面向切面编程）/ 全局埋点 | 完整转发 | 在 `forwardInvocation:` 里记录参数、耗时、返回值 |
| 兼容旧 API | 快速转发 | 把废弃方法转给新实现，避免调用方立刻改 |
| 插件 / 未知消息 | 动态解析 | 运行时按配置注入方法 |
| 字典转模型 | 不靠转发 | 用 `class_copyPropertyList` 等运行时 API 读取属性，再赋值 |

热修复类框架也常把动态解析和 Method Swizzling 组合起来用。上线产品要特别注意审核合规：向 App 下发可执行代码、动态改变核心逻辑，存在被拒风险。能走正常发版就正常发版。

### 3.7 Method Swizzling：运行时换实现

Method Swizzling（方法交换）是把两个方法的 `IMP` 互换。交换后，调用 A 实际执行 B 的代码，调用 B 实际执行 A 的代码。它常用于给系统方法加日志、统计、兼容处理。

```objc
#import <UIKit/UIKit.h>
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Method original = class_getInstanceMethod(self, @selector(viewDidAppear:));
        Method replacement = class_getInstanceMethod(self, @selector(app_viewDidAppear:));
        method_exchangeImplementations(original, replacement);
    });
}

- (void)app_viewDidAppear:(BOOL)animated {
    [self app_viewDidAppear:animated];   // 交换后，这里实际调用原实现
    NSLog(@"viewDidAppear 被拦截");
}

@end
```

这段代码能跑，但不要把它当成日常工具。Swizzling 的风险很实际：

- **顺序敏感。** 多个库都交换同一个方法时，谁先谁后会影响结果。
- **继承陷阱。** 如果子类没有实现原方法，`class_getInstanceMethod` 可能拿到父类的实现，交换后影响面比你想的大。
- **调试困难。** 断点停在你写的函数上，调用栈却是系统方法，新手很容易看懵。
- **审核与稳定性。** 交换系统方法属于高风险操作，出问题往往是全局的。

如果必须做，至少守住几条：用 `dispatch_once` 保证只执行一次；方法名加前缀避免冲突；把交换集中在一处（示例放在 `+load`，也可以放到 App 启动早期），不要散落在多个文件；写清楚为什么交换、影响范围是什么；有自动化测试兜底。

更稳的替代方案是子类、组合、通知、协议扩展。Swizzling 应该是“没有别的办法”时的选择，不是第一选择。

### 3.8 KVO 与关联对象

KVO（Key-Value Observing，键值观察）允许一个对象监听另一个对象某个属性的变化。它的默认实现依赖运行时动态子类：系统在运行时给被观察对象换一个子类，重写属性的 setter，从而在赋值前后插入通知。

```objc
[self.account addObserver:self
               forKeyPath:@"balance"
                  options:NSKeyValueObservingOptionNew
                  context:nil];
```

传统 KVO 必须成对移除观察者，否则对象销毁后再收到通知就可能崩溃。Swift 的 `observe(_:options:changeHandler:)` 返回一个 token，token 生命周期结束会自动失效，比手动移除安全一些。

关联对象（Associated Object）是给 Category “补属性”的常用手段：

```objc
objc_setAssociatedObject(self, @selector(displayName), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
NSString *name = objc_getAssociatedObject(self, @selector(displayName));
```

它把值挂到对象上，生命周期跟随宿主对象。适合少量、明确的附加数据；不要拿它当“万能属性袋”，否则内存和可读性都会变差。

---

## 四、Swift 的派发机制与互操作

### 4.1 四种派发方式

Swift 的方法调用不像 Objective-C 那样“一律走消息”，它会根据类型和修饰符选择不同的派发方式：

| 派发方式 | 触发条件 | 性能 | 例子 |
|---|---|---|---|
| 静态派发（Static Dispatch） | 结构体、枚举、`final` 方法、编译器能确定不会重写 | 最快，可内联 | `struct Point { func norm() }` |
| 虚表派发（VTable Dispatch） | 非 `final` 的类方法 | 快，一次查表 | `class Animal { func speak() }` |
| 见证表派发（Witness Table Dispatch） | 通过协议要求调用 | 中等，一次查表 | `protocol Greeter { func greet() }` |
| 消息派发（Message Dispatch） | `@objc dynamic`；或从 Objective-C 侧调用暴露的方法 | 最灵活，开销最高 | `@objc dynamic func refresh()` |

一句话：**Swift 不是“全静态”，而是“能静态就静态，需要动态才动态”。**

### 4.2 哪些写法会把 Swift 拉回动态派发

- 给方法加 `@objc dynamic`：明确要求走 Objective-C 消息派发，常用于 KVO、Swizzling 或 Objective-C 调用。
- 类继承 `NSObject` 并把方法暴露给 Objective-C：会生成桥接 thunk；从 Objective-C 侧调用时走消息派发，Swift 内部调用仍可能静态派发。
- 使用 `performSelector` 这类运行时 API：调用本身绕过了编译期检查。
- 使用 `@objcMembers`：把整个类的成员都暴露出去，桥接开销和暴露面都会变大。

这些修饰不是“免费”的。每暴露一个方法，编译器就要生成一层 Objective-C 的 thunk（桥接跳板）；从 Objective-C 侧调用时会走消息派发。Swift 内部调用能不能保持静态派发，要看具体修饰和优化情况。合理的做法是只暴露确实需要的那几个，不要为了“看起来兼容 OC”给整个类加 `@objcMembers`。

### 4.3 Swift 与 Objective-C 互操作

Swift 调 Objective-C 很直接：在桥接头里 `#import` Objective-C 头文件，Swift 就能看到这些类和方法。

Objective-C 调 Swift 需要显式暴露：

- 类要继承 `NSObject`。
- 方法/属性加 `@objc`，或者整个类加 `@objcMembers`。
- Xcode 会生成 `ProductModuleName-Swift.h`，Objective-C 文件 `#import` 它就能调用。
- 想改 Objective-C 侧看到的名字，用 `@objc(MyClassName)`。

一个最小例子：

```swift
@objcMembers
final class UserCenter: NSObject {
    func refresh() {
        // 业务逻辑
    }
}
```

Objective-C 侧：

```objc
#import "YourApp-Swift.h"

UserCenter *center = [[UserCenter alloc] init];
[center refresh];
```

桥接头（Bridging Header）的方向相反：它是给 Swift 看 Objective-C 的。两个方向别搞混。

### 4.4 Swift 6 并发的最小例子

```swift
actor Counter {
    private var value = 0

    func increment() -> Int {
        value += 1
        return value
    }
}
```

`actor` 把 `value` 保护起来，外部只能通过 `await counter.increment()` 访问，编译器会检查跨线程访问是否安全。配合 `Sendable`，Swift 6 能在编译期拦下大部分数据竞争。

新手先不用背所有规则，记住“共享可变状态要隔离，跨线程传递要 `Sendable`”就够了。真正迁移时，按 target 开启 Swift 6 模式，跟着编译器报错逐个修，比一次性重写靠谱。

### 4.5 Swift 与 UI 框架的协作机制

UIKit 和 SwiftUI 解决的是同一个问题：框架要调用“编译时还不认识”的代码。它们用的机制不同，写 Swift 时要注意的规则也不同。

#### 4.5.1 UIKit + Swift：还是走 Objective-C Runtime

UIKit 本身是 Objective-C 框架。Swift 类继承 `UIViewController`、实现按钮回调、当 delegate 时，只有暴露给 Objective-C 的方法或属性，UIKit 才能通过 selector 找到。

```swift
final class ViewController: UIViewController {
    @IBAction func buttonTapped(_ sender: UIButton) {
        print("tapped")
    }
}
```

`@IBAction` 和 `@IBOutlet` 都是 Interface Builder 的连接标记，本质上是 `@objc`：它们把方法或属性暴露给 Objective-C 运行时。代理方法只要满足 `@objc` 协议的要求，也会自动暴露；KVO 则要求属性标 `@objc dynamic`。

| 场景 | Swift 写法 | 底层 |
|---|---|---|
| 按钮点击 | `@IBAction` / `@objc func` | target-action 发 selector |
| delegate | 实现 `@objc` 协议的方法 | `respondsToSelector:` + 消息发送 |
| KVO | `@objc dynamic var` | 运行时动态子类 + setter 拦截 |
| 通知 selector | `@objc func` | `NotificationCenter` 发消息 |

所以 UIKit 里有一条很实际的规则：**没标 `@objc` 的 Swift 方法，target-action、KVO、`performSelector` 都看不见它**。这也是为什么 Swift 明明有静态派发，接 UIKit 时还是要把一部分方法“交回”给 Runtime。`@objc` 会生成桥接 thunk，只暴露确实需要的方法即可。

#### 4.5.2 SwiftUI：靠泛型和协议，不靠 Runtime 查表

SwiftUI 是 Swift 原生框架，正常视图更新不需要 `objc_msgSend`。它利用的是 `View` 协议、泛型和属性包装器：

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        Button("+1") { count += 1 }
    }
}
```

可以这样理解：

- `View` 协议要求一个 `body` 属性，SwiftUI 通过泛型 + 协议见证表（Witness Table）调用它，不需要知道你的具体类名。
- `some View` 是 opaque result type（不透明结果类型）：编译器知道真实类型，调用方只看到 `View` 协议；SwiftUI 能在编译期/链接期做特化。
- `@State`、`@Binding` 是属性包装器；`@Observable` 是 iOS 17 起的观察宏。值变了，SwiftUI 重新计算受影响的 `body`，再比较视图值树、更新渲染。
- 因此 `body` 应该保持轻量：它会被频繁调用，不适合在里面做网络请求或大计算。

SwiftUI 不是“完全不用动态机制”：协议类型的动态调用、观察机制仍然有运行时工作。但正常视图更新不依赖每个视图的 `objc_msgSend`，这是它和 UIKit 最大的区别。

#### 4.5.3 一张表对比

| 场景 | 框架怎么找到你的代码 | 是否走 `objc_msgSend` |
|---|---|---|
| UIKit target-action | selector 查表 | 是 |
| UIKit delegate | Objective-C 协议 + 消息发送 | 是 |
| UIKit KVO | 运行时动态子类 | 是 |
| SwiftUI `body` | `View` 协议 + 泛型/见证表 | 通常否 |
| SwiftUI 状态更新 | 属性包装器 + 观察机制 | 通常否 |
| 纯 Swift delegate | Swift 协议见证表 | 否 |

两边互相接的时候：

- SwiftUI 里用 UIKit 控件：`UIViewRepresentable` / `UIViewControllerRepresentable`（把 UIKit 控件包进 SwiftUI 的协议）。
- UIKit 里嵌 SwiftUI：`UIHostingController`（承载 SwiftUI 视图的 UIKit 控制器）。
- SwiftUI 里观察旧式 `ObservableObject`（Combine 时代的可观察对象协议）：`@StateObject` / `@ObservedObject`。
- iOS 17 起可以用 `@Observable` 宏，走 Observation 框架，不再依赖 `@Published`（`ObservableObject` 的属性发布标记）。

一句话收束：**UIKit 靠 `@objc` 把 Swift 代码重新暴露给 Runtime；SwiftUI 靠泛型和协议，在编译期就知道你的视图类型，所以不需要每次更新都去 Runtime 查表。**

---

## 五、从零到跑通：语言与运行时实操 SOP

> 这一章是“执行线”，前面的章节是“知识点”。按顺序做一遍，你会把 .mm 混编、objc_msgSend、消息转发都亲手碰一次。

### 5.0 全景步骤总览（收藏这张）

```
第 0 步 准备 Apple 芯片 Mac / Xcode 27.2 / Apple ID
   → 第 1 步 新建 iOS App 工程
   → 第 2 步 加入 Objective-C 与 .mm 混编
   → 第 3 步 断点观察 objc_msgSend
   → 第 4 步 跑通消息转发三道防线
   → 收尾：内存与性能实验见姊妹篇
```

### 5.1 第 0 步：准备环境

需要一台 Apple 芯片的 Mac（Xcode 27 仅支持 Apple 芯片；Intel Mac 请用 Xcode 26 及更早版本，菜单路径可能不同）、macOS 26.6 或更高版本、Xcode 27.2。Xcode 可以从 Mac App Store 安装，也可以到 `developer.apple.com/download` 下载。首次启动会要求安装附加组件和同意许可，磁盘至少留 40GB。模拟器运行时在 `Xcode → Settings → Components` 里安装（旧版本可能叫 `Platforms`）。下载 Xcode、模拟器运行时和依赖需要稳定网络；离线环境要提前准备好完整安装包。本文示例不依赖第三方库；如果你的工程用 CocoaPods 或 Swift Package Manager，先把依赖装好再开始。

验证安装：

```bash
xcodebuild -version      # 应显示 Xcode 27.2
swift --version          # 应显示 Swift 6.4.2
```

如果命令找不到，打开 `Xcode → Settings → Locations`，确认 `Command Line Tools` 选中了 Xcode 27.2。

没有付费开发者账号也能做本文的实验：用普通 Apple ID 在真机上跑免费签名即可。真机调试需要在 iPhone 上打开“开发者模式”（`设置 → 隐私与安全性 → 开发者模式`），并在 Xcode 的 `Window → Devices and Simulators` 里信任设备。真机系统要在 iOS 17 或更高版本（Xcode 27 的设备调试限制）；设备太旧时，可以先用模拟器跑通流程。签名细节见《iOS 签名与出包知识体系》。

### 5.2 第 1 步：新建工程

`File → New → Project`，选 `iOS → App`，Product Name 填 `RuntimeLab`，Interface 选 `Storyboard`（本文的 UIKit 示例更直观），Language 选 `Swift`，Storage 选 `None`，Testing System 选 `None`（想练测试可以保留 XCTest），点 `Next` 保存。

`Build`（`⌘B`）一次，确认模拟器能跑起来。如果这一步就报错，先解决签名/模拟器选择问题，不要往下走。

### 5.3 第 2 步：加入 Objective-C 与 .mm 混编

1. `File → New → File`，选 `Cocoa Touch Class`，Class 填 `Cat`，Subclass of 填 `NSObject`，Language 选 `Objective-C`。
2. Xcode 会问是否创建桥接头，选 `Create Bridging Header`。在生成的 `RuntimeLab-Bridging-Header.h` 里加一行 `#import "Cat.h"`，否则 Swift 看不到 `Cat`。
3. 在 `Cat.h` 里声明 `- (void)speak;`，在 `Cat.m` 里实现 `NSLog(@"meow")`。
4. 再 `File → New → File`，选 `Cocoa Touch Class`，Class 填 `FooEngineBridge`，Language 选 `Objective-C`，创建后把 `.m` 改成 `.mm`。
5. 按 1.5 节的四个文件，把 C++ 的 `FooEngine` 和桥接类补全。
6. 回到 `ViewController.swift`。桥接头配置正确的话，不需要在 Swift 里再 `import`，直接就能用 `Cat()` 和 `FooEngineBridge()`；调用一次，确认控制台有输出。

完成后你会看到：Swift 能创建 Objective-C 的 `Cat`，也能通过 `.mm` 桥接类调用 C++ 的 `greet`。

### 5.4 第 3 步：断点观察 objc_msgSend

在 `ViewController.swift` 的 `viewDidLoad` 里写：

```swift
let cat = Cat()
cat.speak()
```

在 `cat.speak()` 这一行下断点，运行。断住后打开 `Debug → Debug Workflow → Always Show Disassembly`，再执行 `Debug → Step Into`。你会看到汇编里出现 `objc_msgSend`，这就是“消息发送”的真身。

如果直接 Step Over，它会在一次调用后返回；Step Into 才能进到运行时。看不懂汇编没关系，只要确认“Swift 调 Objective-C 方法会经过 `objc_msgSend`”这件事即可。

### 5.5 第 4 步：跑通消息转发三道防线

新建一个 `ForwardProxy` 类，声明一个 `helper` 属性（类型用 `Cat`），把 3.3 和 3.4 节的 `forwardingTargetForSelector:`、`methodSignatureForSelector:`、`forwardInvocation:` 抄进去。然后：

1. 创建 `ForwardProxy`，调用一个 `Cat` 有、`ForwardProxy` 没有实现的方法。在 Objective-C 测试代码里可以直接写 `[proxy speak]`；在 Swift 里用 `proxy.perform(NSSelectorFromString("speak"))`。因为 `ForwardProxy` 没实现 `speak`，运行时才会进入转发。
2. 运行，确认消息被转给 `helper`，控制台打印 `meow`。
3. 注释掉三道防线，再运行一次，你会看到 `unrecognized selector sent to instance` 崩溃。

这一步的目的是让你亲眼看到：转发不是编译期行为，而是运行时补救。

### 5.6 内存与性能实验

循环引用、Memory Graph、Instruments 的实操在姊妹篇《iOS 内存与性能调优知识体系》第 3 章，本文不重复。

**SOP 常见失败**（卡住时先查这张）：

| 步骤 | 常见现象 | 先查什么 |
|---|---|---|
| 第 0 步 | `xcodebuild` 找不到 | `Xcode → Settings → Locations` 的 Command Line Tools |
| 第 1 步 | 工程跑不起来 | 模拟器选择、签名、Bundle Identifier（App 唯一标识） |
| 第 2 步 | 桥接头找不到 / 头文件报错 | `Build Settings → Objective-C Bridging Header`；公共头不要 include C++ 头 |
| 第 2 步 | `Undefined symbol` | 文件的 Target Membership、静态库架构切片 |
| 第 3 步 | 断点进不到 `objc_msgSend` | 用 Step Into，并打开 Always Show Disassembly |
| 第 4 步 | 仍然 `unrecognized selector` | 方法名和冒号、对象真实类型、三道防线是否都实现 |

---

## 六、常见报错与排查

> 这些报错分布在消息发送、Swift 可选值、链接和构建四个阶段。内存相关的 EXC_BAD_ACCESS、deinit 不打印、Leaks、Jetsam 见姊妹篇第 4 章。

### 6.1 报错一：unrecognized selector sent to instance

```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException',
reason: '-[Cat speak]: unrecognized selector sent to instance 0x600000c12345'
```

这条报错的意思是：运行时把 `speak` 消息送到了 `Cat` 对象，但整条继承链都没找到能处理它的方法，三道防线也没接住。

**怎么办**：

1. 先检查方法名和冒号。`speak` 和 `speak:` 是两个不同的选择器，少一个冒号就是两个方法。
2. 检查对象类型。你以为它是 `Cat`，实际可能是 `NSNull`、`NSString` 或某个被替换过的代理对象。用 `po` 或断点看真实类型。
3. 检查方法有没有实现。声明了但没写实现，或者实现写在错误的 `@implementation` 里，都会走到这里。
4. 如果确实想转发：快速转发能接住就直接返回目标对象；接不住时，还要实现 `methodSignatureForSelector:` 和 `forwardInvocation:`。
5. 如果对象已经释放，会看到 `message sent to deallocated instance`，按姊妹篇《iOS 内存与性能调优知识体系》的 EXC_BAD_ACCESS 一节处理。

**预防**：少用 `performSelector` 调用未知方法；必须用时先 `respondsToSelector:`；怀疑对象已释放时打开 `Zombie Objects`（僵尸对象，把已释放对象保留下来方便定位）复现。

### 6.2 报错二：Unexpectedly found nil while unwrapping an Optional value

```
Fatal error: Unexpectedly found nil while unwrapping an Optional value
```

这是 Swift 的可选值（Optional）为 `nil`，但你用了 `!` 强制解包。常见于 IBOutlet 没连上、网络返回缺字段、字典取值失败。

**怎么办**：

1. 看崩溃行，把 `!` 改成 `if let` 或 `guard let`。
2. 如果是 `@IBOutlet`，检查 Storyboard/XIB 里的连接是否断了；`viewDidLoad` 之前访问 IBOutlet 也可能还没加载。
3. 如果是 JSON 解析，检查字段名、类型、是否可能为 `null`；用 `Codable` 的可选字段接。
4. 如果是 `try!` 或 `as!`，换成 `do/catch` 和 `as?`。

**预防**：业务代码尽量不用 `!`；IB 连接改动后跑一次 UI 测试；对不可信数据一律按可选处理。

### 6.3 报错三：dyld: Library not loaded

```
dyld: Library not loaded: @rpath/Foo.framework/Foo
  Referenced from: /var/containers/Bundle/Application/.../RuntimeLab
  Reason: image not found
```

App 启动时找不到某个动态库。这里的 `dyld` 是动态链接器，负责在启动时加载动态库。常见于 CocoaPods（依赖管理工具）、Carthage（另一种依赖管理工具）/手动集成的 framework（库包）没有正确嵌入，或者 Runpath（运行时库搜索路径）配置不对。

**怎么办**：

1. 选中 Target → `General → Frameworks, Libraries, and Embedded Content`，确认该 framework 是 `Embed & Sign`（嵌入并签名），不是 `Do Not Embed`。
2. 检查 `Build Settings → Runpath Search Paths`（运行时库搜索路径）是否包含 `@executable_path/Frameworks`。
3. `Product → Clean Build Folder`（`⇧⌘K`），删除 App 重装。
4. 用 CocoaPods 的话，回到终端执行 `pod install`，用 `.xcworkspace` 打开。
5. 检查 framework 的架构是否包含当前设备（真机 arm64，模拟器 arm64/x86_64）。

**预防**：动态库统一用 `Embed & Sign`；不要手动删除 Embed 列表里的条目；升级依赖后重新安装。

### 6.4 报错四：Building with bitcode is deprecated

```
Building with bitcode is deprecated. Please update your project and/or target settings to disable bitcode.
```

Xcode 14 起 Bitcode（苹果的中间代码格式）已废弃，App Store 不再接受 Xcode 14 提交的 bitcode。这个警告不影响本地运行，但开启 bitcode 会让构建变慢，提交时也可能直接失败。

**怎么办**：

1. 选中 Target → `Build Settings`，搜索 `Enable Bitcode`，设为 `No`。
2. 如果 `Other C Flags` 里有 `-fembed-bitcode`，删掉。
3. `Product → Clean Build Folder`，重新构建。
4. 第三方库如果强依赖 bitcode，升级到支持新 Xcode 的版本。

**预防**：新工程默认不开启 bitcode；老工程升级 Xcode 时顺手关掉。

> `Undefined symbol`、C/C++ 头文件错误、静态库裁剪等接入问题，见 1.7.5 节。

---

## 七、进阶与业务视角

### 7.1 C++ 在 iOS 里的典型位置

| 场景 | 为什么用 C++ | 常见技术 |
|---|---|---|
| 游戏与图形渲染 | 直接操作内存、榨取 GPU/CPU | Cocos2d-x、Unreal、metal-cpp |
| 音视频编解码 | 计算密集、要求低延迟 | FFmpeg、自研编解码器 |
| 图像处理 | 像素级运算、SIMD 加速 | OpenCV、Core Image 自定义滤镜 |
| AI 推理 | 矩阵运算密集、要求低延迟 | TensorFlow Lite、ONNX Runtime |
| 跨平台核心 | 一次编写，iOS/Android 复用 | 网络层、SQLite、JSON 解析 |
| 硬件/传感器 | 直接处理底层数据 | ARKit 底层、蓝牙协议栈 |
| 复用成熟库 | 避免重复造轮子 | Boost、libcurl、OpenSSL |

正确的分工是：C++ 做“整块计算”，Swift/OC 做 UI、业务编排和系统交互，`.mm` 做桥接。不要把 C++ 当成“更快的 Objective-C”去写业务，跨语言边界和数据拷贝会把优势抵消。

### 7.2 岗位 JD 为什么要求这些

iOS 岗位的 JD 里常见这几条：

- “熟悉 Objective-C / Swift，理解 Runtime 机制”——对应第二章、第三章。
- “熟悉内存管理，能定位内存泄漏”——见姊妹篇第 1 章。
- “有性能优化经验，熟悉 Instruments”——见姊妹篇第 2 章。
- “有 C++ / 跨平台 SDK 经验”——对应 1.5 和 7.1。
- “熟悉 Method Swizzling / AOP / 组件化”——对应 3.7、3.6。

面试时不需要背源码，但要能用自己的话讲清楚“消息怎么找到方法”“找不到怎么办”“C/C++ 怎么接进来”。内存和性能问题见姊妹篇。

### 7.3 面试口述钩子

**问：Objective-C 的方法调用为什么比 C++ 慢？**
答：OC 默认走 `objc_msgSend`，要先查缓存、再查方法列表、再沿继承链找；C++ 在编译期就把地址定好了。OC 的缓存命中后开销很小，真正慢的是冷调用和动态解析。

**问：消息转发有几道防线？**
答：三道。动态方法解析补方法，快速转发换接收者，完整转发用 `NSInvocation` 改参数/返回值。全部失败才 `unrecognized selector`。

内存、循环引用、Instruments 相关的面试题见姊妹篇第 6.2 节。

### 7.4 和其他文档怎么衔接

- 内存、循环引用、Instruments、性能调优：见《iOS 内存与性能调优知识体系》。
- 签名、证书、描述文件、ipa、上架流程：见《iOS 签名与出包知识体系》。
- Unity/Unreal/Cocos/Godot 的 iOS 导出、IL2CPP、dSYM（调试符号文件）：见《iOS 游戏工程与出包知识体系》。
- React Native 的 Bridge、JSI、TurboModule：见《iOS 与 RN 桥接知识体系》。

---

## 附录 A：实操入口地图

| 要做的事 | 位置（界面/命令） | 操作说明 |
|---|---|---|
| 新建 iOS 工程 | `File → New → Project → iOS → App` | Interface 选 Storyboard 或 SwiftUI，Language 选 Swift |
| 新建 Objective-C 类 | `File → New → File → 选 Cocoa Touch Class` | Language 选 Objective-C |
| 新建 C/C++ 文件 | `File → New → File → C File / C++ File` | 记得勾选目标成员（Target Membership） |
| 把 `.m` 改成 `.mm` | 右侧 `File Inspector`（`⌥⌘1`）→ `Identity and Type → Name` | 改完确认 Type 是 Objective-C++ Source |
| 配置桥接头 | `Build Settings → Swift Compiler - General → Objective-C Bridging Header` | 路径指向 `YourApp-Bridging-Header.h` |
| 下断点 | 代码行号左侧点击 | 蓝色箭头表示断点生效 |
| 看消息发送汇编 | `Debug → Debug Workflow → Always Show Disassembly` | 在调用行 Step Into |
| 编辑 Scheme | `Product → Scheme → Edit Scheme`（`⌘<`） | Run/Profile 的配置都在这里 |
| 管理真机 | `Window → Devices and Simulators`（`⇧⌘2`） | 信任设备、看崩溃日志 |
| 清理构建 | `Product → Clean Build Folder`（`⇧⌘K`） | 改签名/依赖后先清一次 |

> Profile、Memory Graph、Organizer 的操作见姊妹篇附录 A。

---

## 附录 B：排障“动手顺序”速查

```
① 先看崩溃类型
   ├─ unrecognized selector     → 查方法名/冒号/对象类型 → 查三道防线
   ├─ nil unwrap                → 找 ! / IBOutlet / JSON 字段
   ├─ dyld: Library not loaded  → 查 Embed & Sign / Runpath
   └─ bitcode 警告              → Enable Bitcode = No

② 再看 C/C++ 接入
   ├─ Undefined symbol          → 查 Target Membership / 架构切片
   ├─ 头文件报错                → 公共头不要 include C++ 头
   └─ 插件类找不到              → 查 Other Linker Flags（-ObjC）
```

内存、循环引用、Instruments 的排障顺序见姊妹篇附录 B。

**新手学习验收清单**（做完打勾）：

- [ ] 能用自己的话解释 `objc_msgSend` 的查找顺序。
- [ ] 能写出消息转发的三道防线，并说明各自开销。
- [ ] 能在 Xcode 里创建 `.mm` 文件并调用 C/C++ 代码。
- [ ] 能解释 Swift 的静态派发、虚表派发、见证表派发和消息派发。
- [ ] 能用断点 + Always Show Disassembly 看到 `objc_msgSend`。
- [ ] 能说清楚 Swift / Objective-C / C / C++ / 其他语言各自适合什么场景。
- [ ] 能处理 `unrecognized selector`、`nil` 解包、`dyld` 找不到库、bitcode 警告。

---

## 官方与参考链接

- [Xcode 27 Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)（Xcode 27 包含 Swift 6.4 与 iOS 27 SDK）
- [Apple Newsroom：Xcode 27 仅支持 Apple 芯片](https://www.apple.com/newsroom/2026/06/apple-aids-app-development-with-new-intelligence-frameworks-and-advanced-tools/)
- [Xcode 27.2 发布记录](https://developer.apple.com/news/releases/?id=10292026a)（2026-10-29）
- [Announcing Swift 6.4.2](https://forums.swift.org/t/announcing-swift-6-4-2/90337)（Xcode 27.2 包含 Swift 6.4.2）
- [About iOS 27 Updates](https://support.apple.com/en-us/149076)（最新 iOS 27.2.1）
- [Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)
- [Messaging（objc_msgSend 与方法查找）](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html)
- [Dynamic Method Resolution](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtDynamicResolution.html)
- [Message Forwarding](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtForwarding.html)
- [objc_msgSend](https://developer.apple.com/documentation/objectivec/objc_msgsend)
- [NSObject.forwardingTarget(for:)](https://developer.apple.com/documentation/objectivec/nsobject-swift.class/forwardingtarget(for:))
- [Objective-C Runtime API](https://developer.apple.com/documentation/objectivec/objective-c-runtime)
- [Write Objective-C Code（.mm 文件说明）](https://developer.apple.com/library/archive/referencelibrary/GettingStarted/RoadMapiOS-Legacy/chapters/WriteObjective-CCode/WriteObjective-CCode/WriteObjective-CCode.html)
- [Adopting Swift 6](https://developer.apple.com/documentation/swift/adoptingswift6)
- [Swift/C++ Interoperability](https://www.swift.org/documentation/cxx-interop/)
- [JavaScriptCore](https://developer.apple.com/documentation/javascriptcore)
- [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview)
- [App Store Review Guidelines（2.5.2 可执行代码）](https://developer.apple.com/app-store/review/guidelines/)
- [Metal](https://developer.apple.com/documentation/metal)
- [Xcode 14 Release Notes（Bitcode 废弃）](https://developer.apple.com/documentation/xcode-release-notes/xcode-14-release-notes)

> 内存与性能相关链接见姊妹篇《iOS 内存与性能调优知识体系》。

> **可靠性说明**：版本号、SDK、菜单路径、Bitcode 政策等“当前事实”以 Apple 官方文档和发布记录为准，核验日期为 2026-11-21；Xcode 菜单在不同大版本间可能微调，找不到时用 Xcode 右上角搜索或 `Help → Search` 搜关键字。循环引用场景、性能优化优先级、面试表述属于通用实践，不同团队和项目会有差异。文中的代码是最小示例，按 Xcode 27.2 / Swift 6.4.2 语法编写，未在真机逐条运行；实际行为以编译器和运行时为准。内存、循环引用、Instruments、性能调优的细节以姊妹篇为准；Flutter、Unity、Kotlin/Native、Rust、Go、Lua/Python 等第三方语言/引擎的接入细节，以各自官方文档为准。
