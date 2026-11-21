# iOS 游戏出包实战（引擎导出 Xcode 工程）

> 覆盖：游戏在 iOS 上如何运行、主流引擎与开发语言、从引擎到 ipa 的完整编译链路、Unity / Unreal / Cocos Creator / Godot 的导出差异、从零到真机的实操 SOP、导出后高频事故排查、包体/启动/内存/GPU 专项、渠道包重签与多渠道工程化、换 Team、加固、内购与版号、IL2CPP 与 dSYM、研发/发行业务链路与知识映射、岗位与面试视角。
> 适用版本：Unity 6 / Unity 2022 LTS、Unreal Engine 5.x、Cocos Creator 3.8、Godot 4.x；Xcode 26+。
> 文档边界：以 iOS 出包为主线；引擎安装、上架审核、商务合作只讲到"能看懂、能交接"的程度，不展开行业经营细节。
> 姊妹篇：《iOS 签名与出包知识体系.md》——Mach-O、证书、签名、描述文件、ipa、上传校验、上架 SOP 等通用知识都在那边；本文只在必要处引用（写作"主文档 x 节"），不重复展开。
> 阅读方式：第 0 节先建立直觉；第一到三节讲运行原理、开发语言和流水线；第四节看各引擎差异；第五节开始照着动手；第六到八节处理游戏特有的签名、事故与性能；第九节做速查，第十节讲研发/发行业务怎么和这些技术对应。
> 信息核验日期：2026-11-21（关键事实来源见文末「官方与参考链接」）。

---

## 目录

- 0. 先用一组比喻建立全局
- 一、游戏在 iOS 上是怎么跑起来的
- 二、游戏是用什么语言开发的
- 三、总体流水线：引擎 → Xcode 工程 → ipa
- 四、主流引擎与 iOS 导出方式（Unity / Unreal / Cocos Creator / Godot）
- 五、从零到真机：一套可照做的 SOP
- 六、游戏特有的签名场景
- 七、IL2CPP 与 dSYM 管理
- 八、导出后的高频事故与性能排查
- 九、高频速查表
- 十、业务视角：研发公司、发行公司与知识映射
- 附录：实操入口地图
- 官方与参考链接

---

## 0. 先用一组比喻建立全局

把"做一款 iOS 游戏"想象成"出版一本要过海关的书"：

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 游戏引擎 | Game Engine | 预制件工厂 | 把代码、美术、音频组装成可运行的游戏 |
| Xcode 工程 | Xcode Project | 施工图纸 + 半成品 | 记录要编译哪些文件、链接哪些库 |
| 编译器 | Compiler（Clang / swiftc） | 翻译官 | 把 C/C++/OC/Swift 翻译成 ARM64 机器码 |
| 链接器 | Linker（ld） | 装订工 | 把目标文件和库缝成一个可执行文件 |
| Mach-O | Mach-O | 成品书 | iOS 能加载的二进制格式 |
| 代码签名 | Code Signing | 防伪封条 | 证明包没被改过、是谁签的 |
| 描述文件 | Provisioning Profile | 门禁卡 | 决定哪台设备能装哪个包 |
| 沙盒 | Sandbox | 独立单间 | 每个 App 只能读写自己的目录 |
| ipa | iOS App Store Package | 快递盒 | 最终分发用的压缩包 |

串起来一句话：**引擎负责把游戏做出来，Xcode（苹果官方的开发工具）负责把它翻译成 iPhone 能执行的机器码并盖章，iOS 负责验章、分配单间、加载运行。**

---

## 一、游戏在 iOS 上是怎么跑起来的

### 1.1 四个阶段

```
打包（引擎 + Xcode） → 安检（签名校验） → 安装（沙盒） → 启动（加载 + 渲染）
```

| 阶段 | 发生了什么 | 结果 |
|---|---|---|
| 打包 | 引擎先导出 Xcode 工程；Xcode 把源码、引擎库、第三方库编译链接成 Mach-O（iOS 的可执行文件格式），再和资源一起装进 `.app`，签名后压成 `.ipa` | 一个签过名的 ipa（本质是改了后缀的 zip，主文档 5.1） |
| 安检 | 安装时系统核对证书（Certificate，签名者身份）、描述文件（Provisioning Profile，设备与 App 的绑定关系）、能力授权（Entitlements，App 能用哪些系统能力）和签名 | 全部匹配才安装，任何一处不匹配就拒绝（主文档 4.3） |
| 安装 | `.app` 被放进系统管理的 Bundle 目录（只读），系统同时给 App 分配一个独立的数据容器 | 存档、缓存、日志只能写进自己的沙盒（Sandbox） |
| 启动 | 点桌面图标 → dyld（动态加载器，Dynamic Loader）加载 Mach-O 和依赖的动态库 → 引擎初始化 → 进入每帧循环 | CPU 跑游戏逻辑，GPU 通过 Metal（苹果的图形接口）渲染画面 |

> **新手常见疑问：为什么 Windows 上做不了 iOS 出包？**
> 因为最后的编译、签名、打包必须用 Xcode，而 Xcode 只运行在 macOS 上。Windows 最多只能完成引擎侧的工程导出（是否支持取决于引擎），最后一步一定要交给 Mac 或云端 Mac 构建机。

### 1.2 引擎和 Xcode 各管什么

| 阶段 | 谁来做 | 产出 |
|---|---|---|
| 写游戏逻辑 | 你 + 引擎编辑器 | C# / C++ / TS 等源码和资源 |
| 跨平台翻译 | 引擎 | C++ 源码（或 JS）、引擎原生库、Xcode 工程 |
| 编译 | Xcode 里的 Clang / swiftc | ARM64 机器码（`.o` 目标文件） |
| 链接 | Xcode 里的 ld | Mach-O 可执行文件 |
| 打包 + 签名 | Xcode + codesign | 已签名的 `.app` / `.ipa` |
| 验签 + 运行 | iOS 系统 | 安装、沙盒、加载执行 |

一句话：**引擎解决"跨平台"，Xcode 解决"在 iPhone 上真正跑起来"。** 换句话说，引擎只是把"写 Swift/OC"替换成"生成引擎运行时 + 桥接层"，出包的后半段和普通 App 完全一样。

### 1.3 编译链路走读

```
你的代码（C# / C++ / TS）
  ↓ 引擎
Xcode 工程：C++ 源码（或 JS）+ 引擎原生库 + 资源 + project.pbxproj（Xcode 工程描述文件）
  ↓ Clang 预处理：展开宏、删除注释，生成 .i 文件
  ↓ 语法分析：生成抽象语法树（AST）
  ↓ 生成 LLVM IR：跨平台的中间表示
  ↓ 生成 ARM64 汇编和机器码：产出多个 .o 目标文件
  ↓ ld 链接：符号解析 + 地址重定位，把 .o 和库缝在一起
Mach-O 可执行文件（arm64）
  ↓ 装进 .app（+ Assets、Info.plist）
  ↓ codesign + 描述文件
.ipa
```

三条容易混淆的规则：

1. **真机只认 arm64**（ARM 64 位架构）；模拟器是 arm64（Apple Silicon Mac）或 x86_64（Intel Mac）。包里混进模拟器架构，上传会被拒（见 8.1）。
2. **`.a` 静态库在链接时被拆开拷进主程序**；`.framework` 动态库运行时才加载，必须 Embed & Sign（嵌入并签名，见 8.2）。
3. **签名由内向外**：先签 framework，再签 `.app`，最后压 ipa。

### 1.4 安装之后：沙盒、内存、渲染

| 运行期机制 | 发生了什么 | 对游戏的影响 |
|---|---|---|
| 沙盒 Sandbox | 每个 App 一个只读 Bundle 目录 + 一个可写 Data 容器 | 存档、日志、缓存只能写自己的目录 |
| 签名校验 | 安装时校验一次；启动时 dyld 还会校验 | 改包不重签 → 启动闪退（见 8.5） |
| 内存回收 Jetsam | iOS 没有交换分区，内存超限时系统直接杀进程 | 大纹理、长音频、加载峰值是闪退高发点 |
| Metal 渲染 | CPU 提交绘制命令，GPU 用 Metal 渲染 | 卡顿要先分清是逻辑瓶颈还是 GPU 瓶颈 |
| 后台挂起 | 切后台后很快被挂起，声明了后台模式才能继续跑 | 下载、语音、定位要单独开 Background Modes（后台模式） |

沙盒里最常用的三个目录：

```
<App 数据容器>/Documents/         用户数据；默认会被 iCloud 备份
<App 数据容器>/Library/Caches/    缓存；不被备份，系统可清理
<App 数据容器>/tmp/               临时文件；随时可能被清理
```

游戏存档建议放 `Library/Application Support/`（目录要自己创建）：不会被系统当缓存清掉，也不会像 Documents 那样在开启文件共享后暴露给"文件"App。

---

## 二、游戏是用什么语言开发的

### 2.1 三类语言，三条路

| 路线 | 语言 | 常见场景 | 出包时发生什么 |
|---|---|---|---|
| iOS 原生 | Swift / Objective-C（可混编 C/C++） | 原生小游戏、SDK（Software Development Kit，软件开发工具包）桥接层、引擎插件 | Xcode 直接编译成 arm64 |
| 引擎主语言 | C#（Unity）、C++（Unreal）、TypeScript（Cocos Creator）、GDScript（Godot 的脚本语言）/ C#（Godot） | 绝大多数商业手游 | 引擎先翻译成 C++ / JS，再交给 Xcode |
| 业务脚本 | Lua / Python | 任务、活动、UI、数值逻辑 | 作为资源随包，运行时由引擎解释 |

### 2.2 "C++ 核心 + 脚本"为什么常见

渲染、物理、动画对性能要求高，用 C++ 写；任务、背包、活动页改得频繁，用 Lua/Python 写。前者保证帧率，后者让策划和运营不用等完整出包就能改逻辑。很多大型商业游戏用的是这个组合。

### 2.3 不会 Swift 也能做 iOS 游戏吗

能。引擎会把 C# / C++ / TS 翻译成 iOS 能执行的代码，你不需要手写 Swift。但接支付、广告、统计等原生 SDK，以及排查线上崩溃时，至少要能读懂 OC/Swift/C++ 的桥接代码——这也是游戏 iOS 岗和纯业务岗的分水岭。

### 2.4 热更新要过审核这一关

审核指南 2.5.2 要求 App 自包含，不得下载、安装或执行会引入或更改 App 特性或功能的代码。资源热更、配置热更一般没问题；用 Lua 等脚本热更业务逻辑时，要按这条规则评估。最稳妥的做法是把会影响核心功能的逻辑随包提交审核，不要等被拒再改架构。

---

## 三、总体流水线：引擎 → Xcode 工程 → ipa

**为什么必须导出 Xcode 工程**：iOS 上一切 App 最终都得是 Xcode 编译出来、Apple 能校验签名的原生包。引擎负责生成游戏逻辑和运行时，但不能绕开 Xcode 直接产出一个能上架的 ipa，所以标准做法是：

```
引擎构建（IL2CPP / UBT / Cocos Build / Godot Export）
    → 导出 .xcodeproj（Xcode 工程）
    → Xcode 编译链接（arm64 Mach-O，含引擎运行时）
    → codesign 签名 + Provisioning Profile
    → .ipa → TestFlight / App Store / 渠道
```

两条落地路径：

- **本地 Mac**：引擎 + Xcode 一条龙，适合调试和接 SDK。
- **云端 / CI**：在 Windows 上导出工程（引擎支持的话），再把工程交给 Mac 构建机（Jenkins 等持续集成工具、GitHub Actions 的 macOS runner、Unity Build Automation（Unity 的云端构建服务））。CI 是持续集成（Continuous Integration）的缩写；签名材料要提前装进构建机（主文档 6.2 / 7.3）。

**当前版本门槛**：2026 年 4 月 28 日起，上传到 App Store Connect（苹果的 App 管理后台）的 iOS App 必须用 iOS 26 SDK（即 Xcode 26 或更高）构建；Xcode 26 需要 macOS Sequoia 15.6 或更高。还在用旧 Xcode 的团队，上传时会收到 ITMS-90725（苹果上传校验错误码，见 8.7）。

---

## 四、主流引擎与 iOS 导出方式

### 4.1 全景速览

| 引擎 | 主语言 | 授权 | iOS 导出入口 | 产物 |
|---|---|---|---|---|
| Unity | C# | 闭源；个人版免费，超过收入门槛需付费 | File → Build Profiles → iOS → Build | Xcode 工程 |
| Unreal Engine | C++ / 蓝图 | 闭源；源码对授权用户开放，超过收入门槛分成 | Platforms → iOS → Package Project | ipa 或 Xcode 工程 |
| Cocos Creator | TypeScript | 引擎开源、编辑器免费 | 构建发布 → iOS → 构建 | `build/ios/proj/*.xcodeproj` |
| Godot | GDScript / C# | MIT 开源 | 项目 → 导出 → iOS 预设 | Xcode 工程 |

按目标选：

- 要生态、要接各种国内 SDK、团队会 C#：Unity。
- 要高品质 3D、团队会 C++：Unreal。
- 做 2D、要兼顾微信/抖音小游戏：Cocos Creator。
- 独立开发、想零授权成本、能接受 iOS 生态较弱：Godot。

其他引擎（GameMaker、CryEngine、O3DE、Phaser、Bevy、LibGDX 等）各有定位；其中能在 iOS 上出包的，最终同样要生成 Xcode 工程，具体支持情况以各自文档为准。

### 4.2 Unity

**导出入口**：Unity 6 是 `File → Build Profiles`（文件 → 生成配置文件）；Unity 2022 及更早是 `File → Build Settings`（文件 → 生成设置）。

1. `Add Build Profile` → 平台选 iOS → `Switch Profile`；
2. 在 `Edit → Project Settings → Player → iOS` 填 Bundle Identifier（包标识符，通常叫包名）；
3. 点 `Build`，选一个空目录，Unity 生成 Xcode 工程；
4. 点 `Build and Run` 会继续调用 Xcode，编译并装到已连接的真机。

> **注意**：iOS 没有"Unity 直接吐出一个 ipa"的黑盒按钮。`Build` 的产物一定是 Xcode 工程，ipa 由 Xcode 后续产出。

**IL2CPP（Intermediate Language To C++）**：iOS 只能用 IL2CPP，也就是提前编译（AOT，Ahead-Of-Time）。C# 先转成 C++，再由 Xcode 编译成 `libGameAssembly.a`（游戏逻辑）和 `il2cpp.a`（运行时）链进包。原因：iOS 第三方 App 不能依赖 JIT（即时编译，Just-In-Time），审核指南也禁止下载可执行代码。

**工程结构**：Unity 生成的工程有两个 target（构建目标）：

- `Unity-iPhone`：壳工程，负责最终 App；
- `UnityFramework`：引擎运行时 + 你的游戏代码，产物是 `UnityFramework.framework`。

接第三方 SDK 时，多数 framework 要加到 `UnityFramework` target，而不是壳 target，这是新手最容易加错的地方。

**PostProcessBuild（构建后处理回调）**：导出后自动执行的 C# 回调，用来代码化地改工程、加 framework、改 Info.plist（App 的配置文件），避免每次出包手工改。最小示例：

```csharp
using UnityEditor;
using UnityEditor.Callbacks;
using UnityEditor.iOS.Xcode;

public static class BuildPostProcessor {
    [PostProcessBuild]
    public static void OnPostProcessBuild(BuildTarget target, string path) {
        // 1. 改 Xcode 工程：给 UnityFramework target 加 framework
        var project = new PBXProject();
        project.ReadFromFile(PBXProject.GetPBXProjectPath(path));
        var frameworkTarget = project.GetUnityFrameworkTargetGuid();
        project.AddFrameworkToProject(frameworkTarget, "AdSupport.framework", false);
        project.WriteToFile(PBXProject.GetPBXProjectPath(path));

        // 2. 改 Info.plist：加权限文案
        var plist = new PlistDocument();
        plist.ReadFromFile(path + "/Info.plist");
        plist.root.SetString("NSCameraUsageDescription", "用于头像上传");
        plist.WriteToFile(path + "/Info.plist");
    }
}
```

**原生桥接（Native Bridging）**：C# 调原生用 `[DllImport("__Internal")]` 调 OC/C 函数；原生回调 C# 用 `UnitySendMessage("GameObject名", "方法名", "参数")`。支付、登录、广告回调基本都走这两条通道。

### 4.3 Unreal

- 出包必须在 macOS + Xcode 环境，另需 Apple 开发者账号。
- 入口：`Platforms → iOS → Package Project`（平台 → iOS → 打包项目），签名配置正确时直接产出可安装包；调试用 `Platforms → iOS → Prepare for Debugging`。
- C++ 工程可以在 `.uproject` 上右键 `Generate Xcode Project`，用 Xcode 断点调试原生代码。
- 权限文案、framework、插件配置在 `Project Settings → Platforms → iOS` 和模块的 `Build.cs` 里；改工程不如 Unity 直接。
- UBT（Unreal Build Tool，虚幻构建工具）接管编译流程，最终仍由 Xcode 工具链完成打包和签名。

### 4.4 Cocos Creator

- 入口：编辑器 `构建发布` 面板 → 平台选 `iOS` → 填 Bundle Identifier → `构建`。
- 产物：`build/ios/proj/<项目名>.xcodeproj`，用 Xcode 打开后配签名即可。
- 结构接近手写原生工程（C++ 引擎 + JS 业务代码），接 SDK 相对直观。
- 同一套 TS 代码可以发布成微信/抖音小游戏，但小游戏不是 ipa；要上 App Store，仍然要出原生 iOS 包并走本文的签名流程。

### 4.5 Godot

- 入口：`项目 → 导出`（Project → Export）→ `添加` → iOS 预设 → `导出项目`，产物是 Xcode 工程。
- 前提：macOS + Xcode + 对应版本的导出模板（`编辑器 → 管理导出模板`）。
- GDScript 项目最稳；C# 项目从 Godot 4.2 起可以导出 iOS，但官方仍标为实验性，限制较多，商业项目要先用目标版本实测。
- 接原生 SDK 需要自己写 iOS 原生插件（Objective-C/Swift 静态库），生态和资料比 Unity/Cocos 少。

---

## 五、从零到真机：一套可照做的 SOP

### 5.0 全景步骤（收藏这张）

```
第 0 步 准备：Mac + Apple 开发者账号 + Xcode 26+ + 引擎
第 1 步 引擎导出 Xcode 工程
第 2 步 Xcode 配 Bundle ID、Team、自动签名
第 3 步 连真机跑通 Debug
第 4 步 Product → Archive 出 Release 包
第 5 步 Organizer 分发 / 上传 App Store Connect
第 6 步 TestFlight 或蒲公英发给测试
第 7 步 提审上架（主文档第九章）
```

### 5.1 第 0 步：前置准备

| 准备项 | 要求 | 说明 |
|---|---|---|
| Mac | Apple Silicon 更省事；macOS Sequoia 15.6+ | Xcode 26 的系统要求；更高版本 Xcode 以发布说明为准 |
| 磁盘 | 建议预留 100GB 以上 | Xcode、模拟器、引擎、工程、Archive 都很占空间 |
| Apple Developer Program | 个人或组织，年费 99 美元（中国区 688 元/年） | 组织账号需要 D-U-N-S（邓白氏）编号 |
| Xcode | 26 或更高 | 低于这个版本无法上传 App Store |
| Xcode 首次启动 | 登录 Apple ID、同意协议、装命令行工具、下载 iOS 模拟器 | 见下面的"Xcode 首次启动 5 步" |
| 引擎模块 | Unity 装 iOS Build Support；Godot 装导出模板；Unreal/Cocos 按官方要求 | 见下面的"引擎侧前置" |
| CocoaPods | 只在工程或 SDK 用到时安装 | 很多第三方 SDK 用 Pods 管理依赖 |
| 真机 | 一台 iPhone | 模拟器不能验证性能、推送和支付回调 |
| 开发者模式 | iPhone：设置 → 隐私与安全性 → 开发者模式 | iOS 16 起需要手动打开 |

装软件的顺序：Xcode（Mac App Store 或 developer.apple.com/download）→ Xcode 首次启动 → 引擎本体（Unity Hub / Epic Games Launcher / Cocos Dashboard / Godot 官网）→ 引擎的 iOS 模块或导出模板 → CocoaPods（需要时）。

**没有 Mac 怎么办**

| 方案 | 适合 | 注意 |
|---|---|---|
| 本地 Mac | 长期开发 | 需要 macOS 15.6+；磁盘 100GB+ |
| 云 Mac | 没有 Mac，或只是短期出包 | MacinCloud、MacStadium、AWS EC2 Mac 等，按小时/月计费，价格以服务商为准 |
| CI Mac | 团队自动化出包 | GitHub Actions 的 macOS runner、Unity Build Automation；证书和描述文件要预装 |

**Xcode 首次启动 5 步**

1. 打开 Xcode，按提示同意许可协议；也可以在终端执行 `sudo xcodebuild -license accept`。
2. `Xcode → Settings → Accounts` → 左下角 `+` → 登录 Apple ID。
3. 终端执行 `xcode-select --install` 安装命令行工具；如果完整 Xcode 没被选中，执行 `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`。
4. `Xcode → Settings → Components → Simulators`，下载一个 iOS 模拟器运行时。
5. `File → New → Project → iOS → App` 新建一个空白工程，选模拟器跑一次；能跑起来说明 Xcode 环境通了。

**引擎侧前置**

| 引擎 | 要装什么 | 在哪里装 |
|---|---|---|
| Unity | iOS Build Support | Unity Hub → Installs → 版本右侧三个点 → Add Modules → iOS Build Support |
| Unreal | Xcode + 对应 iOS 支持 | 先装 Xcode，再用 `Platforms → iOS` 打包；版本兼容查 Epic 文档 |
| Cocos Creator | Xcode 14+（Cocos 3.8 自带构建用 CMake） | 按 Cocos 原生开发环境文档安装；模拟器调试才需额外配 CMake |
| Godot | 导出模板 | 编辑器 → 管理导出模板 → 下载 |

CocoaPods 安装（只在需要时）：`brew install cocoapods`，或 `sudo gem install cocoapods`；没装 Homebrew 就先按 brew.sh 的说明安装。装完用 `pod --version` 验证。

- Apple Developer Program 注册流程见主文档 9.1；个人账号快，组织账号要 D-U-N-S。
- 免费 Apple ID 也能在 Xcode 里做真机调试，但签名有效期很短，不能上架、不能推送。
- 组织账号的 D-U-N-S 编号由 Dun & Bradstreet 分配，申请免费，但需要时间，别等出包当天才办。
- 国内上架还要提前准备版号、App 备案（见 6.5），它们和技术出包是两条并行的时间线。

### 5.2 第 1 步：引擎导出

| 引擎 | 菜单 | 产物 |
|---|---|---|
| Unity 6 | File → Build Profiles → iOS → Build | 你选的空目录里的 Xcode 工程 |
| Unity 2022 及更早 | File → Build Settings → iOS → Build | 同上 |
| Unreal | Platforms → iOS → Package Project | ipa（签名配置正确时） |
| Cocos Creator | 构建发布 → iOS → 构建 | `build/ios/proj/*.xcodeproj` |
| Godot | 项目 → 导出 → iOS → 导出项目 | Xcode 工程 |

导出后先确认三件事：

1. Bundle Identifier 和 App Store Connect、描述文件里的一致；
2. 版本号（Version）和构建号（Build）已填；
3. 需要的 Capability（能力，如内购、推送、Associated Domains（关联域名））已在引擎或 Xcode 里打开。

导出失败先查这几样：

| 现象 | 大概率原因 | 怎么办 |
|---|---|---|
| Unity Build 按钮灰、提示 iOS module not installed | 没装 iOS Build Support | 按 5.1 的 Unity Hub 步骤装模块 |
| Unity 报 Xcode version not supported | 引擎和 Xcode 版本不匹配 | 升级引擎或 Xcode，查官方兼容表 |
| Unreal 报 Xcode not found | 没装 Xcode 或命令行工具 | 按 5.1 的 Xcode 首次启动配置 |
| Cocos 报 CMake/Python 找不到 | 原生构建依赖没装 | 按 Cocos Creator 3.8 原生构建文档安装 |
| Godot 报 No export template | 没装导出模板 | 编辑器 → 管理导出模板 |
| 报 Bundle Identifier is empty | 没填包名 | Player Settings / 构建面板填 Bundle ID |
| 报 Not enough disk space | 磁盘不够 | 清理空间，建议预留 100GB+ |

导出成功后，Unity 的工程目录大致长这样（其他引擎类似，目录名不同）：

```
MyGame/
├── Unity-iPhone.xcodeproj   ← 双击打开
├── Classes/                 ← OC 桥接、main.mm
├── Data/                    ← 游戏资源
├── Libraries/               ← 引擎和插件库
├── UnityFramework/          ← 引擎运行时 + 游戏代码
└── Info.plist               ← App 配置
```

如果工程是在 Windows 上导出的，要把整个目录（不是只拷 `.xcodeproj`）拷到 Mac；`.xcodeproj` 只是索引，真正的代码和资源在同级目录里。

**Version 和 Build 别填反**

| 字段 | Xcode/Info.plist 键 | 含义 | 例子 |
|---|---|---|---|
| Version | `CFBundleShortVersionString` | 对用户显示的版本号 | 1.2.0 |
| Build | `CFBundleVersion` | 构建号，同一 Version 下必须递增 | 120、121 |

App Store Connect 用 Version 建版本记录，用 Build 区分同版本的多次上传；Build 重复会被拒。

### 5.3 第 2 步：Xcode 配签名

**先选签名方式**

| 方式 | 适合 | 优点 | 缺点 |
|---|---|---|---|
| 自动管理签名 | 新手、单机开发、快速跑真机 | Xcode 自动创建 App ID、证书、描述文件 | 团队/CI 不好控制，换机器可能要重来 |
| 手动签名 | 发行、CI、多人协作 | 证书/描述文件可版本化、可审计 | 步骤多，容易出现 profile 和证书不匹配 |

**自动签名（新手先走这条）**

1. 双击 `.xcodeproj`；有 CocoaPods（iOS 的依赖管理工具）就打开 `.xcworkspace`；
2. 左侧选蓝色工程图标 → `TARGETS` → 你的 App target；
3. `Signing & Capabilities`（签名与能力）→ 勾选 `Automatically manage signing`（自动管理签名）→ `Team` 选你的开发团队；
4. 确认 `Bundle Identifier`（包名，反向域名，如 `com.team.game`；只能字母/数字/连字符/点，不能下划线）；
5. 需要内购、推送等能力，点 `+ Capability` 添加，必须和 App ID 的能力一致；
6. 真机首次安装后，如果提示"不受信任的开发者"，到 iPhone 的 `设置 → 通用 → VPN 与设备管理` 信任你的开发者证书。

**手动签名（发行/CI 用）**

1. 打开 [developer.apple.com](https://developer.apple.com) → `Certificates, Identifiers & Profiles`；
2. `Identifiers → App IDs → +` → 选 `App` → 填 Explicit Bundle ID → 勾选需要的 Capabilities（IAP、Push、Associated Domains）→ Register；
3. `Certificates → +` → 选 `Apple Distribution`（上架/Ad Hoc）或 `Apple Development`（真机调试）→ 按提示用钥匙串访问生成 CSR（证书签名请求：证书助理 → 从证书颁发机构请求证书）→ 下载 `.cer` 并双击安装；团队场景通常直接拿 `.p12`；
4. `Profiles → +` → 选 `App Store Connect`（上架）或 `Ad Hoc`（测试）→ 选 App ID → 选证书 → Ad Hoc 还要选设备 → Generate → Download；
5. 双击下载的 `.mobileprovision` 安装，或在 Xcode 的 `Provisioning Profile` 下拉里选 `Import Profile`；
6. 回到 Xcode → `Signing & Capabilities` → 取消 `Automatically manage signing` → 选 Team、Bundle ID、Provisioning Profile、Signing Certificate；
7. 检查 `embedded.mobileprovision` 和 entitlements 是否匹配，再用 `codesign -dv --verbose=4` 验证。

> **最容易错的三件事**：Bundle ID 和 App ID 不一致；Capability 只在 Xcode 开了、App ID 没开；Ad Hoc profile 忘了勾测试设备。
> Team ID 在 Apple Developer → `Membership` 页面查看；内购商品要到 App Store Connect → 你的 App → `In-App Purchases` 创建。

### 5.4 第 3 步：模拟器 → 真机

**先跑模拟器**

1. Xcode 顶部设备选择器选一个 iPhone 模拟器（如 iPhone 17）→ `Run`（Cmd+R）。
2. 模拟器能跑通，说明代码、资源、签名配置基本没问题；再换真机。
3. 模拟器不能验证性能、推送、真实支付回调，别用它做上线判断。

**再跑真机**

1. 数据线连 iPhone，手机上点"信任此电脑"。
2. iPhone：`设置 → 隐私与安全性 → 开发者模式` → 打开 → 重启 → 确认。
3. Xcode → `Window → Devices and Simulators` → 选中设备，等状态变成 ready。
4. Xcode 顶部设备选择器选你的 iPhone → `Run`。
5. 如果提示"不受信任的开发者"：`设置 → 通用 → VPN 与设备管理` → 选你的开发者 App → 信任。

**拿 UDID（Ad Hoc 测试要用）**

- Xcode → `Window → Devices and Simulators` → 选中设备 → 右侧 `Identifier` 就是 UDID；
- 或 Finder → 左侧选中 iPhone → 点设备信息那一行，会在序列号/UDID 之间切换。

**真机装不上的常见原因**

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 设备不出现在 Xcode | 没信任电脑、没开开发者模式、线只充电 | 换线/换口，重新信任，开开发者模式 |
| Untrusted Developer | 没信任开发者证书 | 设置 → 通用 → VPN 与设备管理 → 信任 |
| 提示 iOS 版本过低 | 工程的 Minimum iOS Version 高于手机系统 | 降低 Deployment Target，或换高版本测试机 |
| 提示 profile 不包含设备 | Ad Hoc/Development profile 没勾这台设备 | 到 Apple Developer 重新勾选设备并生成 profile |
| Bundle ID 被占用 | 包名已被别的 Team 注册 | 换一个唯一包名 |

### 5.5 第 4 步：Archive 出包

1. 顶部设备选择器选 `Any iOS Device (arm64)`（任意 iOS 设备）；
2. `Product → Archive`（产品 → 归档）；
3. 完成后自动打开 `Organizer`（归档管理器），选中最新 Archive；
4. `Distribute App`（分发 App）→ 选分发方式：
   - `App Store Connect`：上架 / TestFlight；
   - `Ad Hoc`（临时分发）：装到已登记 UDID（设备唯一标识）的测试机；
   - `Development`：开发调试；
   - `Enterprise`：仅限企业账号内部分发。
5. 确认 `Build Settings → Debug Information Format = DWARF with dSYM File`（调试信息格式包含调试符号文件），否则崩溃无法还原（见第七章）。

**出正式包前必须关掉的开关**

| 开关 | 在哪 | 为什么关 |
|---|---|---|
| Development Build | Unity Build Profiles / Build Settings | 会带 Profiler 和调试符号，包体大、性能低 |
| Script Debugging | 同上，Development Build 的子项 | 脚本执行变慢，不能用来测性能 |
| Deep Profiling Support | 同上 | 每个函数调用都记录，极慢 |

**Archive 失败先查**

| 现象 | 原因 | 怎么办 |
|---|---|---|
| Archive 菜单是灰的 | 设备选择器没选 `Any iOS Device (arm64)` | 改选 `Any iOS Device (arm64)` |
| No signing certificate | 没证书/私钥，或 Team 没选 | 自动签名选 Team；CI 用 match 或导 `.p12`（8.6） |
| No profiles for 'com.xxx' were found | Bundle ID 没建 App ID，或 profile 没装 | 按 5.3 自动或手动签名 |
| Command CodeSign failed | 签名、entitlements、描述文件不匹配 | 见 8.5 |
| 磁盘空间不足 | Archive 和 dSYM 很占空间 | 清理 `~/Library/Developer/Xcode/Archives` 里的旧包 |

Archive 产物默认在 `~/Library/Developer/Xcode/Archives/`，里面的 `.xcarchive` 自带 dSYMs，别删（主文档 6.1 第 5 步）。

CI 里的等价命令：

```bash
# 1. 归档；有 CocoaPods 用 -workspace，没有则用 -project
xcodebuild -workspace MyGame.xcworkspace -scheme MyGame \
  -configuration Release -archivePath build/MyGame.xcarchive archive

# 2. 导出 ipa；ExportOptions.plist 是导出配置文件
xcodebuild -exportArchive -archivePath build/MyGame.xcarchive \
  -exportOptionsPlist ExportOptions.plist -exportPath build/ipa
```

### 5.6 第 5 步：分发测试

- **TestFlight**：苹果官方的灰度测试平台。上传后在 App Store Connect → TestFlight 建测试组，适合外部测试和灰度。
- **蒲公英等第三方平台**：上传 ipa 生成安装链接；Ad Hoc 包要求测试设备 UDID 在描述文件里，否则装不上。
- **渠道包**：发行商要接壳 SDK 时，走第六章的重签流程。

第一次上传前，先在 App Store Connect → My Apps → `+` 新建 App 记录，Bundle ID 要和包里的完全一致；否则上传后找不到构建版本。

---

## 六、游戏特有的签名场景

### 6.1 渠道包与壳 SDK 重签（发行商流程）

游戏发行常见模式：研发商出"裸包"，发行商接入自己的壳 SDK（账号 / 支付 / 统计）后重新出包。整体流程：

```
研发 ipa → 解包（unzip）→ 注入壳 SDK（framework + 桥接层 + 资源）
   → 改 Info.plist（渠道号、URL Scheme（应用间跳转协议）、权限文案）
   → 换描述文件与证书（embedded.mobileprovision 全量替换）
   → framework 逐个重签 → 主程序重签（codesign 自动更新 CodeResources）
   → 重新 zip 成 ipa → 传蒲公英 / 渠道
```

要点：

- **重签顺序**：先签最内层的 framework，最后签主程序——签名是"由内向外"嵌套的；
- `codesign`（苹果的签名命令行工具）会自动重写 `_CodeSignature/CodeResources`（资源签名清单），不用手工维护；
- 渠道号一般写在 Info.plist 自定义字段（如 `ChannelID`）或 Xcode 构建配置里，由脚本按渠道批量替换。

最小命令序列：

```bash
# 解包
unzip MyGame.ipa -d work
# 替换描述文件
cp new.mobileprovision work/Payload/MyGame.app/embedded.mobileprovision
# 从新描述文件导出 entitlements
security cms -D -i new.mobileprovision > profile.plist
plutil -extract Entitlements xml1 -o ent.plist profile.plist
# 由内向外：先签所有 framework（有 .dylib 也要逐个签；嵌套 framework 从最内层开始）
codesign -f -s "Apple Distribution: TeamName (TEAMID)" work/Payload/MyGame.app/Frameworks/*.framework
# 再签主程序
codesign -f -s "Apple Distribution: TeamName (TEAMID)" --entitlements ent.plist work/Payload/MyGame.app
# 重新压成 ipa
cd work
zip -qr ../MyGame-resigned.ipa Payload
```

`security cms` 用来解析描述文件，`plutil` 用来读写 plist，两者都是 macOS 自带命令。

### 6.2 多渠道工程化：scheme / xcconfig / target（在哪里配置）

一个工程出 N 个渠道包，别用"复制 N 份工程"的土办法，标准做法是用 Xcode 的构建配置体系：

1. **Configuration（构建配置）**：Xcode → 选中工程（蓝色图标）→ `Info → Configurations` → 点 + 复制 Release，创建 `Release-ChannelA`、`Release-ChannelB`；
2. **xcconfig（配置文件）**：每个渠道一个 `.xcconfig` 文件，把差异写成构建设置变量：

```text
// ChannelA.xcconfig
CHANNEL_ID = channel_a
PRODUCT_BUNDLE_IDENTIFIER = com.team.game.a
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) CHANNEL_A=1
```

3. **引用变量**：target → `Build Settings`（构建设置）里把 Bundle ID 等改成 `$(PRODUCT_BUNDLE_IDENTIFIER)`，Info.plist 里加自定义字段 `ChannelID = $(CHANNEL_ID)`，代码里用宏 `#if CHANNEL_A` 区分渠道逻辑；
4. **Scheme（方案）**：`Product → Scheme → Manage Schemes` → 每个 Configuration 建一个 scheme（如 `MyApp-ChannelA`），打包时选对应 scheme；
5. **Jenkins 接入**：xcodebuild 的 `-scheme MyApp-ChannelA` 换一下，就能循环出全渠道包。

> Unity 工程等效做法：在 `[PostProcessBuild]` 里按渠道宏改工程和 Info.plist，效果相同。

### 6.3 换 Team / 换主体（最容易翻车的场景）

游戏换发行商、公司换开发者账号时，**改的不只是证书**，以下五处全部联动：

| 联动项 | 说明 |
|---|---|
| 证书 + 描述文件 | 全部重新申请，`embedded.mobileprovision`（包内描述文件）的证书、App ID、设备列表、entitlements 四要素全换 |
| Bundle ID | 通常随 Team 变 → 代码里所有硬编码包名、URL Scheme 都要改 |
| Keychain 共享 | `keychain-access-groups` 的值以 Team ID 前缀开头（`TEAMID.com.team.app`）→ 换 Team 必改，否则旧设备上钥匙串（Keychain）里的数据迁移、登录态共享失效 |
| Universal Link | AASA（Apple App Site Association，苹果的应用关联文件）里的 Team ID 要同步更新（主文档 8.6）——否则深链接退化为网页跳转 |
| 推送 | APNs（Apple Push Notification service，苹果推送服务）证书 / 密钥（`.p8`）绑 Team → 服务端也要换，否则推送静默失效 |

> **经验法则：换 Team = 按上面的清单逐项打勾，漏一项就是一个线上事故。**

### 6.4 防二次打包与加固（游戏安全）

游戏是破解重灾区，签名知识直接支撑安全防护：

- **签名自校验**：运行时读取自身的 `embedded.mobileprovision`、校验 CodeDirectory（签名目录）哈希，发现不匹配就拒绝运行——防"破解组解包改资源后重签分发"；
- **反调试 Anti-Debug**：`ptrace(PT_DENY_ATTACH)` 阻止调试器附加（配合主文档 3.3 的 `get-task-allow=false`）；
- **符号混淆**：审核会扫描 Mach-O 符号表检测私有 API（Application Programming Interface，应用程序编程接口；主文档 8.5），加固同时混淆自有符号，增加逆向难度；
- **第三方加固**：爱加密、腾讯乐固等提供整体加固（防内存修改 / 防重签），原理绕不开上面三条。

### 6.5 内购与版号（国内上架的硬门槛）

**支付红线（Guideline 3.1.1）**：游戏内的虚拟货币、道具、会员订阅等数字内容，必须走苹果 IAP（In-App Purchase，内购），不得接支付宝/微信等第三方支付，也不得跳转外部网页购买——这是游戏被下架的高频原因。商品类型、创建位置、沙盒测试见主文档 9.7。

**版号（游戏出版备案）**：

- 2020 年 7 月起，中国区 App Store 上架的付费 / 含内购游戏必须提供版号；无版号无法在中国区提审，只能走海外区；
- 版号由出版社 / 发行商申请，流程远长于 App 备案，研发团队要提前和发行确认；
- 提审时在 App Store Connect 按要求提供版号信息，并在审核备注中说明。

**国内游戏上架中国区的合规三件套**：

1. **版号**：游戏本体合规；
2. **App 备案（ICP）**：主文档 9.8，绑 Bundle ID；
3. **App Privacy 隐私标签 + 权限文案**：与 SDK 收集行为一致（主文档 8.1 / 9.5）。

> 游戏通常全占：内购 + 备案 + 版号 + 隐私合规。排期时把这些前置周期（版号数月、备案数周）算进发版计划。另外，游戏接入的第三方 SDK 要各自带隐私清单（`PrivacyInfo.xcprivacy`），否则上传会收到 ITMS-91053 / 91056 / 91061（见 8.9）。

---

## 七、IL2CPP 与 dSYM 管理

**为什么游戏的 dSYM 特别重要**：Unity IL2CPP 把游戏逻辑编进 `UnityFramework`（老版本在主程序），线上崩溃堆栈全是内存地址；**没有对应 UUID（唯一标识）的 dSYM（调试符号文件，Debug Symbols File），堆栈就是一堆乱码**。

**管理规则**：

1. **每个构建唯一**：主程序、`UnityFramework`、第三方 framework 的 UUID 每次编译都不同——归档整个 `.xcarchive`（里面自带 dSYMs），按版本永久保存（主文档 6.1 第 5 步）；游戏包大，建议出包脚本自动把 dSYM 拷到符号服务器或网盘；
2. **上传崩溃平台**：
   - Firebase Crashlytics：`${PODS_ROOT}/FirebaseCrashlytics/upload-symbols -gsp GoogleService-Info.plist path/MyApp.app.dSYM`
   - Bugly：`java -jar buglySymboliOS.jar -i MyApp.app.dSYM`
3. **C# 层堆栈**：dSYM 只能还原 native（C++）层；C# 脚本层堆栈需要 Unity 的托管调试符号配合，线上排查看 native 堆栈 + IL2CPP 映射表；
4. **验证 UUID 匹配**：用 `dwarfdump --uuid MyApp.app.dSYM`（dwarfdump 是 Xcode 自带的 dSYM 查看工具）对比崩溃日志里的 UUID，对不上就是拿错了版本。

**手动还原一条堆栈**：

```bash
# 1. 从崩溃日志的 Binary Images 里找到 load address、UUID 和崩溃地址
#    Binary Images: 0x104a00000 - 0x1050fffff MyGame arm64 <UUID> ...
# 2. 用对应 UUID 的 dSYM 还原地址（-l 后面是 load address）
atos -o MyGame.app.dSYM/Contents/Resources/DWARF/MyGame \
  -arch arm64 -l 0x104a00000 0x104a12345
```

---

## 八、导出后的高频事故与性能排查

共同规律：**引擎导出后的报错，大多集中在"架构、签名、重复符号、符号文件"四件事上；能出包之后，下一关是包体、启动、内存和 GPU。**

### 8.1 模拟器架构混入（ITMS-90087）

**现象**：上传后苹果邮件：

```
ERROR ITMS-90087: "Unsupported Architectures. The executable for MyGame.app/Frameworks/XXX.framework
contains unsupported architectures '[x86_64, i386]'."
```

**原理（新手版）**：第三方 framework 为了同时支持模拟器和真机，打成了"胖包"（Fat Binary，多架构二进制），里面既有真机的 arm64，也有模拟器的 x86_64。App Store 只接受真机架构，模拟器 slice（架构切片）必须去掉。

**怎么办**：

1. 用 `lipo -info XXX.framework/XXX` 看包含哪些架构（lipo 是 Xcode 自带的架构处理工具）；
2. 只保留真机的 arm64：用 `lipo -thin arm64` 把 arm64 那一份抽出来；
3. Apple Silicon 上模拟器也是 arm64，不能只看架构名，用 `vtool -show-build XXX.framework/XXX` 看 platform 是不是 `IOSSIMULATOR`（vtool 是 Xcode 自带的构建信息查看工具）；
4. 推动 SDK 供应商改发 `.xcframework`（苹果推荐的多平台二进制格式），从源头解决。

第 2 步的命令（先输出到临时文件再覆盖，避免直接改同一个文件被截断）：

```bash
lipo -thin arm64 XXX.framework/XXX -output XXX.framework/XXX.arm64
mv XXX.framework/XXX.arm64 XXX.framework/XXX
```

**预防**：上架前跑一遍 Strip 脚本，遍历 `Frameworks/` 检查架构和 platform。

### 8.2 framework 签名方式错

**现象**：真机安装失败，提示"App 无法安装，因为无法验证其完整性"；或上传报 `Invalid Signature` / `ITMS-90035`。

**原理（新手版）**：动态库（`.framework`，运行时才加载）必须勾 `Embed & Sign`，跟着 App 一起装进包并签名；静态库（`.a` 或静态 framework）在链接时已经拷进主程序，只能 `Do Not Embed`。选错就会多一份没签名的二进制，或者把静态库当动态库嵌入。

**怎么办**：

1. Xcode → target → `General → Frameworks, Libraries, and Embedded Content`（通用 → 框架、库和嵌入式内容）；
2. 动态库选 `Embed & Sign`，静态库选 `Do Not Embed`；
3. 用 `otool -l XXX.framework/XXX | grep -A2 LC_ID_DYLIB` 判断是不是动态库（otool 是 Xcode 自带的二进制查看工具）；
4. 重签后再校验：`codesign -v --strict XXX.framework`。

**预防**：接 SDK 时先问清是静态库还是动态库，写进接入文档。

### 8.3 符号冲突（duplicate symbol）

**现象**：链接阶段报：

```
duplicate symbol '_OBJC_CLASS_$_XXX' in:
    libA.a(XXX.o)
    libB.a(XXX.o)
```

**原理（新手版）**：两个静态库打包了同一份底层库（游戏 SDK 尤其常见，比如两个广告 SDK 都带了同一版网络库）。链接器不知道用哪一份，直接报错。

**怎么办**（详细步骤见主文档 7.1）：

1. 用 `nm -g libA.a | grep "OBJC_CLASS_\$_XXX"`（nm 是查看符号表的命令）确认冲突符号在哪些库里；
2. 首选：让供应商提供"不含底层库"的版本，全项目共用一份；
3. 拿得到源码：给其中一个版本的类加前缀重新编译；
4. 兜底：把冲突的公共库抽成一个动态 framework，两个 SDK 都改为外链它。

**预防**：接 SDK 前统一依赖版本，出包脚本加一步 `duplicate symbol` 扫描。

### 8.4 dSYM 缺失 / 对不上

**现象**：Crashlytics / Bugly 上的堆栈全是 `0x0000000104a...`，看不到函数名。

**原理（新手版）**：dSYM 是"地址 → 代码行"的映射表，每次编译都会生成新的 UUID。上传时如果拿的是旧版本的 dSYM，地址对不上，堆栈就还原不出来。

**怎么办**：

1. `dwarfdump --uuid MyApp.app.dSYM` 看 dSYM 的 UUID；
2. 和崩溃日志 `Binary Images` 里的 UUID 对比，必须完全一致；
3. 用对应版本的 dSYM 重新上传（命令见第七章）；
4. 检查 Xcode 的 `Debug Information Format` 是否为 `DWARF with dSYM File`。

**预防**：CI 每次 Archive 自动归档 dSYM，文件名带上版本号和构建号。

### 8.5 重签后启动闪退

**现象**：安装成功，点图标立刻闪退；Xcode → `Window → Devices and Simulators → View Device Logs`（窗口 → 设备与模拟器 → 查看设备日志）里能看到：

```
dyld: Library not loaded: @rpath/XXX.framework/XXX
Referenced from: /var/containers/.../MyGame.app/MyGame
Reason: no suitable image found. Did find: ... code signature invalid
```

`@rpath` 是运行时库的搜索路径。这段日志的意思是：系统按路径找到了 framework，但签名校验没通过，所以拒绝加载。

**原理（新手版）**：改了包里的任何二进制（framework、主程序、资源）都必须重新签名；只重签主程序、漏掉 framework，或者签名顺序反了，系统加载时就会拒绝。

**怎么办**：

1. 按"由内向外"重签：先所有 framework，再主程序；
2. 检查每个 framework 的签名：`codesign -dv --verbose=4 XXX.framework`；
3. 检查描述文件：`security cms -D -i embedded.mobileprovision`，确认 Team、Bundle ID、设备列表、entitlements 都对；
4. 清掉旧包重装，避免系统缓存。

**预防**：用脚本重签，签完跑一遍 `codesign -v --strict` 全量校验。

### 8.6 CI 签名失败（本地却正常）

**现象**：本地 Archive 正常，Jenkins（持续集成工具）或 GitHub Actions 报：

```
error: No signing certificate "iOS Distribution" found
```

**原理（新手版）**：签名需要"证书 + 私钥"成对出现。CI 机器是裸机，只导入了 `.cer` 证书没有私钥，或者没装描述文件，自然签不了。

**怎么办**：

1. 用 fastlane match（证书与描述文件统一管理工具）统一管理证书和描述文件（主文档 6.2）；
2. 或手动把 `.p12`（含私钥）和 `.mobileprovision` 导入 CI 的钥匙串；
3. 加上 `-allowProvisioningUpdates`，让 xcodebuild 自动拉取描述文件；
4. 确认 CI 的 Xcode 版本和本地一致。

**预防**：证书集中管理，CI 固定 Xcode 版本，出包前跑一次签名自检。

### 8.7 Xcode / SDK 版本过低（ITMS-90725）

**现象**：上传成功，随后收到邮件（大意如下，关键词是错误码 ITMS-90725）：

```
ITMS-90725: SDK version issue - This app was built with the iOS 18 SDK.
Starting April 28, 2026, apps must be built with the iOS 26 SDK or later.
```

**原理（新手版）**：苹果会定期提高最低 SDK 门槛。2026 年 4 月 28 日起，上传 App Store Connect 必须用 iOS 26 SDK（Xcode 26+）构建；旧 Xcode 打的包会被直接拒收。

**怎么办**：

1. 升级到 Xcode 26 或更高（系统要求见对应 Xcode 发布说明；Xcode 26 需要 macOS Sequoia 15.6+）；
2. 引擎侧确认支持该 Xcode 版本：Unity、Unreal、Cocos Creator、Godot 都要升到官方支持 Xcode 26 的版本；
3. 重新 Archive、重新上传，不要试图改包绕过。

**预防**：把"Xcode 版本"写进发版检查清单，并关注苹果每年的 SDK 门槛公告。

### 8.8 大内存被系统杀掉（Jetsam）

**现象**：低端机启动或切场景时闪退，Xcode 日志里出现 `JetsamEvent`，没有崩溃堆栈。

**原理（新手版）**：iOS 没有交换分区，内存不够时系统会直接杀掉占用最大的进程。游戏的高清纹理、长音频、大场景加载是内存大户，峰值一超就被杀。

**怎么办**：

1. 用 Instruments（Xcode 自带的性能分析工具）的 Allocations / VM Tracker 看内存峰值；
2. 压缩纹理（ASTC / PVRTC 是两种常用的纹理压缩格式）、按场景卸载、音频流式加载；
3. 在 `didReceiveMemoryWarning` 里释放缓存；
4. 低端机降画质、降分辨率。

**预防**：把内存峰值纳入每个版本的性能测试，按机型分档。

### 8.9 隐私清单相关（ITMS-91053 / 91056 / 91061）

**现象**：上传后收到邮件，提示缺少 API 声明、隐私清单格式错误，或某个第三方 SDK 没有隐私清单。

**原理（新手版）**：苹果要求 App 和第三方 SDK 用 `PrivacyInfo.xcprivacy` 声明用了哪些"必填理由 API"、收集哪些数据。游戏接的 SDK 多，任何一个缺失都会触发邮件。

**怎么办**：

1. 按邮件里的路径找到对应 SDK，升级到带隐私清单的版本；
2. 自己工程补 `PrivacyInfo.xcprivacy`（主文档 8.7）；
3. 重新 Archive 上传。

**预防**：接入 SDK 时把"是否带隐私清单"写进验收项。

### 8.10 编译 / 链接 / 签名报错关键词索引

先在 Xcode 左侧的 Report navigator（报告导航器）看**第一条红色错误**，不要从最后一条看；链接错误往往只有第一条有用。

| 报错关键词 | 大概率原因 | 先看 |
|---|---|---|
| `No such module 'XXX'` | framework/module 没链接，或 Framework Search Paths 不对 | 8.2、4.2 |
| `framework not found XXX` | framework 路径不对、没加到工程 | 8.2 |
| `Undefined symbol: _OBJC_CLASS_$_XXX` | 静态库没链接，或缺 `-ObjC` | 8.3 |
| `duplicate symbol` | 两个库带同一份符号 | 8.3 |
| `Building for iOS, but the linked framework 'XXX.framework' was built for iOS Simulator` | 链进了模拟器 slice | 8.1 |
| `missing one or more architectures required by this target: arm64` | 静态库没有 arm64 | 8.1 |
| `No signing certificate "iOS Distribution" found` | 本机/CI 没证书私钥 | 8.6 |
| `Provisioning profile ... doesn't include signing certificate` | profile 和证书不匹配 | 5.3 |
| `The app ID ... cannot be registered to your development team` | Bundle ID 被占用，或 Team 不对 | 5.3 |
| `Command CodeSign failed with a nonzero exit code` | 签名、entitlements、profile 不匹配 | 8.5 |
| `UnityFramework.framework/UnityFramework` not found | UnityFramework 没编出来，或 Embed 设置错 | 8.2 |
| `ITMS-...` | 上传校验 | 9.3 |

### 8.11 崩溃日志怎么读（5 步）

**第 1 步：拿到日志**

Xcode → `Window → Devices and Simulators` → 选设备 → `View Device Logs` → 找最新的一条（按时间/进程名）。

**第 2 步：找 UUID 和 load address**

在日志底部 `Binary Images` 里找你的 App 或 `UnityFramework`：

```
0x104a00000 - 0x1050fffff MyGame arm64 <UUID> /var/containers/.../MyGame
0x106000000 - 0x107ffffff UnityFramework arm64 <UUID> ...
```

第一个地址是 load address，尖括号里是 UUID。

**第 3 步：找崩溃地址**

看崩溃线程（Crashed Thread）的调用栈，记下最上面几行的地址，例如 `0x104a12345`。

**第 4 步：确认 dSYM**

```
dwarfdump --uuid MyGame.app.dSYM
```

UUID 必须和日志里的一致；dSYM 在 `.xcarchive/dSYMs/` 里。

**第 5 步：还原**

```
atos -o MyGame.app.dSYM/Contents/Resources/DWARF/MyGame \
  -arch arm64 -l 0x104a00000 0x104a12345
```

`-l` 后面是第 2 步的 load address。崩在 `UnityFramework` 就用 `UnityFramework` 的 dSYM 和 load address。命令参数以 macOS 自带的 `man atos` 为准。

### 8.12 IL2CPP 裁剪导致真机崩溃（link.xml / [Preserve]）

**现象**：编辑器、模拟器都正常，真机 Release 包一启动或走到某个功能就崩；日志里常见 `MissingMethodException`、`TypeLoadException`、`NullReferenceException`，或者 SDK 回调完全不触发。

**原理（新手版）**：Unity 的托管代码裁剪（Managed Stripping）会把"看起来没人用"的代码删掉。反射、JSON 序列化、依赖注入、SDK 回调这些用法，静态分析看不到，真机包里类就被裁没了。

**怎么办**：

1. 先把 `Player Settings → Other Settings → Optimization → Managed Stripping Level` 调低（如 Disabled / Minimal）验证是不是裁剪问题；
2. 在 `Assets/link.xml` 里保留被反射用到的程序集/类：

```xml
<linker>
  <assembly fullname="Assembly-CSharp">
    <type fullname="MyGame.SdkCallback" preserve="all"/>
  </assembly>
</linker>
```

3. 或在类/方法/字段上加 `[Preserve]`（`UnityEngine.Scripting.PreserveAttribute`）：

```csharp
using UnityEngine;
using UnityEngine.Scripting;

[Preserve]
public class SdkCallback : MonoBehaviour {
    // SDK 通过反射/消息调用的方法
    public void OnLoginResult(string json) { /* ... */ }
}
```

4. 看第三方 SDK 文档，很多 SDK 自带 link.xml，接入时要一起拷进 `Assets/`；
5. 用 Release 配置重新出真机包验证，不要只看编辑器。

**预防**：反射重的模块统一维护一份 link.xml；发版前用 Release 真机包跑一遍核心流程。

### 8.13 包体、启动、内存、GPU 专项排查

游戏卡顿、闪退、包体大，先分清是哪一类问题，不要一上来就降画质。

| 问题 | 先看什么 | 常用手段 |
|---|---|---|
| 包体大 | 纹理、音频、重复 SDK、Resources 目录 | ASTC/PVRTC 压缩纹理；音频压缩/流式；Addressables/AssetBundles；删无用插件；Managed Stripping；Strip Engine Code |
| 启动慢 | 首场景、Awake、Resources 加载、IL2CPP 初始化 | 精简首场景；避免 `Resources.LoadAll`；Addressables 异步加载；Instruments 的 App Launch 看耗时 |
| 内存高 | 纹理、音频、泄漏、缓存 | Instruments Allocations/VM Tracker；Memory Profiler；`Resources.UnloadUnusedAssets`；`Addressables.Release`；`didReceiveMemoryWarning` |
| GPU 卡 | overdraw、draw call、shader 变体、分辨率 | Xcode → Product → Scheme → Edit Scheme → Run → Diagnostics → 勾 GPU Frame Capture，运行时点 Metal Capture；Unity Frame Debugger；降分辨率/overdraw/shader 变体 |
| CPU 卡 | GC、逻辑、物理、UI | Unity Profiler；Unreal Insights；减少每帧分配；对象池 |

**三分钟定位顺序**：

```
卡顿 → 先看 CPU 还是 GPU：
  CPU 高 → Profiler 看 GC、逻辑、物理
  GPU 高 → Metal Capture 看 overdraw、draw call、shader
闪退 → 先看 Jetsam（内存）还是 crash（签名/代码）
包体大 → 先看纹理、音频、重复 SDK
```

**Unity 专属提醒**：Development Build / Script Debugging 会让性能明显变慢，测性能必须用 Release 包；Debug 包的数据不能拿来和竞品比。

---

## 九、高频速查表

### 9.1 引擎入口与产物

| 引擎 | 导出/打包入口 | 产物 | 备注 |
|---|---|---|---|
| Unity 6 | File → Build Profiles → iOS → Build | Xcode 工程 | Build and Run 会继续调 Xcode |
| Unity 2022- | File → Build Settings → iOS → Build | Xcode 工程 | 旧入口 |
| Unreal | Platforms → iOS → Package Project | ipa / Xcode 工程 | 调试用 Prepare for Debugging |
| Cocos Creator | 构建发布 → iOS → 构建 | `build/ios/proj/*.xcodeproj` | 结构接近原生 |
| Godot | 项目 → 导出 → iOS → 导出项目 | Xcode 工程 | C# 导出仍为实验性 |

### 9.2 常用命令

| 目的 | 命令 |
|---|---|
| 看 framework 架构 | `lipo -info XXX.framework/XXX` |
| 看 slice 的平台 | `vtool -show-build XXX.framework/XXX` |
| 判断动态库 | `otool -l XXX.framework/XXX`，搜 `LC_ID_DYLIB` |
| 校验签名 | `codesign -v --strict XXX.framework` |
| 看签名详情 | `codesign -dv --verbose=4 XXX.app` |
| 看描述文件 | `security cms -D -i embedded.mobileprovision` |
| 看 dSYM UUID | `dwarfdump --uuid MyApp.app.dSYM` |
| 还原堆栈 | `atos -o MyApp.app.dSYM/Contents/Resources/DWARF/MyGame -arch arm64 -l 0x装载地址 0x崩溃地址` |
| 装命令行工具 | `xcode-select --install` |
| 选中完整 Xcode | `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` |
| 看 Xcode 版本 | `xcodebuild -version` |
| 看 macOS 版本 | `sw_vers` |
| 看 CocoaPods 版本 | `pod --version` |
| 看静态库符号 | `nm -g libA.a \| grep "OBJC_CLASS"` |
| 归档 | `xcodebuild -workspace MyGame.xcworkspace -scheme MyGame -configuration Release -archivePath build/MyGame.xcarchive archive` |
| 导出 ipa | `xcodebuild -exportArchive -archivePath build/MyGame.xcarchive -exportOptionsPlist ExportOptions.plist -exportPath build/ipa` |

### 9.3 上传错误码

| 错误码 | 含义 | 先查什么 |
|---|---|---|
| ITMS-90087 | 包含不支持的架构（模拟器 slice） | 8.1 |
| ITMS-90035 | 签名无效 / 某文件没签名 | 8.2 / 8.5 |
| ITMS-90426 | SwiftSupport（Swift 运行时支持目录）缺失 | 是否用非 Mac 工具改过包、Swift 库是否完整 |
| ITMS-90725 | SDK 版本过低 | 8.7，升级 Xcode 26+ |
| ITMS-91053 | 缺少必填理由 API 声明 | 8.9，补隐私清单 |
| ITMS-91056 | 隐私清单格式无效 | 8.9 |
| ITMS-91061 | 第三方 SDK 缺隐私清单 | 8.9，升级 SDK |

> 错误码含义来自 Apple 上传邮件和开发者论坛案例，具体以邮件原文和 App Store Connect 提示为准。

### 9.4 游戏发版前检查清单（收藏）

- [ ] Xcode 首次启动、iOS 模拟器、命令行工具已就绪
- [ ] Unity iOS Build Support / Godot 导出模板等引擎依赖已安装
- [ ] Bundle Identifier、版本号、构建号已确认
- [ ] Signing Team、描述文件、Capabilities 与 App ID 一致；Ad Hoc profile 含测试设备
- [ ] 第三方 framework 的架构、签名方式（Embed & Sign / Do Not Embed）已检查
- [ ] Release 包已关闭 Development Build / Script Debugging
- [ ] `link.xml` / `[Preserve]` 已覆盖反射代码
- [ ] 本版本的 dSYM 已随 `.xcarchive` 归档，UUID 和崩溃日志能对上
- [ ] 所有第三方 SDK 都带 `PrivacyInfo.xcprivacy`
- [ ] 内购商品已创建，沙盒测试通过
- [ ] 版号、App 备案已就绪（国内上架）
- [ ] 用 Xcode 26+、iOS 26 SDK 构建
- [ ] 包体、启动耗时、内存峰值、低端机帧率达标
- [ ] 真机或 TestFlight 跑过启动、登录、支付、弱网、低端机

---

## 十、业务视角：研发公司、发行公司与知识映射

### 10.1 产业链：谁把游戏送到玩家手机里

```
玩家
 ↑ 付费 / 看广告
App Store（iOS 公开分发的主渠道；欧盟另有替代市场）
 ↑ 上架、买量、运营
发行商 Publisher：市场、买量、渠道、合规、运营、客服
 ↑ 代理 / 独代 / 联运 / 分成
研发商 CP（Content Provider，内容提供商）：策划、美术、客户端、服务端、引擎、QA
 ↑ 外包
美术外包 / QA 外包 / SDK 与工具链供应商
```

一句话：**研发负责"把游戏做出来、跑得稳"，发行负责"把游戏送上架、卖出去、运营好"。**

先记住几个业务词：

| 业务词 | 英文/全称 | 大白话 |
|---|---|---|
| 研发商 | CP（Content Provider） | 做游戏的公司 |
| 发行商 | Publisher | 负责上架、买量、运营的公司 |
| 独代 / 联运 | Exclusive Agent / Joint Operation | 一个区域只给一家卖 / 研发和渠道一起运营、分成 |
| 买量 / 归因 | User Acquisition / Attribution | 花钱投广告买用户 / 判断用户是哪条广告带来的 |
| 内购 / 广告变现 | IAP / IAA | 玩家付钱 / 玩家看广告付钱 |
| 苹果搜索广告 / 追踪授权 | ASA / ATT | 在 App Store 投广告 / 问用户能不能被追踪 |

### 10.2 研发公司：对"能不能跑、跑得好不好"负责

研发公司的核心交付是**可上线的版本**：

- 玩法、数值、关卡、美术、音频；
- 客户端（引擎、渲染、UI、SDK 接入）、服务端、工具链；
- 性能、包体、内存、崩溃率、机型适配；
- 按发行节奏出包、修 Bug、跟版本。

研发最关心的指标：帧率、崩溃率、包体大小、加载时长、留存（和发行共同看）。对应本文第一、二、三、四、五、七、八章。

### 10.3 发行公司：对"能不能上、卖给谁、赚不赚钱"负责

发行公司的核心交付是**合规上线 + 用户增长 + 商业化**：

- 上架：App Store 提审、元数据、截图/视频、本地化；
- 合规：版号、App 备案、隐私政策、IAP 合规；
- 渠道与买量：ASA（Apple Search Ads，苹果搜索广告）、信息流、达人、品牌；
- SDK：账号、支付、统计、归因、推送、客服系统；
- 运营：活动、版本节奏、用户分层、客服与社区；
- 商业化：IAP（内购）、IAA（In-App Advertising，广告变现）、订阅。

发行关心三件事：用户从哪来（买量/归因）、留不留得住（留存）、能不能回本（ROI，投入产出比）。买量归因受 ATT（App Tracking Transparency，应用追踪透明度）授权影响，常用 SKAdNetwork / AdAttributionKit 等机制；研发侧要配合接归因 SDK、处理授权弹窗。

对应本文第三、五、六、八、九章，尤其是签名、渠道包、换 Team、内购与版号。

> **iOS 的"渠道"和安卓不一样**：iOS 公开分发以 App Store 为主，没有安卓那种多应用商店并行的渠道包。发行侧说的"渠道"更多是买量渠道和归因渠道；"重签/渠道包"主要用于测试分发、企业内部分发、海外区域或特殊合作，不能照搬安卓思路。

### 10.4 合作模式（点到为止）

常见合作方式有代理发行、独代、联运、自研自发。具体条款（代理金、保底、分成）由合同决定，不影响你怎么出包，但决定了**谁持有 Apple 开发者账号、谁负责提审、出了事故谁先处理**。

### 10.5 一次合作从签约到上线：谁在哪个环节做什么

下表最后一列是本文的章节号，方便对照技术细节。

| 阶段 | 研发 | 发行 | 对应本文章节 |
|---|---|---|---|
| 签约 | 提供 Demo / 技术方案 | 评估产品、谈代理金/分成 | — |
| 里程碑 | 可玩版本、性能/包体达标 | 反馈市场/本地化需求 | 1、4 |
| 接壳 SDK | 接账号/支付/统计/归因/推送 | 提供 SDK、验收回调 | 4.2、6.1 |
| 合规 | 配 IAP 商品、隐私清单 | 版号、备案、隐私政策、提审 | 6.5、8.9 |
| 出包 | Xcode 工程、签名、Archive | 提供 Team/描述文件/分发账号 | 5、6 |
| 提审 | 修审核问题 | 填元数据、提交审核 | 5.6、9.3 |
| 上线买量 | 保版本稳定、看崩溃 | 投放、归因、ROI | 7、8 |
| 运营迭代 | 活动版本、热更 | 活动策划、用户运营 | 2.4 |

### 10.6 知识 ↔ 业务对照表

| 本文知识 | 研发公司用它做什么 | 发行公司用它做什么 |
|---|---|---|
| 编译链路 / IL2CPP（1.3、4.2） | 控包体、查性能、定位 C++ 崩溃 | 评估版本节奏、技术风险 |
| 引擎与语言（2、4） | 选型、招人、跨平台 | 评估包体、SDK 兼容、上线周期 |
| 导出 Xcode 工程（3、5） | 出包、接 SDK、Archive | 接收包、验收、分发 |
| 签名 / 描述文件（5.3、6.3） | 真机调试、上架包 | 换 Team、换主体、渠道分发 |
| PostProcessBuild / 桥接（4.2） | 代码化接 SDK | 壳 SDK 验收、减少手工 |
| 多渠道工程化（6.2） | 配合渠道改配置 | 批量出包、归因/渠道管理 |
| dSYM / 崩溃（7） | 崩溃治理、还原堆栈 | 监控崩溃率、保留存/ROI |
| IAP / 版号 / 备案（6.5） | 商品配置、技术配合 | 合规、提审、上线节奏 |
| 隐私清单（8.9） | SDK 合规、补清单 | 上传前检查、避免拒审 |
| Xcode/SDK 版本（3、8.7） | 升级引擎、重新出包 | 排发版计划、避免 ITMS-90725 |
| 沙盒 / 内存（1.4、8.8） | 性能优化、低端机适配 | 机型覆盖、留存 |
| 包体 / 启动 / 内存 / GPU（8.13） | 优化下载转化、低端机留存 | 关注买量成本、留存、ROI |
| 热更新合规（2.4） | 技术方案、热更 | 审核风险、版本策略 |

### 10.7 常见协作事故：责任边界在哪

| 事故 | 通常发生在谁手里 | 怎么处理 |
|---|---|---|
| 发行壳 SDK 接入后启动崩溃 | 发行接入 / 研发配合 | 查架构、签名、符号冲突（8.1–8.3） |
| 换发行商/换 Team 后推送、深链、登录失效 | 发行交接 | 按 6.3 清单逐项换 |
| 渠道/测试分发包安装失败 | 发行 | 查描述文件、UDID、重签顺序（6.1、8.5） |
| 版号/备案没下来，无法提审 | 发行 | 提前排期（6.5） |
| 隐私清单缺失，上传被拒 | 发行/研发 SDK 验收 | 8.9 |
| dSYM 没交付，崩溃无法定位 | 研发 | 7，Archive 自动归档 |
| Xcode 版本过低，ITMS-90725 | 研发/发行发版 | 8.7，升级 Xcode 26+ |

> 这不是"甩锅表"，而是交接清单：每一条都对应一个可验收的产物（包、签名、dSYM、隐私清单、备案号）。

### 10.8 岗位与面试视角

| 岗位 | 主要职责 | 本文重点 |
|---|---|---|
| 游戏客户端 / 引擎 | 引擎、渲染、性能、SDK 接入 | 1–4、7、8 |
| 发行技术 / 中台 | 出包流水线、签名、渠道包、SDK 验收 | 3、5、6、8、9 |
| 平台 / 渠道对接 | 渠道规则、提审、合规 | 6.5、9.3 |
| 工具链 / CI | 自动化出包、dSYM 归档、签名管理 | 3、5.5、7、8.6 |

**面试口述钩子（游戏方向）**：

- **Unity 接 SDK 的 STAR**（Situation-Task-Action-Result，情境-任务-行动-结果）：S 现象（SDK 要求加 framework + 权限文案）→ T 任务（每次出包手工改工程又慢又漏）→ A 行动（`[PostProcessBuild]` 代码化 + DllImport/UnitySendMessage 双向桥接）→ R 结果（出包零手工介入，接新 SDK 从半天变 10 分钟）；
- **渠道重签案例**：讲清"由内向外重签 + CodeResources 自动更新 + 蒲公英分发"整条链，体现对签名体系的理解；
- **崩溃治理案例**：`UnityFramework`（`GameAssembly`）的 UUID 匹配 dSYM → upload-symbols 上传 → 堆栈还原到具体 C++ 函数，体现 IL2CPP 原理掌握；
- **研发/发行协作案例**：发行接壳 SDK 后启动崩溃 → 按 8.1–8.3 查架构、签名、符号冲突 → 研发提供干净包 + dSYM → 发行重新出包上线，体现"技术问题能翻译成业务语言"。

**面试时怎么讲业务**：

- 研发岗：强调"稳定出包 + 崩溃可定位 + SDK 可扩展"；
- 发行岗：强调"合规上架 + 渠道可追踪 + 版本可交付"；
- 中台/工具链岗：强调"自动化 + 标准化 + 可审计"。

**一句话总结**：研发对"能不能跑、跑得好不好"负责，发行对"能不能上、卖给谁、赚不赚钱"负责；出包、签名、合规就是两者的交接面。从技术面试角度再看一遍：主文档的签名体系是"通用规则"，游戏出包是"同一套规则 + 引擎桥接 + 渠道分发"的组合应用。

---

## 附录：实操入口地图（去哪里动手）

| 要做的事 | 位置（界面/命令） | 操作说明 |
|---|---|---|
| 改 Unity iOS 的 Bundle ID | Unity：Edit → Project Settings → Player → iOS | 出包前确认和描述文件一致 |
| 导出 Unity Xcode 工程 | Unity 6：File → Build Profiles → iOS → Build | 选空目录；Build and Run 会继续调 Xcode |
| 导出 Unreal iOS 包 | Unreal：Platforms → iOS → Package Project | 先配好签名；调试用 Prepare for Debugging |
| 导出 Cocos iOS 工程 | Cocos Creator：构建发布 → iOS → 构建 | 产物在 `build/ios/proj/` |
| 导出 Godot iOS 工程 | Godot：项目 → 导出 → iOS → 导出项目 | 先装导出模板 |
| 配自动签名 | Xcode：target → Signing & Capabilities | 勾 Automatically manage signing，选 Team |
| 加内购/推送能力 | Xcode：target → Signing & Capabilities → + Capability | 必须和 App ID 能力一致 |
| 真机运行 | Xcode 顶部设备选择器 → Run（Cmd+R） | 首次要信任开发者证书 |
| 出 Release 包 | Xcode：Any iOS Device (arm64) → Product → Archive | Archive 完在 Organizer 分发 |
| 上传 App Store Connect | Organizer → Distribute App → App Store Connect | 或 TestFlight |
| 看真机崩溃日志 | Xcode：Window → Devices and Simulators → View Device Logs | 搜 `JetsamEvent` 看内存杀 |
| 看 dSYM UUID | 终端：`dwarfdump --uuid MyApp.app.dSYM` | 和崩溃日志 UUID 对比 |

**排障"动手顺序"速查**：

```
① 先看报错发生在哪一步：编译 / 链接 / 签名 / 上传 / 启动
② 编译链接错 → 查架构、重复符号、framework 引入方式
③ 签名错 → 查证书私钥、描述文件、entitlements、重签顺序
④ 上传错 → 查 ITMS 错误码（第九章）
⑤ 启动闪退 → 查 dyld 日志、签名完整性、内存峰值
```

---

## 官方与参考链接（信息核验于 2026-11-21）

> 政策、版本、菜单路径会变，使用前以官方页面为准。

**Apple**

- App Review Guidelines（2.5.2 自包含 / 3.1.1 内购）：https://developer.apple.com/app-store/review/guidelines/
- Upcoming SDK minimum requirements（2026-02-03 公告，2026-04-28 生效）：https://developer.apple.com/news/?id=ehcqwnyg
- Xcode 26 Release Notes（要求 macOS Sequoia 15.6+）：https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes
- Apple Developer Program 注册与年费：https://developer.apple.com/cn/support/enrollment/
- 注册 App ID：https://developer.apple.com/help/account/identifiers/register-an-app-id
- 证书概览（Apple Development / Apple Distribution）：https://developer.apple.com/help/account/certificates/certificates-overview
- 创建 CSR（证书签名请求）：https://developer.apple.com/help/account/certificates/create-a-certificate-signing-request
- 创建开发描述文件：https://developer.apple.com/help/account/provisioning-profiles/create-a-development-provisioning-profile
- 注册测试设备：https://developer.apple.com/help/account/devices/register-multiple-devices
- App Store Connect 新建 App：https://developer.apple.com/help/app-store-connect/create-an-app-record/add-a-new-app
- 中国大陆上架要求（版号 + ICP 备案，App Store Connect 帮助）：https://developer.apple.com/cn/help/app-store-connect/reference/app-information/app-information
- 游戏版号说明（Apple 官方论坛）：https://developer.apple.com/forums/thread/657959
- 隐私清单排错（TN3181）：https://developer.apple.com/documentation/technotes/tn3181-debugging-invalid-privacy-manifest
- App Tracking Transparency（ATT）：https://developer.apple.com/documentation/apptrackingtransparency
- App 广告归因（AdAttributionKit / SKAdNetwork）：https://developer.apple.com/app-store/ad-attribution/
- Apple Search Ads（ASA）：https://searchads.apple.com/

**Unity**

- Build an iOS application：https://docs.unity.com/en-us/engine/6000.6/manual/platform-specific/iphone/ios-building-and-delivering/build-process
- How Unity builds iOS applications：https://docs.unity.com/en-us/engine/6000.6/manual/platform-specific/iphone/ios-introducing/how-unity-builds-ios-applications
- iOS build settings reference：https://docs.unity.com/en-us/engine/6000.0/manual/platform-specific/iphone/ios-building-and-delivering/build-settings-ios
- Scripting back ends（IL2CPP 为 iOS 必需）：https://docs.unity.com/en-us/engine/6000.6/manual/programming-environment/scripting-backends/intro
- Structure of a Unity Xcode project：https://docs.unity.com/en-us/engine/6000.6/manual/platform-specific/iphone/ios-developing/structure-of-xcode-project
- iOS requirements and compatibility：https://docs.unity.com/en-us/engine/6000.0/manual/platform-specific/iphone/ios-introducing/ios-requirements-and-compatibility
- Available Xcode versions（上架需 Xcode 26+）：https://docs.unity.com/en-us/build-automation/reference/available-xcode-versions
- Unity Learn: Publishing for iOS（Unity Hub 装 iOS 模块）：https://learn.unity.com/tutorial/publishing-for-ios

**Unreal / Cocos / Godot**

- Unreal iOS、iPadOS、tvOS 开发要求：https://dev.epicgames.com/documentation/unreal-engine/ios-ipados-and-tvos-development-requirements-for-unreal-engine
- Unreal iOS 快速入门：https://dev.epicgames.com/documentation/unreal-engine/ios-ipados-tvos-quick-start-guide-for-unreal-engine
- Cocos Creator iOS 构建示例：https://docs.cocos.com/creator/3.8/manual/zh/editor/publish/ios/build-example-ios.html
- Cocos Creator 原生开发环境：https://docs.cocos.com/creator/manual/en/editor/publish/setup-native-development.html
- Godot 为 iOS 导出：https://docs.godotengine.org/en/stable/tutorials/export/exporting_for_ios.html
- Godot C# 平台支持：https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/

**Xcode / 性能**

- Xcode 手动签名（Apple Help）：https://help.apple.com/xcode/mac/current/en.lproj/dev1bf96f17e.html
- Xcode 下载模拟器运行时（Apple Help）：https://help.apple.com/xcode/mac/current/en.lproj/deva7379ae35.html
- Xcode 捕获 Metal 工作负载：https://developer.apple.com/documentation/xcode/capturing-a-metal-workload-in-xcode
- Unity Managed code stripping：https://docs.unity.com/en-us/engine/6000.0/manual/programming-environment/script-compilation/managed-code-stripping

**工具链**

- fastlane（match / gym / pilot）：https://docs.fastlane.tools/
- Jenkins：https://www.jenkins.io/doc/
- 蒲公英：https://www.pgyer.com/
- 主文档的官方来源清单见《iOS 签名与出包知识体系》文末「官方与参考链接」。

> 可靠性说明：版本、政策、菜单路径以 Apple / Unity / Epic / Cocos / Godot 官方文档为准；ITMS 错误码来自上传邮件和 Apple Developer Forums 案例，具体以邮件原文为准；业务术语是行业通用说法，各公司叫法可能不同。
