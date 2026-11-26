# iOS 与 RN 桥接知识体系（新手友好版）

> 覆盖：RN 与 iOS 原生如何共存、两套架构的来龙去脉、桥接代码怎么写（OC + Swift）、数据过桥规则、工程集成、业界工程组织模式（单仓/三仓/容器化）、原生 SDK 封装实战、版本升级、版本适配专题（RN 0.75 + iOS 15）、报错排查、全景 SOP、面试视角。
> 阅读方式：章节按学习顺序编排——第 0~二章建立直觉、搞清自己该学哪条线；三~五章学两条线的写法和过桥规则；六章起是工程、实战、排错和面试。专业名词首次出现都带中英文和大白话解释。
> 信息核验日期：2026-11-20（版本类信息以官方页面为准，来源见文末链接）。
> 姊妹篇：《iOS 签名与出包知识体系》《iOS 游戏工程与出包知识体系》。

---

## 目录

- 0. 一组比喻建立全局
- 一、RN 和原生是怎么"同居"的：双世界模型
- 二、先定位：你处在哪个架构时代
- 三、旧架构：Bridge 和 NativeModule
- 四、新架构：JSI、TurboModule 和 Codegen
- 五、过桥规则：什么数据能过、什么线程能碰
- 六、工程集成与依赖
- 七、实战：把一个原生 SDK 封装成 RN 库
- 八、版本升级怎么做
- 九、常见报错与排查
- 十、从零到调通的全景 SOP
- 十一、高频速查表
- 十二、业务与面试视角
- 附录：实操入口地图（去哪里动手操作）

---

## 0. 一组比喻建立全局

把 RN 混合开发想象成**一栋楼里合租办公的两家公司**：一家只说原生话（OC/Swift），一家只说 JS 话。他们业务上要频繁合作，但语言不通、内存不通，所有往来都得通过中间的"口岸"。

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 双世界模型 | Native + JS Runtime | 楼里的两家公司 | 一个 App 里同时跑着原生世界和 JS 世界 |
| 旧架构桥接 | Bridge | 边境口岸（填单 → 批量安检 → 异步送达） | JS 与原生之间的消息翻译、排队、异步传递 |
| 原生模块 | Native Module | 常驻口岸的翻译官（启动时全员到岗） | JS 按名字点名，翻译官转接原生方法 |
| 事件广播 | Event Emitter | 大喇叭广播站 | 原生主动向 JS 喊话（蓝牙回调、推送等） |
| 回执单 | Promise / Callback | 办结回执 | JS 调原生后等待"办好了/办不成" |
| JSI | JavaScript Interface | C++ 直连专线 | JS 直接握住 C++ 对象，可以同步等回话 |
| TurboModule | Turbo Native Module | 按需上岗的翻译官 | 谁被点到名谁才上岗（懒加载） |
| Fabric | Fabric Renderer | 新边检大楼 | 新架构的渲染器，UI 布局全在 C++ 里完成 |
| Codegen | Codegen（代码生成器） | 两端合同公证处 | 按 TS 合同自动生成两端接口代码 |
| Hermes | Hermes 引擎 | JS 公司的母语执行器 | RN 默认的 JS 引擎 |

串起来一句话：**两个世界本来互相听不懂——旧架构靠口岸异步传纸条，新架构修了直连专线，JS 能直接拨号给原生，甚至当场等回话。**

---

## 一、RN 和原生是怎么"同居"的：双世界模型

### 1.1 一个 App，两个世界

RN 混合 App 在 iOS 上其实是三样东西拼在一起：**原生壳 + RN 运行时 + JS 包**。你写的 UIKit 代码住在原生世界，`App.tsx` 的编译产物住在 JS 世界，它们在同一个进程里，但互相不能直接伸手拿对方的东西。

| 组成 | 英文 | 说明 |
|---|---|---|
| 原生壳 | Native Shell | AppDelegate、导航栈这些 UIKit 部分 |
| RN 根视图 | RCTRootView | JS 界面渲染所在的 UIView 容器 |
| JS 引擎 | Hermes | 执行 JS 代码的虚拟机 |
| JS 包 | JS Bundle | `App.tsx` 的打包产物，Release 包里是 Hermes 字节码 |
| 桥接层 | Bridge（旧）/ JSI + TurboModule（新） | 两个世界的通信设施，本文主角 |

> **新手常见疑问：RN App 里还能写纯原生页面吗？**
> 能，而且很常见。原生导航栈里随时可以推入一个 RN 页面（RCTRootView），很多大厂 App 就是"原生骨架 + 若干 RN 页面"。桥接是双向的：JS 能调原生，原生也能反过来通知 JS。

### 1.2 一张图看懂结构

```
┌───────────────────────────────────────────┐
│ iOS 进程                                   │
│                                           │
│  ┌──────────────┐        ┌──────────────┐ │
│  │ 原生世界       │  桥接  │ JS 世界       │ │
│  │ AppDelegate   │ ←────→ │ Hermes 引擎   │ │
│  │ UIKit / SDK   │        │ App.tsx 逻辑  │ │
│  └──────────────┘        └──────────────┘ │
│     线程不同、语言不同、内存不能直接互访        │
└───────────────────────────────────────────┘
```

### 1.3 四条线程，各自的地盘

| 线程 | 英文 | 谁在上面 | 记住一句 |
|---|---|---|---|
| UI 主线程 | Main Thread | UIKit、所有界面刷新 | 碰 UI 必须在这里 |
| JS 线程 | JS Thread | Hermes 跑业务逻辑 | JS 卡住 = 全 App 卡住 |
| 渲染 | Shadow Thread（旧）→ C++（新） | 布局计算（Yoga，RN 的布局引擎） | 新架构把渲染整条搬进 C++ |
| 模块队列 | Method Queue | 原生模块方法执行 | 不指定就默认一条串行队列 |

大白话记忆法：**JS 在自己房间写作业（JS 线程），想动客厅的东西（UI）必须去客厅（主线程）；写完想交给原生，走专线（桥接）；原生做完想通知你，按门铃（事件）。**

---

## 二、先定位：你处在哪个架构时代

在读任何桥接教程之前，必须先知道一件事：**RN 在 2024~2026 年换了整套通信架构，网上两套画风完全不同的教程不是谁写错了，是时代不同。** 先看一眼时间线，后面学代码你就知道哪条是旧路、哪条是新路。

### 2.1 三年三步走的时间线

| 版本 | 发布 | 发生了什么 |
|---|---|---|
| 0.76 | 2024-10 | **新架构默认开启**（bridgeless，即"无桥模式"——JS 与原生不再经旧 Bridge 传消息）；最低 iOS 从 13.4 提到 **15.1** |
| 0.80 | 2025-06 | **旧架构官方冻结**，开始出现弃用警告 |
| 0.81 | 2025-08 | SafeAreaView 弃用；JavaScriptCore 转社区维护 |
| 0.82 | 2025-10 | **整个框架只跑新架构**，旧架构开关彻底失效 |
| 0.83 | 2025-12 | React 19.2；第一个没有明显 breaking change 的版本 |
| 0.84 | 2026-02 | **Hermes V1 默认**；**iOS 预编译二进制默认**（编译提速）；开始删除旧架构代码；Node 22 起步 |
| 0.85 | 2026-04 | **旧架构代码彻底退出源码库** |
| 0.86 | 2026-06 | Android 15+ 沉浸式；DevTools 改进 |
| 0.87 | 2026-08 | **Strict TypeScript API 成为默认**；**实验性 SwiftPM 支持**（CocoaPods 仍是默认） |
| 0.88 | 2026-10 | **iOS SceneDelegate 生命周期支持**；React 19.3；官方计划 0.89 起 SwiftPM 成为 iOS 默认 |

一句话总结：**0.76 新架构成为默认 → 0.80 冻结旧架构 → 0.82 谁也关不掉 → 0.84/0.85 旧代码被物理删除。** 2026 年起，"要不要迁新架构"不再是问题，"还没迁"才是问题。

### 2.2 这对你意味着什么

- **开新项目、写新库**：直接学第四章的新架构写法，别再按旧教程起步。
- **接手老项目**：第三章的旧写法要能看懂、会迁移——存量代码里到处都是。
- **另外记住版本节奏**：RN 每 2 个月发一个版本，官方只维护**最新 3 个 minor**。停在 Unsupported 版本上意味着没有 bug 修复，安全风险自担。

### 2.3 对号入座：RN 0.75 + 最低 iOS 15 意味着什么

对照上面的时间线看我们公司的组合（RN 0.75 + 最低 iOS 15），有两个关键事实要先摆清楚：

**第一，RN 0.75 是"旧架构默认时代"的最后一版。** 官方在 0.75 发布博客里明确写了：这是最后一个支持最低 iOS 13.4 的版本；同时新架构在那个时代还处于稳定化阶段，需要手动 opt-in（iOS 侧 `RCT_NEW_ARCH_ENABLED=1`）才开启。也就是说，0.75 离分水岭只差一个版本——0.76 起新架构默认开启、最低 iOS 提到 15.1，世界就换了一套规则。

**第二，"最低 iOS 15" 和 RN 0.76+ 存在硬性冲突。** RN 0.76 起要求最低 iOS 15.1（见 2.1 时间线）。如果 App 的 deployment target（App 声明的"最低能装到哪个 iOS 版本"）是 15.0，直接升 0.76+ 在编译期就过不去——想升级，第一步就是把 target 提到 15.1（实际设备损失几乎为零，iOS 15.0 的设备都能在线升级（OTA）到 15.1+，但这是要走决策流程的变更）。

所以我们的真实处境可以概括为：**能用、能跑，但升级窗口在收窄。** 按官方"只维护最新 3 个 minor"的政策，0.75 早已不在支持线内；更重要的是生态在往前走——0.84/0.85 起旧架构代码被官方从源码移除，越来越多三方库只保证新架构兼容。等在原地不是稳定，是慢性出局。而升级是一次"跨分水岭"动作：target 15.0 → 15.1、新架构默认开启、自研桥接模块按第四章迁移、三方库逐一排查兼容性。

### 2.4 低版本与高版本，技术原理差在哪

**先看 RN 版本（0.75 ↔ 0.76+/0.84+）：差别不在功能多少，而在通信原理的代际差。**

| 层面 | RN 0.75（旧架构默认） | 0.76 及以后 |
|---|---|---|
| 通信层 | 旧 Bridge：JSON 序列化、异步、批量传消息（见 3.1） | bridgeless + JSI/TurboModule：直引用、可同步（见 4.1） |
| 启动模型 | 所有原生模块启动时全量注册 | 模块懒加载，首次被访问才初始化 |
| 调试器 | Flipper（Meta 出的旧调试器）时代 | React Native DevTools（0.76 上线） |
| 引擎与构建 | Hermes（0.70 起默认）+ 源码全量编译 | 0.84 起 Hermes V1 + iOS 预编译二进制默认，编译明显提速 |
| JS API | 类型宽松，deep imports 可用 | 0.80 弃用 deep imports，0.87 起 Strict TypeScript API 默认 |

原理上一句话：旧桥是"排队传纸条"，新架构是"C++ 直连专线"。这也决定了**升级到 0.76+ 不是改改配置就能跑**——桥接代码必须按第四章的方式迁移（方法改走 Codegen 协议、事件改走 `CodegenTypes.EventEmitter`），好在旧模块有互操作层（Interop Layer）可以过渡（见 4.3）。

**再看 iOS 版本（15 ↔ 16/17+）：iOS 版本影响的是系统能力与运行行为，不影响桥接原理。**

1. **deployment target 的原理**：编译永远用最新 Xcode/SDK，target 只决定"能装到哪些设备"。想在 iOS 15 上安全使用 16+ 才有的系统能力（如 Live Activities 16.1+、App Intents 16+），原生代码要用 `@available(iOS 16, *)` 做运行时守卫，否则老系统直接崩溃。
2. **桥接层本身不挑 iOS 版本**：Bridge/JSI 是纯代码层，iOS 15 和 17 上行为一致。差异体现在桥接映射到的系统组件上（键盘、弹窗、滚动这些 UIKit 行为随系统演进），所以跨版本真机矩阵测试不可省。
3. **隐私合规与 OS 版本无关**：隐私清单（Privacy Manifest，含 Required Reason API 声明）是 App Store 提交政策，2024 年起强制执行——最低支持 iOS 15 的 App 一样要交（细节见姊妹篇《iOS 签名与出包知识体系》）。

**两件事交叉出来的升级决策点**：iOS target 15.0 是 RN 0.76+ 的门槛（先提 target）；RN 0.75 的旧架构生态在退场（迟早要迁）。两个约束叠加，答案通常是：**短期内守住 0.75 稳定发版，中期立项升级——提 target、桥接模块迁移、生态兼容排查一次做完。**

---

## 三、旧架构：Bridge 和 NativeModule

> 虽然旧架构正在退场，但存量项目、互操作层和面试都绕不开它。这一章的目标是：**理解原理 + 能看懂老代码**。

### 3.1 它是怎么工作的

旧架构的 Bridge 像一个**很讲规矩但效率不高的邮政系统**。以 JS 调用原生 `connect("A1")` 为例，一次完整旅程是这样的：

```
① JS 按名字在模块注册表里找到 RNTDevice（启动时全员到岗）
② 参数被序列化成 JSON 字符串（相当于填了张报关单）
③ 消息进队列，攒一批一起寄到原生
④ 原生拆包（反序列化），在 Method Queue 上执行方法
⑤ 原生干活（要碰 UI 就切主线程）
⑥ 结果再异步寄回 JS（resolve/reject）
```

三个天生特征：**异步**（一切调用都拿不到同步返回值）、**只有 JSON 兼容类型能过桥**、**批量传递**（延迟高但吞吐还行）。

它为什么被换掉？正是上面这三条：JSON 打包拆包开销大；永远异步；启动时所有模块全员到岗，哪怕一个都用不上。

### 3.2 写一个旧架构模块（OC 精简版）

OC 版核心就四件事：声明、注册、暴露方法、登记事件。去掉业务细节后是下面这个最小骨架：

```objc
// RNTDevice.h 里就一句声明：
// @interface RNTDevice : RCTEventEmitter <RCTBridgeModule> @end

// RNTDevice.mm —— 核心骨架
@implementation RNTDevice {
  BOOL _hasListeners;                       // JS 侧有没有人在听广播
}

RCT_EXPORT_MODULE(RNTDevice);               // 注册名 → JS 里 NativeModules.RNTDevice

RCT_EXPORT_METHOD(connect:(NSString *)deviceId           // 暴露给 JS 的方法
                  resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject) // 带这两个参数 = JS 拿到 Promise
{
  // 实际项目里这里调用你的 SDK；成功 resolve、失败 reject
}

- (NSArray<NSString *> *)supportedEvents    // 广播清单：不登记的事件名发出去会崩溃
{
  return @[@"onDeviceFound"];
}

- (void)startObserving { _hasListeners = YES; }  // 第一个 JS 监听者出现时回调
- (void)stopObserving  { _hasListeners = NO;  }

- (void)onDeviceFound:(NSDictionary *)info  // SDK 回调 → 转成对 JS 的广播
{
  if (_hasListeners) {
    [self sendEventWithName:@"onDeviceFound" body:info];
  }
}
@end
```

记住两个新手高频坑：**不加 `RCT_EXPORT_METHOD` 的方法 JS 完全看不见**；**事件名不登记进 `supportedEvents` 会直接崩溃**。

### 3.3 Swift 写法的三个差异点

Swift 写法与 OC 一一对应，但有三处容易踩坑。直接看片段：

```swift
@objc(RNTDevice)                            // ① 注册名写在类名前，而不是宏
final class RNTDevice: RCTEventEmitter {

  @objc static func requiresMainQueueSetup() -> Bool { false }

  @objc func startScan() {                  // ② 每个要暴露的方法都要 @objc
    // 调用你的 SDK
  }

  @objc func connect(_ deviceId: String,
                     resolver resolve: @escaping RCTPromiseResolveBlock,   // ③ Promise
                     rejecter reject: @escaping RCTPromiseRejectBlock) {
  }
}
```

三个差异：**① 注册靠 `@objc(名字)` 而不是宏；② 方法必须加 `@objc`，否则 JS 点不到；③ 类必须继承 NSObject 家族**（RCTEventEmitter 本身就是）。另外 App 工程的 Swift 需要**桥接头**（Bridging Header）才能 `import React`——`Xcode → 项目 → Build Settings → 搜 Bridging Header → 填 <App名>-Bridging-Header.h 路径`。

### 3.4 JS 侧怎么用

```ts
import { NativeModules, NativeEventEmitter } from 'react-native';

const RNTDevice = NativeModules.RNTDevice;  // 运行时按名字查找
await RNTDevice.connect(deviceId);          // Promise

const sub = new NativeEventEmitter(RNTDevice)   // 事件订阅
  .addListener('onDeviceFound', (info) => { /* ... */ });
sub.remove();                               // 用完必须注销，否则泄漏
```

原生反过来通知 JS，除了事件广播还有两条路：JS 把函数传给原生（`RCTResponseSenderBlock` 回调，只能调一次）、以及无模块实例时的全局广播站 `RCTDeviceEventEmitter`。日常九成场景用事件广播就够了。

---

## 四、新架构：JSI、TurboModule 和 Codegen

### 4.1 四件套各管什么

| 概念 | 英文 | 比喻 | 治好了旧架构哪个病 |
|---|---|---|---|
| JSI | JavaScript Interface | C++ 直连专线 | 不再 JSON 打包：JS 直接持有 C++ 对象引用，**能同步调用** |
| TurboModule | Turbo Native Module | 按需上岗的翻译官 | 不再全员到岗：第一次被访问才初始化 |
| Fabric | Fabric Renderer | 新边检大楼 | 渲染管线搬进 C++，UI 更快 |
| Codegen | Codegen | 合同公证处 | 不再口头约定：按 TS 合同生成两端代码，编译期查错 |

其中 **Codegen 是理解新架构写法的钥匙**：你先写一份 TypeScript 合同（spec），它自动生成两端的接口代码。**spec 是唯一事实源**——JS 和原生都照着它干活，名字对不上、类型不对，编译期就报错，不用等运行时炸。

### 4.2 写一个新架构模块（六步走）

**第 1 步：写 TS 合同。** 这份文件是新架构模块的全部入口：

```ts
// src/NativeDevice.ts
import type { TurboModule, CodegenTypes } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export type DeviceInfo = { deviceId: string; name: string };

export interface Spec extends TurboModule {
  connect(deviceId: string): Promise<boolean>;          // 方法
  startScan(): void;
  // 事件：readonly 属性，载荷类型写清楚
  readonly onDeviceFound: CodegenTypes.EventEmitter<DeviceInfo>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('RNTDevice');
```

**第 2 步：在 package.json 里登记 Codegen。**

```json
{
  "codegenConfig": {
    "name": "RNTDeviceSpecs",   // 生成物的命名空间（pod/头文件前缀）
    "type": "modules",
    "jsSrcsDir": "src"          // spec 文件所在目录
  }
}
```

**第 3 步：`pod install`。** Codegen 会自动执行，生成两端接口代码（在库的 example/ios 目录下执行，完整命令见 10.2 第 3 步）。

**第 4 步：写原生实现。** OC++ 只留一薄层胶水，方法签名照着生成的协议写：

```objc
// ios/RNTDevice.mm —— 核心三段：挂专线、写方法、发事件
@implementation RNTDevice

- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params
{   // 固定模板：把模块挂上 JSI 专线
  return std::make_shared<facebook::react::RNTDeviceSpecJSI>(params);
}

- (void)connect:(NSString *)deviceId
        resolve:(RCTPromiseResolveBlock)resolve
          reject:(RCTPromiseRejectBlock)reject
{   // 签名必须与 spec 一致；Promise 在生成接口里拆成 resolve/reject
}

- (void)onDeviceFound:(NSDictionary *)info
{
  [self emitOnDeviceFound:info];   // 继承 SpecBase 后自动获得 emitOnXxx
}
@end
```

> **新手常见疑问：RNTDeviceSpec、SpecBase、SpecJSI 这些名字哪来的？**
> 全是 Codegen 按你的 spec 生成的：JS 注册名 + Spec → 协议名 `RNTDeviceSpec`，基类 `RNTDeviceSpecBase`，JSI 类 `RNTDeviceSpecJSI`。**拿不准就以 `pod install` 后生成的实际文件为准**，在 Pods 的 ReactCodegen 产物里都能找到。

**第 5 步（想用 Swift 写业务时）：官方 Adapter 模式。** Swift 与 C++ 互操作还不完善，官方推荐的做法是：**业务逻辑全部写在 Swift 类里，OC++ 胶水只负责转发**。

```swift
@objcMembers public class DeviceCore: NSObject {   // 必须继承 NSObject
  public func getItem(for key: String) -> String? {
    UserDefaults.standard.string(forKey: key)      // 你的业务逻辑
  }
}
```

```objc
// OC++ 胶水里创建 Swift 实例并转发（Adapter）
#import "MyApp-Swift.h"   // Xcode 自动生成，前缀 = App target 名
@implementation RNTDevice {
  DeviceCore *_core;
}
- (id)init { if (self = [super init]) { _core = [DeviceCore new]; } return self; }
@end
```

四条官方要求记牢：Swift 类必须继承 `NSObject` 并用 `@objcMembers`（或逐个 `@objc`）暴露；OC++ 侧 import 的 `<App名>-Swift.h` 是 Xcode 自动生成的，**App 改名它跟着改**；App 工程需要配桥接头（见 3.3），**库作者分发时不需要**。

**第 6 步：JS 侧使用。** 事件订阅方式和旧架构完全不同——事件像函数一样直接调用：

```ts
import NativeDevice from './src/NativeDevice';

await NativeDevice.connect(deviceId);      // 有完整 TS 类型提示

const sub = NativeDevice.onDeviceFound((info) => {  // 直接调用事件
  console.log(info.deviceId);
});
sub.remove();                              // 组件卸载时注销
```

### 4.3 老模块怎么办

新架构为旧模块提供了互操作层（Interop Layer），让存量代码暂时继续工作——但这是**过渡方案**，旧架构代码 0.84/0.85 起已经从源码里移除。正确的姿势是按 4.2 的六步迁移。另外旧的事件写法（`RCTEventEmitter` + `NativeEventEmitter`）多数场景仍能用，但新写法有类型检查、和 Codegen 一条管线，新代码一律用新写法。

---

## 五、过桥规则：什么数据能过、什么线程能碰

这一章是新旧架构通用的"海关条例"，背下来能避开一大半桥接 bug。

### 5.1 什么数据能过桥

| JS | OC | Swift | 备注 |
|---|---|---|---|
| `string` | `NSString *` | `String` | |
| `number` | `NSNumber` / `NSInteger` / `CGFloat` | `Int` / `Double` | 自动转换 |
| `boolean` | `BOOL`（容器内是 `NSNumber`） | `Bool` | |
| `object` | `NSDictionary *` | `[String: Any]` | 出桥前转普通字典 |
| `array` | `NSArray *` | `[Any]` | |
| `null` / `undefined` | `nil`（容器内是 `NSNull`） | `nil` | **容器内先判 NSNull 再用** |
| `function` | `RCTResponseSenderBlock` | 同左 | 只能调用一次 |
| `Promise` | `RCTPromiseResolveBlock` / `RejectBlock` | 同左 | 参数尾部声明即生效 |
| 自定义对象 | ❌ 过不了 | ❌ | 转成字典/数组/标量再过 |

三条铁律展开说：

1. **自定义对象过不了桥**。SDK 给你的 `CBPeripheral`、自定义 Model，出桥前先转成只含基础类型的字典——顺带把 JS 用不到的字段留在原生，又安全又省。
2. **nil 在容器里会变成 NSNull**。JS 传来 `null`，放进数组/字典后取出来是 `NSNull` 对象而不是 `nil`，直接当 `nil` 用就崩溃。这是新手崩溃高发点第一名。
3. **超大数字丢精度**。JS 的 number 就是 double，超过 2^53 的整数（比如某些订单 ID）过桥会悄悄丢精度——这种用字符串传。

### 5.2 线程铁律

- **碰 UI 必须主线程**：`dispatch_async(dispatch_get_main_queue(), ^{ ... })`。桥接方法默认跑在后台的 Method Queue 上，直接改 UI 会触发 Main Thread Checker 报警甚至崩溃。
- **不指定 methodQueue 就用默认串行队列**：同模块的方法会互相排队，高频场景要自己控制。
- **JS 卡住 = 全 App 卡住**：高频数据（传感器、RSSI）先在原生节流再广播，别把 JS 线程淹了。

### 5.3 通信方式怎么选

| 场景 | 用什么 | 为什么 |
|---|---|---|
| JS 调原生，要结果 | Promise | 语义清晰，一次一果 |
| JS 调原生，不要结果 | 普通方法 | 最轻 |
| 原生主动通知 JS（回调流、推送） | 事件广播 | 一对多、随时发 |
| 高频数据流 | 原生节流后广播 | 防止淹掉 JS |
| 一次性回执 | Callback（RCTResponseSenderBlock） | 只可调用一次 |
| 必须同步拿返回值 | 仅新架构 JSI 支持 | 旧架构做不到 |

---

## 六、工程集成与依赖

### 6.1 一个 RN 库长什么样

```
react-native-device-bridge/
├── package.json   # JS 入口 + codegenConfig + peerDependencies
├── src/           # spec（NativeDevice.ts）+ JS 封装
├── ios/           # OC/Swift 实现 + podspec
├── android/       # Android 侧（本文不展开）
└── example/       # 调试用的示例 App
```

### 6.2 podspec：iOS 库的身份证

宿主 App 的 `pod install` 靠它认识你的库。精简到只剩关键行：

```ruby
require "json"
package = JSON.parse(File.read(File.join(__dir__, "../package.json")))  # 读 package.json 当单一事实源

Pod::Spec.new do |s|
  s.name         = "react-native-device-bridge" # pod 名，autolinking 靠它对上 package.json
  s.version      = package["version"]
  s.platforms    = { :ios => "15.1" }           # RN 0.76 起官方底线就是 15.1
  s.source_files = "ios/**/*.{h,m,mm,swift}"    # 参与编译的文件（Swift 别漏 .swift）
  s.swift_version = "5.0"                       # 混编 Swift 必须声明

  install_modules_dependencies(s)               # 官方推荐的一行：搞定 React 依赖 + Codegen 集成
end
```

> **新手常见疑问：为什么装完库，Xcode 里找不到我的文件？**
> RN 靠 autolinking（自动链接）接入原生库：`pod install` 时扫描 node_modules 里每个库的 package.json/podspec。找不到文件 = autolinking 没认出你的库——先核对 pod 名和 package.json 的 `name` 是否一致，再重跑 `pod install`。

### 6.3 CocoaPods 与 SwiftPM 的现状

| 事项 | 状态（2026-11 核验） |
|---|---|
| CocoaPods | 0.88 及以前仍是 iOS 默认依赖方式 |
| SwiftPM（Swift Package Manager，苹果官方包管理器） | 0.87 起实验性支持（只需 Xcode、不用 Ruby）；官方计划 0.89 成为默认，CocoaPods 后续移除 |
| 预编译二进制 | 0.84 起 iOS 预编译产物默认启用，编译明显提速 |
| 生态动向 | Firebase Apple SDK 13.0.0 起已不再经 CocoaPods 分发——整个生态在向 SPM 走 |

给库作者的建议：0.8x 时代 podspec 仍是标配，但新库建议顺便验证 SwiftPM 兼容，别把 Ruby 工具链写死在 CI 里。这类动态信息以官方升级公告为准。

### 6.4 monorepo 与发布三要点

- **monorepo 的坑**：依赖被提升（hoisting）到仓库根后，Metro（RN 的 JS 打包与开发服务器）找不到 RN 核心模块。解法是在 Metro 配置里显式指定 `watchFolders` 和 `nodeModulesPaths`。
- **peerDependencies 别装错**：`react`、`react-native` 必须声明为 peer（跟随宿主版本），写进 dependencies 会重复安装两份。
- **发布时别忘了 codegenConfig**：它要随包一起发出去，宿主 `pod install` 时才能在本地跑 Codegen。

### 6.5 业界工程组织模式：你们的"三仓"处在哪个位置

先说结论：**"RN 业务仓 + iOS 宿主仓 + Android 宿主仓"三仓独立，是业界通用的 Brownfield（存量集成）模式**，也是国内大厂做"原生壳 + 动态化业务"的主流形态——RN 只交付业务代码，原生端提供运行环境和设备能力。RN 官方文档把这条路叫 "Integration with Existing Apps"。

常见组织方式对号入座：

| 模式 | 结构 | 适用场景 |
|---|---|---|
| Greenfield（RN 主导） | 单仓单 App，`ios/`、`android/` 内嵌在 RN 工程 | RN 是应用主体，原生只是壳 |
| Brownfield / 三仓（我们的模式） | RN 业务仓 + 双端宿主仓各自独立 | 原生是既有 App，RN 承担部分业务页，双端团队分工明确 |
| Monorepo（单仓多包） | 双端宿主 + RN 业务 + 共享库全在一个仓库（workspace 管理） | 团队一体化，跨端改动需要原子提交 |
| 容器化 / 平台化 | 自研 RN 容器统一内置宿主（bundle 加载、热更、监控、路由注册、JSBridge——原生与 JS 的通信通道——白名单），RN 仓只发业务 bundle | 多业务线共用基建（如携程 CRN、美团 MRN 这类自研框架）；是三仓模式的演进方向 |
| Expo（CNG，配置生成原生工程） | 原生工程不再手写、由配置自动生成（CNG / prebuild），原生能力写成 config plugin | RN 主导的新项目；对已有原生宿主的团队不友好 |

bundle 分发有两条路：**内置**（随 App 版本发布）或**远程下发**（热更，走自建分发平台/CDN）。注意一个时代背景：微软 CodePush 官方服务已随 App Center 于 2025-03-31 停服（`react-native-code-push` 仓库也已归档），热更基建要么自建、要么选社区替代方案。

三仓模式的代价主要在"跨仓协作"，业界通常这样补（按优先级）：

1. **原生能力独立成 RN 库**：桥接代码按能力域封装成独立库（蓝牙一个库、支付一个库），发私有 npm 源 + podspec，宿主与 RN 双侧引用——桥接代码才有独立版本和复用性（封装方法见第七章）。
2. **版本对齐机制**：宿主内置的原生模块版本 ≠ bundle 期望的版本。常见做法是兼容矩阵（bundle 声明最低容器版本）+ 运行时上报/协商，不匹配就降级或拦截。
3. **契约先行**：桥接 API 用 TS spec 定义并单独维护（甚至发布类型包），双端照合同实现，避免"口头约定、运行期才炸"。
4. **本地联调链路**：Metro 指向本地 RN 仓、宿主跑模拟器加载本地 bundle；CI 上用轻量壳跑回归。
5. **发版节奏解耦**：宿主走应用市场节奏，bundle 走自己的灰度/热更节奏——这是选三仓而不是单仓的核心原因，别让基建决策破坏它。

演进方向一句话：**三仓 → 容器化（把宿主里的 RN 基建标准化、能力库化）→ 团队形态适合时再考虑 monorepo。**

---

## 七、实战：把一个原生 SDK 封装成 RN 库

以蓝牙/车控类 SDK 为例（车控指令通道几乎全是"原生 SDK → RN 桥接"这个形态）。封装前先对一张表——SDK 的每个"原始形态"都有对应的 RN 翻译方式：

| SDK 的原始形态 | 翻译成 RN 的什么 | 关键动作 |
|---|---|---|
| 同步阻塞方法 | Promise | 设超时并 reject，防 JS 永久 pending |
| delegate 回调流（如 CBCentralManager） | CodegenTypes.EventEmitter | 原生节流/去重后再广播 |
| SDK 自定义对象（CBPeripheral 等） | 出桥前转字典 | 只带 JS 需要的字段 |
| SDK 私有回调队列 | 收敛到 methodQueue 或显式 dispatch | 碰 UI 切主线程 |
| SDK 生命周期 | 模块 invalidate 时释放 | 停扫描、移 observer、断连接 |
| 权限 | Info.plist 权限串 | 蓝牙是 `NSBluetoothAlwaysUsageDescription` |

**七步 SOP**：

```
第 0 步 环境就绪（见 10.1）
第 1 步 建库：npx create-react-native-library@latest react-native-device-bridge
        （社区标准脚手架，podspec + codegenConfig + example 全带）
第 2 步 写 TS spec：方法 + 事件（见 4.2 第 1 步）
第 3 步 接 SDK：podspec 加依赖（SPM/静态库/framework 三选一，别混用）
第 4 步 原生实现：OC++ 薄胶水 + Swift 主体（Adapter，见 4.2 第 5 步）
第 5 步 翻译异步语义：delegate → 事件、block → Promise、切线程
第 6 步 example 里端到端联调：扫描 → 连接 → 收发 → 断开
第 7 步 发布 npm，宿主接入 + 全量回归
```

**过来人踩过的几个坑**：

最容易踩的是**循环引用**。SDK 的 block 回调里直接用了 `self`（也就是模块自己），模块就永远释放不掉。
**怎么办**：OC 写 `__weak typeof(self) weakSelf = self;`，Swift 用 `[weak self]`，没有例外。

其次是 **Promise 悬死**，这个很隐蔽：SDK 的回调可能触发好几次，但 Promise 只能 resolve 一次，第二次调用会被静默吞掉，JS 那边就一直挂着等不到结果。
**怎么办**：加个 `_resolved` 标记位挡一下，并且保证每个分支最后都会走到 resolve 或 reject。

第三个坑只在蓝牙、传感器这类场景出现：**高频回调把 JS 淹了**。RSSI 每秒能来几十次，全量广播出去，JS 线程很快就瘫了。
**怎么办**：节流放在原生侧做，比如 500ms 合并一次再广播。

还有一类是**调用时机问题**。JS 不管你连接没连接，拿到方法就会调，状态判断如果放在 JS 侧，只能猜来猜去。
**怎么办**：把状态判断留在原生——没连接就直接 `reject(@"E_NOT_CONNECTED", ...)`。

最后一个最隐蔽：**热重载之后事件会重复**。Metro 热重载会重建 JS 侧的监听，但原生模块还是那个旧实例，还在照常广播。
**怎么办**：订阅放进 `useEffect`，清理函数里调 `sub.remove()`。

---

## 八、版本升级怎么做

### 8.1 升级七步（以 0.81 → 0.86 为例）

```
第 0 步 git checkout -b upgrade/rn-0.86    # 永远别在主干直接升
第 1 步 读目标区间每个版本的官方博客 Breaking Changes 段落
第 2 步 打开 Upgrade Helper（react-native-community.github.io/upgrade-helper）
        from=0.81.4 to=0.86.x → 逐文件照 diff 改工程模板文件
第 3 步 package.json：react-native、react 按官方配套表改版本
第 4 步 对齐依赖：npx @rnx-kit/align-deps --requirements react-native@0.86
第 5 步 iOS 重来：cd ios && rm -rf Pods Podfile.lock && bundle exec pod install
第 6 步 清缓存重启：npx react-native start --reset-cache
第 7 步 全量回归：重点测桥接模块、事件、三方库的新架构兼容性
```

### 8.2 废弃/变更清单（防旧教程误导）

| 机制 | 变化 | 版本 | 你该怎么办 |
|---|---|---|---|
| 旧架构 Bridge | 冻结 → 只跑新架构 → 源码删除 | 0.80/0.82/0.84-0.85 | 迁移 TurboModule（第四章） |
| `newArchEnabled=false` | 失效，关不掉了 | 0.82 起 | 删掉开关，直接修兼容问题 |
| Flipper 调试器 | 被 React Native DevTools 取代 | 0.76 起 | 升级前先删干净残留的 Flipper 依赖 |
| JavaScriptCore | 默认早已是 Hermes；JSC 转社区维护 | 0.81 起 | 没特殊依赖就留在 Hermes |
| JS deep imports | 弃用 → Strict TypeScript API 默认 | 0.80/0.87 | 改用官方公开导出的 API |
| `SafeAreaView` | 弃用 | 0.81 起 | 改用 react-native-safe-area-context |
| CocoaPods | SwiftPM 实验性并行 → 计划 0.89 默认 | 0.87/0.88 | 关注官方公告，库尽量双兼容 |
| iOS 最低版本 | 13.4 → 15.1 | 0.76 起 | Podfile 和 Xcode 工程同步改 |

---

## 九、常见报错与排查

> 共同规律：桥接问题八成集中在四件事上——**名字对不上、产物没生成、线程不对、类型不兼容**。先判断报错发生在哪个阶段（装依赖 → 编译 → 启动 → 调用），再对号入座。

### 9.1 pod 装不上

```
[!] CocoaPods could not find compatible versions for pod "React-Core"
```
（典型报错，截取）

一句话原理：Podfile.lock 是"上次装货的账本"，账本版本和这次要装的版本对不上。
解法：① `cd ios && rm -rf Pods Podfile.lock && bundle exec pod install` 让账本重记；② 不行再 `pod repo update`；③ 还不行用 `npx @rnx-kit/align-deps` 对齐三方库版本。
预防：升级 RN 后必重装 Pods；lockfile 提交 git，出问题可回滚。

### 9.2 编译期符号冲突

```
duplicate symbol '_OBJC_CLASS_$_XXX' in: ...
```

一句话原理：同一个类被装进 App 两次（静态库一份、源码一份），链接时撞车。原理延伸见姊妹篇《iOS 签名与出包知识体系》第一章。
解法：① 用 `nm -gU libXXX.a | grep XXX` 定位重复来源；② SDK 只以一种形态接入（源码或 framework 二选一），删掉另一份。
预防：接三方 SDK 前先看官方推荐的接入方式，别混用。

### 9.3 React 头文件找不到 / Swift 里看不到 React 类

```
'React/RCTBridgeModule.h' file not found
Cannot find 'RCTEventEmitter' in scope
```

一句话原理：库的 podspec 没声明 React 依赖，或 Swift 没 import 到 OC 世界。
解法：① podspec 加 `install_modules_dependencies(s)`；② Swift 文件顶部 `import React`，App 工程配好桥接头（3.3）；③ 重跑 `pod install`。
预防：新库一律用 `create-react-native-library` 脚手架生成 podspec，别手写裸模板。

### 9.4 新架构下模块找不到（最高频）

```
Invariant Violation: TurboModuleRegistry.getEnforcing(...):
Could not find TurboModule 'RNTDevice'
```

一句话原理：JS 按合同点名，但原生没上岗——Codegen 没跑，或三处名字对不上。
解法：① 核对名字链路一致：TS spec 的 `'RNTDevice'` ↔ 原生注册名 ↔ import 的 spec 文件；② 重跑 `pod install`（重跑 Codegen）；③ `--reset-cache` 清 JS 缓存。
预防：模块名用常量收敛，原生和 TS 引用同一处定义。

### 9.5 旧 API 拿到 undefined

```
TypeError: Cannot read property 'connect' of undefined
```

一句话原理：JS 按名字找人没找到——autolinking 没接上，或注册名对不上。
解法：① `pod install` 输出里确认有你的 pod；② 核对 `RCT_EXPORT_MODULE(名字)` / `@objc(名字)`；③ 新架构模块改用 spec 导入，别走 `NativeModules`。
预防：业务侧永远 import 你封装的 `src/index.ts`，不裸用 `NativeModules`。

### 9.6 事件名没登记，直接崩溃

```
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException',
reason: 'sendEventWithName... is not a registered event name'
```
（典型报错，截取）

一句话原理：`sendEventWithName` 的名字不在 `supportedEvents` 清单里，RN 拒绝发送并崩溃。
解法：把事件名加进 `supportedEvents`。
预防：事件名定义成常量，登记处和发送处引用同一个常量。

### 9.7 UI 操作不在主线程

```
Main Thread Checker: UI API called on background thread
```

一句话原理：桥接方法跑在后台队列里直接改了 UI，而 UIKit 不是线程安全的。
解法：UI 代码包进 `dispatch_async(dispatch_get_main_queue(), ^{ ... })`。
预防：模块内约定"业务在 Method Queue，UI 一律主线程"，封装一个 `runOnMain` 工具方法。

---

## 十、从零到调通的全景 SOP

> 本节是"执行线"，前面章节是"知识点"，每步括号里指回对应章节。

### 10.0 全景步骤总览（收藏这张）

```
第 0 步 [环境与账号] → 第 1 步 [建库] → 第 2 步 [写 spec] → 第 3 步 [装依赖+Codegen]
→ 第 4 步 [原生实现] → 第 5 步 [JS 封装] → 第 6 步 [联调] → 第 7 步 [接入宿主 App] → 第 8 步 [发布]
```

### 10.1 第 0 步：环境与账号（新手最常卡在"还没动手的准备"）

| 准备项 | 要求 | 说明 |
|---|---|---|
| Mac + Xcode | 最新版 | Command Line Tools：Xcode → Settings → Locations |
| Node | ≥ 22.11 | 用 nvm 的注意 Xcode 找不到 node 的问题（.xcode.env） |
| Watchman | 推荐安装 | `brew install watchman`，热更新提速 |
| CocoaPods | 0.88 及以前需要 | `sudo gem install cocoapods`；CI 里用 `bundle exec` 锁版本 |
| Apple 开发者账号 | 真机调试/分发需要 | $99/年；签名细节见姊妹篇 |
| RN 宿主工程 | 0.8x 系 | `npx @react-native-community/cli@latest init MyApp` |

### 10.2 第 1~8 步

1. **建库**（七）：脚手架生成，自带 example 调试工程。
2. **写 spec**（4.2 第 1 步）：先把方法和事件定义完——JS 侧此时就能开发，编译期即查错。
3. **装依赖 + Codegen**（4.2 第 3 步）：`cd example/ios && bundle exec pod install`。
4. **原生实现**（4.2 第 4~5 步）：OC++ 胶水 + Swift 主体，SDK 接入按七章的对照表。
5. **JS 封装**（4.2 第 6 步）：`src/index.ts` 导出统一 API，业务侧不裸用 `NativeModules`。
6. **联调**：`cd example && npx react-native run-ios`；JS 调试用 React Native DevTools（0.76+ 默认），原生侧直接在 Xcode 打断点。
7. **接入宿主 App**：`npm install` → `pod install`（autolinking 自动接入）→ 按 9.4 核对名字链路。
8. **发布**（6.4）：`npm publish`；回归全场景（方法、Promise、事件、热重载）。

---

## 十一、高频速查表

### 11.1 新旧架构 API 对照表

| 事项 | 旧架构 | 新架构 |
|---|---|---|
| 声明模块 | `RCT_EXPORT_MODULE(Name)` | spec + `codegenConfig` |
| 暴露方法 | `RCT_EXPORT_METHOD` | 实现 Codegen 生成的协议方法 |
| JS 取模块 | `NativeModules.Name` | `TurboModuleRegistry.getEnforcing<Spec>()` |
| 事件定义 | `supportedEvents` + `sendEventWithName` | spec 里 `CodegenTypes.EventEmitter<T>` + 原生 `emitOnXxx` |
| JS 订事件 | `new NativeEventEmitter(module)` | `module.onXxx(cb)` 直接调用 |
| 调用模型 | JSON 序列化、异步、批量 | JSI 直引用、可同步 |
| 模块初始化 | 启动全量注册 | 首次访问懒加载 |

### 11.2 工具链底线

| 项 | 底线 |
|---|---|
| 最低 iOS | 15.1（0.76 起） |
| Xcode | 最新版（官方环境页口径） |
| Node | ≥ 22.11（0.84 起硬性 Node 22） |
| JS 引擎 | Hermes（0.84 起 V1） |
| 版本支持 | 最新 3 个 minor |

### 11.3 版本时间线一句话

`0.76 新架构默认 → 0.80 冻结旧架构 → 0.82 仅新架构 → 0.84 Hermes V1+预编译 → 0.85 删旧桥代码 → 0.87 SwiftPM 实验 → 0.88 SceneDelegate`

---

## 十二、业务与面试视角

这些知识支撑什么岗位：RN 混合开发、跨端业务团队、车联网 App（车控指令通道几乎全是"原生 SDK → RN 桥接"）、电商 App（扫码/支付/推送出桥）。面试高频钩子（关键词 + 可验证细节，细节留到口述）：

1. **旧桥 vs 新架构**：JSON 序列化异步批量 ↔ JSI 直引用可同步；全员到岗 ↔ 懒加载。能讲出"同步返回值只有新架构能做到"就超过多数候选人。
2. **线程模型**：JS 线程 / 主线程 / Method Queue 各管什么；"为什么原生方法里改 UI 要切主线程"是必追问点。
3. **Codegen 类型安全**：spec 是唯一事实源，编译期暴露不一致——对比旧架构"口头约定，运行期才炸"。
4. **迁移故事**：讲一次旧模块迁 TurboModule 的完整过程（spec → Codegen → OC 胶水 → Swift Adapter → 回归），最好带前后性能数据。
5. **事件设计**：delegate → EventEmitter + 节流去重 + `hasListeners` 防泄漏；能讲"高频数据流为什么必须在原生侧节流"。
6. **版本升级实战**：Upgrade Helper + align-deps + Pods 重装组合拳；能背出 0.76 默认、0.82 只跑新架构、0.85 删旧代码这条时间线。
7. **工程化视角**：podspec / autolinking / codegenConfig / peerDependencies / monorepo hoisting——体现"库作者视角"而不只是会用库。
8. **SDK 封装案例**（车联网方向强钩子）：蓝牙 SDK 封装全链路——权限串、状态机留原生、循环引用治理、Promise 超时兜底。

---

## 附录：实操入口地图（去哪里动手操作）

| 要做的事 | 位置（界面/命令） | 易错点 |
|---|---|---|
| 建 RN 工程 | 终端 `npx @react-native-community/cli@latest init MyApp` | 版本用 `--version` 锁定 |
| 建原生库 | 终端 `npx create-react-native-library@latest <name>` | 模板自带 podspec + codegenConfig |
| 写/看 spec | 工程内 `src/NativeXxx.ts` | 唯一事实源；改完必重跑 Codegen |
| 跑 Codegen + 接库 | 终端 `cd ios && bundle exec pod install` | 失败先删 `Pods/` + `Podfile.lock` |
| 配桥接头 | Xcode → 项目 → Build Settings → 搜 "Bridging Header" | App 改名后 `<App>-Swift.h` 前缀跟着变 |
| 看生成物 | Pods 的 ReactCodegen 生成目录 | 拿不准协议名/基类名时以此为准 |
| 启动 JS 服务 | 终端 `npx react-native start --reset-cache` | 疑似缓存问题先加 `--reset-cache` |
| 升级版本 | 浏览器 react-native-community.github.io/upgrade-helper | from/to 选版本号，照 diff 改 |
| 对齐依赖 | 终端 `npx @rnx-kit/align-deps --requirements react-native@0.8x` | 升级第 4 步标配 |
| 查版本信息 | 浏览器 reactnative.dev/docs/releases | 动态信息，以页面为准 |
| 调试 | React Native DevTools（JS）/ Xcode 断点（原生） | Flipper 相关旧教程已过时 |

**排障"动手顺序"速查**（收藏）：

```
① 先判断报错阶段：pod install → 编译 → 启动 → 调用时
② pod/编译失败：删 Pods + Podfile.lock 重装 → pod repo update → align-deps 对齐版本
③ 启动失败（找不到模块）：核对三处名字 → 重跑 pod install → --reset-cache
④ 调用失败（undefined/无响应）：JS 是否走 spec/封装 → 原生 resolve/reject 是否全覆盖
⑤ 事件不通：supportedEvents 核对 → hasListeners → useEffect 清理函数
⑥ 仍无解：Upgrade Helper 看官方 diff + 对照官方 New Architecture 文档逐行比对
```

---

## 官方与参考链接

**官方（以官方页面为准）：**

- 版本发布与支持策略：https://reactnative.dev/docs/releases
- 0.75 发布博客（0.75 是支持最低 iOS 13.4 的最后一个版本）：https://reactnative.dev/blog/2024/08/12/release-0.75
- 0.76 新架构默认与最低 iOS 15.1：https://reactnative.dev/blog/2024/10/23/release-0.76-new-architecture
- 0.80 冻结旧架构：https://reactnative.dev/blog/2025/06/12/react-native-0.80
- 0.82 仅新架构：https://reactnative.dev/blog/2025/10/08/react-native-0.82
- 0.84 Hermes V1 默认 / iOS 预编译默认：https://reactnative.dev/blog/2026/02/11/react-native-0.84
- 0.85 移除旧架构组件：https://reactnative.dev/blog/2026/04/07/react-native-0.85
- 0.87 Strict TS API / 实验性 SwiftPM：https://reactnative.dev/blog/2026/08/11/react-native-0.87
- 0.88 SceneDelegate / React 19.3：https://reactnative.dev/blog/2026/10/19/react-native-0.88
- 环境搭建：https://reactnative.dev/docs/set-up-your-environment
- 新架构 Turbo Native Modules 指南：https://reactnative.dev/docs/the-new-architecture/turbo-native-modules-introduction
- 官方 Swift 模块指南（Adapter 模式）：https://reactnative.dev/docs/the-new-architecture/turbo-modules-with-swift
- 新架构事件（CodegenTypes.EventEmitter）：https://reactnative.dev/docs/the-new-architecture/native-modules-custom-events
- Codegen 原理：https://reactnative.dev/docs/the-new-architecture/what-is-codegen
- Upgrade Helper：https://react-native-community.github.io/upgrade-helper/
- App Center / CodePush 停服公告（2025-03-31，Microsoft Learn）：https://learn.microsoft.com/en-us/appcenter/distribution/codepush/
- Apple 隐私清单（Privacy Manifest Files）：https://developer.apple.com/documentation/bundleresources/privacy_manifest_files

**中文实践来源：**

- React Native 中文网（官方中文文档）：https://reactnative.cn/
- 掘金：RN 调用 iOS 原生方法（Swift 实操）：https://juejin.cn/post/7552432968091090983
- 掘金：RN 项目接入原生 SDK 实操（桥接头/依赖配置）：https://juejin.cn/post/7541714394016792585

**生态参考（动态信息，注意时效）：**

- Firebase Apple SDK 弃用 CocoaPods 分发（SPM 趋势实例）：https://rnfirebase.io/ios-spm
- SwiftPM 0.89 默认计划分析（第三方）：https://www.codeflamme.com/insights/react-native-0-89-swift-package-manager-default-cocoapods-exit-plan
- Nitro Modules（社区高性能 Swift/C++ JSI 方案，进阶可了解）：https://reactnativedirectory.com/packages/react-native-nitro-modules
