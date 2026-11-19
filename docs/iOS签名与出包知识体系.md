# iOS 签名与出包知识体系（新手友好版）

> 覆盖：Mach-O、证书（Certificate）、代码签名（Code Signing）、描述文件（Provisioning Profile）、ipa 包（iOS App Store Package）、出包实战三条路（Xcode 手动 / Jenkins / 蒲公英）、发版流水线事故排查、上传前高频清单（含隐私清单）、从零到上架全景 SOP（账号 / 建档 / 元数据 / 内购 / 备案 / 审核 / 发布）、业务与面试视角。文末附实操入口地图（Xcode / Apple Developer 网站操作位置）。
> 阅读方式：每个概念先讲"是什么（带比喻）"，再讲"为什么需要它"，最后讲"怎么用/怎么排查"。关键词均标注中英文。
> 信息核验日期：2026-11-19（关键事实来源见文末「官方与参考链接」）。
> 姊妹篇：《iOS 游戏出包实战（引擎导出 Xcode 工程）》——Unity/Unreal/Cocos 等游戏相关内容已拆分至该文档。

---

## 目录

- 0. 先用一组比喻建立全局
- 一、Mach-O —— 程序的二进制本体
- 二、证书 —— 签名者身份
- 三、代码签名 —— 防篡改封条
- 四、描述文件 —— 设备/App/证书的绑定
- 五、ipa 包 —— 最终分发产物
- 六、出包实战三条路（Xcode 手动 / Jenkins / 蒲公英）
- 七、发版流水线事故排查（5 类事故源）
- 八、上传前打包高频清单（Info.plist / TestFlight / ITMS 错误码 / 隐私清单）
- 九、从零到上架全景 SOP（账号 → 建档 → 元数据 → 提审 → 发布）
- 十、业务视角：JD 为什么要求这些知识
- 十一、面试口述钩子
- 附录：实操入口地图（去哪里动手操作）

---

## 0. 先用一组比喻建立全局

把"发布一个 iOS App"想象成"寄一份贵重的文件"：

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 二进制可执行文件 | Mach-O | **文件本身**（装订好的纸质文件） | 程序真正的二进制内容 |
| 代码签名 | Code Signing | **防伪封条**（每页都盖章，撕一页都能发现） | 证明内容没被篡改 |
| 证书 | Certificate | **你的身份证**（公安系统颁发，无法伪造） | 证明"封条是谁盖的" |
| 描述文件 | Provisioning Profile | **门禁卡**（登记了：哪栋楼、谁刷卡、能开哪扇门） | 决定"这台设备能装谁签的哪个 App" |
| 能力授权 | Entitlements | **门禁卡的附加权限**（能进机房、能坐电梯） | App 能用哪些系统能力 |
| 上传校验 | App Store Validation | **大楼管理员**（进门时验身份证+门禁卡+封条） | 安装前的最后一道审查 |

串起来一句话：**iOS 上每个 App 都是"盖了章的文件"，系统装它之前会核对身份证、门禁卡和每一页的章，任何一处对不上就拒绝安装。**

---

## 一、Mach-O（Mach Object File Format）

### 1.1 是什么

Mach-O 是 macOS / iOS 上所有"能运行的二进制"共用的文件格式。你写的 Swift/OC 代码经过编译（Compile）和链接（Link）后，就变成了 Mach-O 文件。它不只是一种文件，而是一个"家族"：

| 类型（英文） | 说明 | 在项目里的样子 |
|---|---|---|
| `MH_EXECUTE`（可执行文件 Executable） | App 主程序 | `Payload/App.app/App` |
| `MH_DYLIB`（动态库 Dynamic Library） | 运行时才加载的库 | `Frameworks/` 下 framework 里的二进制 |
| `MH_BUNDLE`（插件 Bundle / Plug-in） | 扩展程序 | 通知 Extension、键盘 Extension 等 |
| dSYM（调试符号文件 Debug Symbols File） | 崩溃解析字典 | "地址→代码行"映射表，不参与运行（严格说它是一个 bundle，其主二进制仍是 Mach-O） |

> **新手常见疑问：`.a` 静态库（Static Library）是 Mach-O 吗？**
> 不是。`.a` 本质是把一堆编译产物（目标文件 Object File，即 `.o`）打包成的归档（Archive），它自己不能运行，是在**链接阶段被拆开拷贝**进主程序的。这也是后面"符号冲突（Duplicate Symbol）"事故的根源。

### 1.2 文件内部长什么样

从上到下五个区块：

```
┌──────────────────────────────────┐
│ 头部 Header（说明书封面）           │  ← 魔数 Magic Number、CPU 架构、文件类型
├──────────────────────────────────┤
│ 加载命令 Load Commands（目录页）     │  ← 告诉系统：我有哪些部分、依赖哪些库、入口在哪
├──────────────────────────────────┤
│ __TEXT 段 Segment（正文，只读）     │  ← 机器代码 + 只读数据，权限 r-x（可读可执行不可写）
├──────────────────────────────────┤
│ __DATA 段 Segment（草稿纸，可写）   │  ← 全局变量、静态变量，运行期会被修改
├──────────────────────────────────┤
│ __LINKEDIT（附录）                │  ← 符号表 Symbol Table + 重定位 Relocation + 签名块
└──────────────────────────────────┘
```

**为什么这些结构很重要？**

- **Header 的魔数（Magic Number）`0xfeedfacf`**：系统靠它一眼识别"这是一个合法的 Mach-O"。格式错了直接拒载。
- **加载命令（Load Commands）是核心目录**：
  - `LC_LOAD_DYLIB`：列出所有依赖的动态库（所以 `otool -L` 能看到依赖清单）；
  - `LC_MAIN`：程序入口在哪；
  - `LC_CODE_SIGNATURE`：**签名块的位置**——签名不是附加在外面的，是 Mach-O 的一部分。
- **`__TEXT` 段按 4KB 分页（Page）**：代码签名不是只对整个文件算一个哈希（Hash），而是**每 4KB 一页、每页一个哈希**。这样任何一页被改（哪怕 1 个字节），那页的哈希就对不上，立刻被发现。这是 iOS 防篡改的根基。
- **`__LINKEDIT` 里的签名块（CMS 块，Cryptographic Message Syntax，即 PKCS#7）**：包含证书链、哈希清单、Entitlements——签名校验的所有材料都在这。

### 1.3 dyld（动态加载器，Dynamic Loader）是怎么加载它的（App 启动的第一步）

`dyld`（dynamic loader，动态加载器）是 iOS 系统里负责"把 Mach-O 装进内存"的程序：

1. 读 Header，确认是合法 Mach-O；
2. 读 Load Commands，知道依赖哪些动态库 → **递归**去加载它们（UIKit 这些系统库都依赖着）；
3. 重定位 Rebase/Bind：因为 ASLR（地址空间布局随机化，Address Space Layout Randomization，防攻击的安全机制），每次启动内存地址都不同，dyld 要修正代码里的指针；
4. 跳到 `LC_MAIN` 指定的入口，开始执行你的代码。

> **知识点**：系统库（UIKit 等）不在这里逐个加载，而是映射进一个提前生成好的大缓存 `dyld_shared_cache`（共享缓存）。所以你在真机文件系统里找不到 UIKit 的二进制文件；想替换系统函数一般用 `fishhook`（运行时替换符号表里的绑定）。

### 1.4 实用命令（排查问题时背这几个就够）

```bash
otool -L App              # 列出依赖哪些动态库（找缺库问题的第一步）
otool -l App              # 列出全部 Load Commands（看签名位置、入口等细节）
nm -g libX.a              # 列出库里的符号 Symbol（符号冲突时用来定位同名类/函数）
lipo -info App            # 查看文件里包含哪些架构 Architecture（排查模拟器 slice 混入）
lipo -thin arm64 App -output App64            # 只保留 arm64（真机）架构
lipo -remove x86_64 App -output App_out       # 从文件里剔除 x86_64（模拟器）架构 slice
security cms -D -i xx.mobileprovision         # 解开描述文件 Provisioning Profile，查看内容
codesign -d --entitlements - App.app          # 查看 App 的能力授权 Entitlements
codesign -vvv App.app                         # 校验签名完整性（改过文件会报错）
```

> **架构小词典**：arm64 = 真机 iPhone；x86_64 / arm64-simulator = Mac 上的模拟器（Simulator）。一个文件里同时装多个架构，叫通用二进制（Fat Binary / Universal Binary），其中每一个架构的部分叫一个 slice。

---

## 二、证书（Certificate）

### 2.1 是什么：身份证 + 一把不能复制的私钥

证书（Certificate，标准是 X.509 格式）解决的问题是：**"凭什么相信这个签名是张三盖的，而不是李四伪造的？"**

它的原理是**非对称加密（Asymmetric Cryptography）**，新手可以这样理解：

- **私钥（Private Key）= 一把独一无二的章**，只有张三自己有，盖出来的章（签名）别人无法模仿；
- **证书 = 身份证**，上面写着"这是张三"，并附上张三的"章印比对样本"（公钥 Public Key），由权威机构（Apple）盖章认证。

验证逻辑：你盖的签名，任何人都能用证书上的公钥来验——**能验通过，就说明一定是私钥持有者签的**，因为私钥全世界只有一份。

### 2.2 信任链（Trust Chain）：为什么 Apple 的认证可信

一张证书靠"上级机构签字背书"层层往上：

```
根证书 Root CA（Apple Root Certification Authority，出厂烧在 iOS 系统里，黑客碰不到）
    └── 中间证书 Intermediate CA（WWDR：Apple Worldwide Developer Relations，专管开发者业务）
            └── 你的开发者证书 Developer Certificate（由 WWDR 签发给你）
```

验证时系统沿链条一路核对到 Root CA。因为 Root CA 在系统里且无法替换，**伪造整条链等于伪造 iOS 系统**，实际上做不到。

### 2.3 私钥在哪里、怎么迁移

- 申请证书时，Xcode 先生成 **CSR（证书签名请求，Certificate Signing Request）**：在你电脑的钥匙串（Keychain）里生成一对公私钥，公钥装在 CSR 里发给 Apple，换回证书；**私钥永远留在你机器上**；
- 换电脑 / 给同事 / 配 CI 机器时，必须导出 **`.p12` 文件（证书+私钥打包，有密码保护）** 再导入；
- 证书分两类：
  - `Apple Development`：开发调试证书；
  - `Apple Distribution`：分发证书，覆盖 App Store / Ad Hoc / 企业 In-House 分发；
- 数量限制：每个 Apple ID 的开发/分发证书各有 2~3 张上限，满了必须先吊销（Revoke）一张——**Revoke 会让用这张证书签的包全部失效**，团队里要约定谁管理证书，避免互相 revoke。

---

## 三、代码签名（Code Signing）

### 3.1 签名时到底发生了什么

`codesign` 命令（或 Xcode 构建的最后一步）做三件事：

1. **算哈希**：对主程序的 Mach-O **每 4KB 页（Page）**算一个 SHA256 哈希，对 `.app` 包里的**每个资源文件**（图片、plist、内嵌 framework）也算一个，全部记录成一张清单 → 这张清单叫**代码目录（CodeDirectory）**；
2. **盖章**：用证书的**私钥（Private Key）**对 CodeDirectory 和 **Entitlements**（能力声明）整体签名；
3. **封存**：签名结果打包成 **CMS 块（Cryptographic Message Syntax，即 PKCS#7 格式）**写进 Mach-O 的 `__LINKEDIT`；资源文件的哈希清单另存为 `_CodeSignature/CodeResources`。

注意：`.app` 包里**每一个 Mach-O**（主程序、每个内嵌 framework）都要各自签名，漏一个就装不上——这是新手接第三方 framework 最常见的坑之一。

> **多 target 补充**：带 Widget / 通知 Extension / Watch App 的工程里，每个 Extension 是**独立的 target、独立的 Mach-O、各自签名**，且主 App 和 Extension 通过 **App Group** entitlement（形如 `group.com.team.app`）共享数据——App Group 必须在主 App 和 Extension **两边同时声明**，少一边数据共享就失效。

### 3.2 系统什么时候验签名

- **安装时**：`installd`（安装守护进程，Install Daemon）做全套校验——证书链（Certificate Chain）是否可信 → 逐页哈希比对 → Entitlements 与描述文件是否一致。任何一项失败，App 装不上；
- **运行时**：内核（Kernel）在内存分页调入代码时按需校验那一页的哈希。这意味着**文件装上后哪怕再被改一个字节，运行到那页也会崩**——这就是 iOS 无法像 Android 那样随意改包重分发的原因。

### 3.3 Entitlements（能力授权）：门禁卡的附加权限

Entitlements 是一个 XML plist 文件，声明 App 要用哪些"特殊能力"。新手必须知道的两个字段：

| 字段 | 含义 |
|---|---|
| `get-task-allow` | 是否允许调试器（Debugger）附加进程。开发包 = `true`（所以开发包能被 lldb/Xcode 调试）；**发行包必须 = `false`**，否则无法上架 |
| `aps-environment` | 推送（APNs，Apple Push Notification service）环境（development / production）。改描述文件时如果这个值变了，推送服务端证书也要跟着换，否则**推送静默失效** |

### 3.4 重签名（Re-signing）：改了包就必须重盖封条

```bash
# -f 强制替换现有签名；-s 指定签名证书；--entitlements 指定新的能力声明
codesign -f -s "iPhone Distribution: TeamName" --entitlements new.plist App.app
```

典型场景：
- 企业换主体 / 换开发者账号，全量包换签；
- 一个包分发多个渠道，每个渠道换自己的证书/描述文件；
- 企业内部分发换企业证书。

> **原则：只要动了 `.app` 里的任何文件（加 framework、改 Info.plist、换图标），签名即失效，必须重签。** 越狱（Jailbroken）设备可以用 `ldid -S` 做"假签名"（Fakesign），因为越狱后系统校验被绕过了，正式设备不行。

---

## 四、描述文件（Provisioning Profile）

### 4.1 是什么：一张 Apple 签发的门禁卡

描述文件（Provisioning Profile）解决的问题是：**"哪台设备，能装哪个 App，用什么能力"**。

它是一个 plist 文件，把四样东西绑在一起，并且**由 Apple 服务器用自己的私钥整体签名**——所以你手工改它没用，系统验签名就过不了。有效期约 1 年。

| 要素 | 英文 | 大白话 | 备注 |
|---|---|---|---|
| 证书列表 | Certificates | "刷卡人名单"——允许用哪几张证书签出来的包 | 签名证书必须在这张名单里 |
| App ID | App Identifier | "房间号"——`TeamID.BundleID`，可通配（`com.team.*`） | 包名（Bundle Identifier）必须匹配 |
| 设备 UDID | Unique Device Identifier | "允许进门的人"——设备硬件级唯一 ID | 只有开发/Ad Hoc 需要，App Store 包不需要 |
| 能力授权 | Entitlements | "附加权限" | 必须与 App 的 Entitlements 一致 |

### 4.2 四种类型怎么选

| 类型 | 英文/标识 | 设备数限制 | 用途 | 装机方式 |
|---|---|---|---|---|
| 开发 | Development | ≤100 台/年 | 开发调试 | Xcode 直连 |
| 测试分发 | Ad Hoc | ≤100 台 | 测试发行包（和商店包几乎一样） | 拖 ipa 安装 |
| 企业分发 | In-House（Enterprise） | 无限制 | 企业账号（Apple Enterprise Program）内部分发 | 链接/MDM 下发 |
| 商店上架 | App Store | 不限 | 商店上架 | 只有商店能装 |

### 4.3 装机时系统怎么校验（出错时按这个清单对号入座）

描述文件编译后被塞进包里，成为 `.app/embedded.mobileprovision`（内嵌描述文件）。装机时系统：

1. 解开它，验 Apple 签名（防篡改）；
2. 签名证书在"刷卡人名单"里？→ 不在报 `doesn't match signing certificate`；
3. Bundle ID 匹配房间号？→ 不匹配报 `provisioning profile doesn't include ...`；
4. 设备 UDID 在名单里？→ 不在报 `device not registered` / 直接提示无法安装；
5. Entitlements 一致？→ 不一致推送/登录等能力失效，甚至装不上。

**排查万能命令**：

```bash
security cms -D -i embedded.mobileprovision
# 重点看：ExpirationDate（过期时间）、ProvisionedDevices（设备名单）、Entitlements（权限）、Name（用的哪张证书）
```

---

## 五、ipa 包（iOS App Store Package）

### 5.1 ipa 是什么：一个改了后缀的 zip 压缩包

ipa（iOS App Store Package）是 iOS App 的**最终分发格式**——TestFlight、App Store、Ad Hoc、企业分发，装的统统是 ipa。

新手最重要的认知：**ipa 本质就是一个 zip 压缩包**。把 `.ipa` 后缀改成 `.zip` 解压，就能看到 App 的全部内容：

```
App.ipa（本质是 zip 压缩包）
└── Payload/
    └── App.app/                     ← 应用包（App Bundle），一个目录
        ├── App                      ← 主程序（Mach-O 可执行文件，签名就盖在它身上）
        ├── Info.plist               ← 包身份信息：版本号、Bundle ID、图标、权限声明
        ├── embedded.mobileprovision ← 内嵌描述文件（装机校验的门禁卡，见第四节）
        ├── _CodeSignature/
        │   └── CodeResources        ← 所有资源文件的哈希清单（签名的一部分）
        ├── Frameworks/              ← 内嵌动态 framework（每个都带自己的独立签名）
        ├── Assets.car               ← 编译后的图片资源包
        └── ...（图标、多语言文件等资源）
```

> **对照前面学的知识**：装机时系统验的就是这个包里的东西——主程序的页哈希、每个 framework 的签名、`embedded.mobileprovision` 四要素、`CodeResources` 资源清单，一环扣一环。任何一处被动过，对应环节就报错。

### 5.2 ipa 是怎么产生的

生成过程分两步，**签名在第一步就完成了**：

```
Xcode/fastlane：xcodebuild archive
    → .xcarchive（归档：未导出的完整产物，含 dSYM、符号信息）
        → export（导出，此时按 distribution 方法选定签名与描述文件）
            → .ipa（最终上传/分发文件）
```

- **.xcarchive vs .ipa 的区别（新手必懂）**：archive 是"留档的全量产物"（二进制 + dSYM + 符号表），ipa 是从 archive 导出的"上架件"。上架后发现崩溃，**要用同一个 archive 解析 dSYM**，所以 archive 必须永久归档；包内容有争议时也从原 archive 重新导出，而不是重新编译——重新编译出的二进制和线上那份已经不一样了；
- 导出时的分发方式（Export 方法）决定签名细节：`App Store Connect` / `Ad Hoc` / `Enterprise` 各对应不同的描述文件与 entitlements（比如 `get-task-allow` 都会被强制置为 `false`）；
- **dSYM 要主动喂给崩溃平台**：archive 里的 dSYM 不会自动上传。Firebase Crashlytics 用 `upload-symbols` 命令，Bugly 用官方符号表工具（jar 包），都按 **UUID 匹配**——dSYM 的 UUID 和二进制的 UUID 对得上才能解析堆栈。

### 5.3 ipa 的常用操作（排查问题就这几招）

```bash
# 1. 解包：改后缀解压，或直接 unzip
unzip App.ipa -d unpacked/

# 2. 查看 Info.plist（版本、包名、权限声明）
plutil -p unpacked/Payload/App.app/Info.plist

# 3. 校验签名完整性（--deep 连内嵌 framework 一起验）
codesign -vvv --deep unpacked/Payload/App.app

# 4. 重签：动了包内任何文件后，framework 逐个重签 → 主程序重签 → 重新 zip
codesign -f -s "Apple Distribution: Team" unpacked/Payload/App.app/Frameworks/X.framework
codesign -f -s "Apple Distribution: Team" --entitlements new.plist unpacked/Payload/App.app
zip -r App.ipa Payload/
```

上传渠道：Xcode Organizer（图形界面）、Transporter（苹果官方工具）、`fastlane pilot`/`deliver`（CI 自动化）。上传后 App Store Connect 会对包内所有 Mach-O 做服务端校验——第七节事故二（ITMS-90426）就发生在这一步。

> **一句话总结**：ipa = zip 壳 + App Bundle，Bundle 里躺着主 Mach-O、描述文件和签名清单；**签名不是贴在 ipa 外面的标签，而是渗透在包内每个二进制里的指纹**。

---

## 六、出包实战三条路（Xcode 手动 / Jenkins 自动化 / 蒲公英分发）

> 三条路线解决三个不同的问题：**Xcode 手动打包**是地基（第一次必须走通，理解每一步发生了什么）；**Jenkins 自动化**解决"天天打包、谁都能打包"；**蒲公英**解决"测试包几分钟发到几十个测试员手机上"。

| 路线 | 定位 | 适用场景 |
|---|---|---|
| Xcode 手动打包 | 图形界面，看得见摸得着 | 新手学习、偶尔出正式包 |
| Jenkins + fastlane | 服务器自动构建，谁触发谁打包 | 每日构建、团队协作、CI 标准 |
| 蒲公英（Pgyer） | 测试包托管分发 | 快速给测试同学/客户发体验包 |

### 6.1 路线一：Xcode 手动打包（新手必须先完整走一遍）

**前提检查**：Apple 账号已登录（`Xcode → Settings → Accounts`）、签名配置无红字（target → Signing & Capabilities）。

**完整步骤**（每步在哪个界面）：

1. **选设备目标**：Xcode 顶部工具栏的设备下拉框，选 **`Any iOS Device (arm64)`**——不能选模拟器或某台具体真机，否则 Archive 选项不可用；
2. **开始归档**：菜单 `Product → Archive`（如果是灰色，检查当前选中的 scheme 是否支持 iOS、设备目标是否为真机架构）；
3. **等待编译完成**：Xcode 自动开始 Release 构建并签名，成功后自动弹出 **Organizer** 窗口（也可手动打开：`Window → Organizer`）；
4. **分发导出**：Organizer 里选中刚才的 archive → 点右侧 **`Distribute App`** 按钮 → 选分发方式：
   - `App Store Connect`：上传到商店/TestFlight（选 Upload，一路下一步，Xcode 自动重签）；
   - `Ad Hoc`：导出 ipa 文件（用于测试机直装/传蒲公英）；
   - `Enterprise`：企业账号内部分发导出；
   - `Development`：开发包导出；
5. **拿到产物**：导出完成后点 `Export`，选择保存目录，得到 `.ipa` 文件；archive 本体在 `~/Library/Developer/Xcode/Archives/` 目录（dSYM 就在里面，**这个目录别删**）；
6. **上架提审**（商店包）：上传成功后登录 `appstoreconnect.apple.com` → **我的 App（My Apps）** → 选中 App → 左侧 `App Store → 版本或平台` → 在"构建版本"区点 **+** 选择刚处理完的 Build → 完成版本说明、截图等信息 → 右上角**`提交以供审核（Submit for Review）`**。审核被拒的打包相关高频原因：权限文案缺失（Guideline 5.1.1）、启动崩溃（2.1）。**提审前的元数据、审核实务与发布方式，见第九节全景 SOP。**

**新手最容易卡的两步**：
- Archive 灰色/报 `No signing certificate` → 回到附录 D 排障顺序，从 Signing & Capabilities 查起；
- Distribute 时报 `No matching provisioning profiles found` → 手动签名时 profile 类型要和分发方式对应（Ad Hoc 导出必须选 Ad Hoc profile，不能拿 App Store profile 导 Ad Hoc 包）。

### 6.2 路线二：Jenkins 自动化打包

**是什么（新手版）**：Jenkins 是一个开源的持续集成（CI）服务器——本质上是一个**装在某台 Mac 上的网页服务**，它定时或被触发后自动执行你配置的脚本（打包命令），把结果（成功/失败、产物、日志）记录在网页上。**注意：真正干打包活的不是 Jenkins 本身，而是它调用的 fastlane / xcodebuild；Jenkins 负责"什么时候打、打了记下来、失败了通知谁"。**

**典型架构**：

```
Git push / 定时触发
   → Jenkins（调度：拉代码、跑脚本、记录日志）
      → fastlane / xcodebuild（实际打包：archive + 签名）
      → 上传：蒲公英 / TestFlight / App Store Connect
      → 通知：飞书 / 钉钉 / 企业微信 Webhook 群消息
```

**搭建步骤（每一步在哪里操作）**：

1. **装 Jenkins**（打包机 Mac 上）：
   - 方式一：官网下载 `.pkg` 安装包（`jenkins.io/download`），装完浏览器自动打开 `http://localhost:8080`；
   - 方式二：终端 `brew install jenkins-lts`，启动 `brew services start jenkins-lts`；
   - 首次打开要求输入初始密码，页面会告诉你密码文件路径（`~/.jenkins/secrets/initialAdminPassword`），终端 `cat` 一下复制进去；
2. **装插件**：Jenkins 网页 → `系统管理（Manage Jenkins）→ 插件管理（Plugins）`→ 搜索安装 `Git` 插件（拉代码用）；
3. **装打包环境**（打包机终端）：Xcode（完整版，Command Line Tools 不够）、`sudo gem install fastlane`、CocoaPods；**证书和描述文件导入这台机器的钥匙串**（推荐 fastlane match 一条命令搞定，见 7.3）；
4. **新建任务（Job）**：Jenkins 网页首页 → `新建 Item（New Item）→ Freestyle project`（自由风格项目，新手够用）；
5. **配置任务**（任务详情页左侧菜单逐项配）：
   - `源码管理（Source Code Management）`：选 Git → 填仓库地址 → 添加凭据（Credentials，即 Git 账号/token）；
   - `构建触发器（Build Triggers）`：可选"定时构建"（如 `H 20 * * *` 每晚 8 点）、"轮询 SCM"，或装 Webhook 插件由 Git push 触发；新手先用手动"立即构建（Build Now）"跑通；
   - `构建（Build）→ 增加构建步骤 → 执行 shell（Execute shell）`：粘贴打包脚本（见下方示例）；
6. **配置产物归档与通知**：`构建后操作（Post-build Actions）→ Archive the artifacts` 填 `build/*.ipa`；通知在 shell 脚本末尾用 curl 调飞书/钉钉 Webhook。

**核心打包脚本（两种写法二选一）**：

写法 A——纯 xcodebuild（理解底层）：

```bash
# 1. 归档（编译 + 签名一步完成）
xcodebuild -workspace MyApp.xcworkspace \
  -scheme MyApp \
  -configuration Release \
  -archivePath build/MyApp.xcarchive \
  archive

# 2. 从 archive 导出 ipa（ExportOptions.plist 指定分发方式和证书）
xcodebuild -exportArchive \
  -archivePath build/MyApp.xcarchive \
  -exportPath build \
  -exportOptionsPlist ExportOptions.plist
```

> **ExportOptions.plist 是什么**：导出配置文件（method=ad-hoc/app-store、teamID、签名样式）。**最省事的生成方法**：先用 6.1 的 Xcode 手动流程导出一次 Ad Hoc 包，导出目录里会自动生成一个 `ExportOptions.plist`，拿去改改就能用。

写法 B——fastlane（推荐，工具链自动处理细节）：

```ruby
# fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "打包并上传蒲公英"
  lane :beta do
    match(type: "adhoc")                # 从 Git 仓库自动装好证书+描述文件
    gym(                                # gym = 打包（archive+导出）
      scheme: "MyApp",
      export_method: "ad-hoc",
      output_directory: "build"
    )
    pgyer(api_key: "填蒲公英的 API Key")  # 上传蒲公英
    # 上 TestFlight 改成：upload_to_testflight
  end
end
```

本地先验证：工程根目录 `fastlane beta`；跑通后把这一行填进 Jenkins 的 shell 步骤即可。

**Jenkins 打包机新手三大坑**：
1. 裸机没证书 → `No signing certificate`，用 match 或手动导入 .p12；
2. Jenkins 以守护进程运行时**钥匙串处于锁定状态**，签名报错 → 脚本开头加解锁命令（match 会自动处理，纯 xcodebuild 需要自己 `security unlock-keychain`）；
3. Xcode 版本路径不对 → 终端 `sudo xcode-select -s /Applications/Xcode.app` 指定。

### 6.3 路线三：蒲公英（Pgyer）——测试包几分钟发到测试员手机

**是什么（新手版）**：蒲公英（pgyer.com）是国内的测试包托管分发平台。把签好的 ipa 传上去，它给你生成**二维码和下载短链**，测试员扫码就能装。**注意分工：蒲公英只管分发，不管签名——ipa 必须先在 Xcode/Jenkins 里签好（Ad Hoc 或企业签），蒲公英只是"快递员"。**

**与 TestFlight 怎么选**：

| 对比项 | 蒲公英 | TestFlight |
|---|---|---|
| 审核 | 无需审核，传完即装 | 外部测试要 Beta App Review |
| 安装方式 | 二维码/短链，国内访问快 | 需要 App Store 环境 |
| 前提 | Ad Hoc 包（设备 UDID 已注册）或企业签 | App Store 类型的包 |
| 适用 | 快速迭代、给客户演示 | 正式灰度、苹果背书 |

**网页操作步骤（在哪个界面）**：

1. 注册/登录 `pgyer.com` → 首页点 **`发布应用`** → 把 ipa 拖进上传框；
2. 上传完成自动解析出包信息（版本号、图标、包名）→ 系统生成**安装页**（含二维码 + 短链）；
3. 测试员扫码 → 浏览器打开安装页 → 点安装 →（Ad Hoc 包需 UDID 已注册，企业签包首次安装要在手机上信任证书）；
4. 版本管理：同一个应用重新上传会生成新版本记录，可在应用详情页看到历史版本列表；
5. 权限设置（应用详情 → `应用设置`）：可开启"安装密码"、"UDID 白名单"（只有指定设备能装）。

**手机侧信任企业证书（企业签/部分 Ad Hoc 安装后打不开时）**：

```
设置 → 通用 → VPN 与设备管理 → 找到对应的企业证书 → 点击"信任"
```

> **合规提醒**：企业证书（In-House）按苹果条款只能给**企业内部员工**使用，拿企业签给外部用户大量分发属于违规，有封号风险。给外部测试走 **Ad Hoc（注册 UDID）+ 蒲公英** 或 TestFlight 才是正路。

**命令行/自动化上传（接进 Jenkins）**：

```bash
# 方式一：curl 直传（最简单）
curl -F "file=@build/MyApp.ipa" \
     -F "_api_key=你的APIKey" \
     https://www.pgyer.com/apiv2/app/upload

# API Key 位置：蒲公英网页右上角头像 → API 信息 → API Key
```

方式二：fastlane 插件（`fastlane add_plugin pgyer`，lane 里写 `pgyer(api_key: "xxx")`，见 6.2 写法 B）——Jenkins 打包 + fastlane + 蒲公英通知，就构成了一条完整自动化流水线。

---

## 七、发版流水线（Release Pipeline）事故排查（5 类事故源）

> 共同规律：**这些故障不在代码里，在环境里**。代码问题本地一跑就知道；下面这些问题要到打包、上传甚至安装时才爆——而那时通常正是发版截止日。CI（Continuous Integration，持续集成）环境里的故障尤其常见，因为 CI 机器和你本地环境不同。

```
编译链接 ──→ codesign 签名 ──→ 上传校验 ──→ 分发安装
符号冲突        签名失败         ITMS 打回      装机失败
slice 混入      profile 过期     加密合规未填    运行崩溃
（越靠右暴露的问题，发现越晚、排查成本越高）
```

### 7.1 事故一：符号冲突（Duplicate Symbol）

**现象**：链接（Linking）阶段直接报错，构建失败：

```text
duplicate symbol '_OBJC_CLASS_$_OpenSSLUtils' in:
    libThirdPay.a(OpenSSLUtils.o)
    libPushSDK.a(OpenSSLUtils.o)
ld: 1 duplicate symbol for architecture arm64
```

**原理（新手版）**：OC（Objective-C）没有"命名空间（Namespace）"，全 App 的类名必须唯一。A SDK 和 B SDK 各自把同一份底层库（OpenSSL、libcurl、protobuf 等是惯犯）**编译进了自己的 `.a` 静态库**。链接器（Linker）把两份同名类拷进同一个程序时发现撞名，直接拒绝。

**为什么总在最后才爆**：只有所有业务方的 SDK 都集成完，才会编一次完整包——所以冲突总出现在发版前几天，然后开始跨团队扯皮（A 说 B 删，B 说 A 删）。

**解法**（按推荐顺序）：
1. 定位冲突范围：`nm -g libX.a | grep "OBJC_CLASS_\$_SameName"`，看两个库里同名符号有多少；
2. 首选：让 SDK 方提供"不含底层库"的版本，全项目共用一份；
3. 拿得到源码：给其中一个版本的类加前缀（Prefix）重编译（OC 只能靠改名隔离）；
4. 高级：链接器别名 `-Wl,-alias,_OldSymbol,_NewSymbol` 给其中一份换符号名；
5. 兜底：把冲突的公共库抽成一个动态 framework（Dynamic Framework），两个 SDK 都改为外链它。

### 7.2 事故二：模拟器 slice 混入（Simulator Slice Contamination）

**现象**：本地真机一切正常，上传 App Store Connect 几分钟后收到邮件：

```text
ITMS-90426: Invalid Swift Support — The files ... include
architectures ['x86_64'] instead of ['arm64'].
```

更隐蔽的变体：审核被拒 / 用户设备启动闪退。

**原理（新手版）**：一个 Mach-O 可以同时装多个架构（Fat Binary，通用二进制），就像一本书附了中英两版内容。第三方 framework 为了方便自己调试，附带了 `x86_64`（Intel 模拟器）版；构建脚本把它整个拷进 app，"模拟器专用的那一版"就跟着上架了。Apple 服务端检查包里所有 Mach-O 的架构，发现非法内容直接打回。

**解法**：
1. **根治**：推动 SDK 方发 **xcframework**——苹果官方格式，按平台分目录（`ios-arm64/`、`ios-arm64_x86_64-simulator/`），Xcode 按目标平台自动选对应目录，从机制上杜绝混入；
2. 拿到的还是老式 framework：上架前跑脚本，遍历 `Frameworks/` 对每个 Mach-O 执行 `lipo -remove x86_64`（俗称 Strip Frameworks；第三方 SDK 多的老工程基本都要配这个脚本进 CI）；
3. 临时压制：`EXCLUDED_ARCHS[sdk=iphonesimulator*]=arm64`——注意这只影响编译选型，救不了已经拷进包里的 framework。

### 7.3 事故三：签名错（Code Sign Error）

**现象**：

```text
Code Sign error: No signing certificate ("iOS Distribution") found:
No valid private key pairs were found in the keychain.
```

**原理（新手版）**：签名需要"证书+私钥"配对（Keypair，密钥对）。这个报错的意思是：你钥匙串里有证书（身份证），但没有对应的私钥（章）——因为证书是别的电脑申请的，私钥留在那台电脑上。

**三种根因**：
1. 证书和私钥不配对（换电脑 / 用了同事发的证书）；
2. 证书被吊销（Revoke）（证书数量到上限，有人删了你的）；
3. CI 机器是裸机（Clean Machine），从没装过证书。

**解法**：
- 个人环境：让证书持有者导出 `.p12` 给你导入；或 Xcode Settings 里勾选云签名（Cloud Signing，私钥由 Apple 托管）；
- **CI 才是重灾区**：用 `fastlane match`——把证书和描述文件加密存进 Git 仓库，CI 每次构建自动解密安装，彻底告别"祖传打包机"（那台只有它能出包、别人不敢动、离职就失传的机器）。

### 7.4 事故四：profile 过期 / 不匹配（Profile Expired / Mismatch）

**现象**（最阴险——编译签名全绿，最后装机才爆）：
- Ad Hoc / 企业包装机失败：`A valid provisioning profile for this executable was not found`；
- **已装用户某天 App 集体打不开**（企业分发 profile 一年过期，没人记得续期的经典事故）；
- 推送静默失效：profile 换新后 `aps-environment` 与服务端推送证书不匹配，用户无感知直到投诉。

**原理（新手版）**：描述文件有效期 1 年且 Apple 不提醒；装机时系统会校验它——过期/设备不在名单/证书对不上，安装直接失败。企业分发包尤其危险：**用户已经装好的 App 也会因为内嵌 profile 过期而失效**（下次启动被系统拒绝）。

**解法**：
```bash
security cms -D -i embedded.mobileprovision   # 重点检查 ExpirationDate（过期时间）
```
- CI 出包后加一步脚本，自动检查 profile 剩余有效期和设备名单，**比用户先发现问题**；
- 企业分发设置到期提醒（提前 30/60 天），换签流程写成文档。

### 7.5 事故五：WWDR 中间证书轮换（Certificate Chain Rotation）

**现象**：本机什么都没改，某天大量安装突然全线失败；钥匙串里证书显示"此证书已吊销/不再有效"，开发者中心里 profile 状态集体异常。

**原理（新手版）**：回顾 2.2 的信任链——你的开发者证书由 **WWDR 中间证书**背书。Apple 会**定期轮换这张中间证书**：官方帮助页记载，旧 WWDR 中间证书已于 2023 年 2 月 7 日过期，续期版有效期至 2030 年 2 月 20 日，轮换期间还会核发额外细分用途的中间证书。中间证书一换，用旧证书链签的包、旧 profile **立刻全部失效**——它和事故四的区别是：这不是你的 profile 过期，而是 Apple 主动换锁，全世界开发者同时"被迫搬家"。

**解法**：
1. 到 `developer.apple.com` 重新下载最新的 **WWDR 中间证书**，双击导入钥匙串（放入系统钥匙串）；
2. 重新生成并下载**所有 profile**（重新生成的 profile 由新中间证书签发）；
3. CI / fastlane match 仓库里的证书与 profile 同步更新，重新出包；
4. 已分发出去的旧包（企业/Ad Hoc）救不回来，只能重签后重新分发，用户删旧装新。

**预防**：订阅 Apple 开发者新闻/邮件通知；证书管理用 match 版本化，出事时全员一条命令恢复，而不是各自找证书。

---

## 八、上传前打包高频清单（新手最容易漏的环节）

> 第七节解决"包挂了怎么办"，这一节解决"包还没挂、但大概率会被卡"的检查项。每次上架前过一遍，能消灭大部分上传/审核返工。

### 8.1 Info.plist 必查项

| 字段 | 说明 | 不查的后果 |
|---|---|---|
| Version（`CFBundleShortVersionString`） | 对外展示版本号 | — |
| Build（`CFBundleVersion`） | 内部构建号：同一 Version 下必须**递增**，且被上传占用的号不能复用 | 上传被拒：`build number already used` |
| 权限文案（Privacy Usage Descriptions） | 高频六个：`NSCameraUsageDescription`（相机）、`NSLocationWhenInUseUsageDescription`（定位）、`NSPhotoLibraryUsageDescription`（相册）、`NSMicrophoneUsageDescription`（麦克风）、`NSBluetoothAlwaysUsageDescription`（蓝牙）、`NSUserTrackingUsageDescription`（广告追踪 ATT） | **缺哪个，运行到对应 API 直接闪退**，审核必拒 |
| 出口加密合规 `ITSAppUsesNonExemptEncryption` | App 只使用系统/豁免加密时设为 `NO` | 不填则每次上传都要人工回答合规问卷；设 `NO` 可跳过 |
| ATS 例外（`NSAppTransportSecurity`） | 第三方 SDK 走 http 的域名例外要逐个写明 | 上传能过，但审核可能以安全理由拒绝 |

> 权限文案缺失是"本地好好的、用户机器上闪退"的头号原因——上线前用真机把每个会弹权限的功能点一遍。

### 8.2 TestFlight 分发流程（灰度测试标准工具）

- **内部测试（Internal Testing）**：最多 100 名团队成员，上传后立即可装，**无需审核**；
- **外部测试（External Testing）**：最多 10000 名外部用户，需通过 Beta App Review（远比正式审核宽松），支持公开链接邀请；
- **每个 Build 有效期 90 天**，到期自动失效——测试群里突然有人喊"装不上了"，八成是这个；
- 测试员反馈截图和崩溃报告都在 App Store Connect → TestFlight 标签页查看。

### 8.3 常见 ITMS 错误码速查（上传后收到苹果邮件时对号）

| 错误码 | 含义 | 对应处理 |
|---|---|---|
| ITMS-90426 | Swift Support 目录架构非法（模拟器 slice） | 见 7.2 事故二 |
| ITMS-90087 | Bundle 结构非法：framework 内又嵌 framework / 防拷贝问题 | 检查 Frameworks 嵌套层级（只允许一层） |
| ITMS-90158 | Info.plist 缺 key 或值非法 | 按邮件指出的 key 补齐 |
| ITMS-90171 / 90186 | Invalid Bundle：二进制缺失或多余 | 清理 Derived Data 重新 Archive |
| ITMS-90535 | 架构/符号重复导致 universal binary 非法 | `lipo` 处理重复 slice |

（错误码以邮件原文为准，邮件会给出具体文件路径。）

### 8.4 Bitcode 已废弃（别被旧教程带偏）

2015–2022 年的旧教程会讲 "Enable Bitcode = NO" 的各种坑。按官方 Xcode 14 Release Notes：**Xcode 14 起 Bitcode 弃用，App Store 不再接受 bitcode 提交**（一种上传中间码、由苹果服务器代编译的机制），之后的 Xcode 版本连构建设置项都已移除——遇到相关内容直接跳过。

### 8.5 审核与二进制扫描（一句话了解）

App Store 审核会扫描包内二进制的**符号表**，检测私有 API 调用。金融等敏感类 App 常做的"字符串混淆、符号混淆"加固，防的就是这个——这也是 Mach-O 符号表知识的又一个实际用途。

### 8.6 Universal Link（通用链接）与签名的关系

深链接依赖三件套：`Associated Domains` entitlement + 域名服务器上的 **AASA 文件**（apple-app-site-association，内含 Team ID + Bundle ID）+ HTTPS 部署。三者缺一，链接直接退化为普通网页跳转。**换证书/换 Team 重签后 Team ID 变了而 AASA 没更新，深链接立刻失效**——换主体迁移时最常踩的隐藏坑。

### 8.7 隐私清单与 Required Reason API（2024 起的上传新门槛）

**是什么**：苹果要求 App 和包内的第三方 SDK 都要携带**隐私清单文件（Privacy Manifest，即 `PrivacyInfo.xcprivacy`）**——声明三件事：收集了哪些数据、是否用于追踪（tracking）、使用了哪些"必需理由 API（Required Reason API）"。

**Required Reason API（新手最容易漏的）**：一批"有滥用风险"的系统 API 必须在清单里写明使用理由代码，高频类别：

| API 类别 | 典型场景 | 理由代码示例 |
|---|---|---|
| `UserDefaults` | `UserDefaults.standard` 存偏好设置 | `CA92.1`（仅访问 App 自身数据） |
| 系统启动时间 / 磁盘空间 / 文件时间戳 / 键盘活跃 | 统计、日志等 | 各有对应理由码，按实际用途选 |

**新手实操**：
- Xcode 15+：`File → New → File → 搜 "App Privacy"` 创建隐私清单文件，加进 target；自己的 framework 也要各自带一份；
- 第三方 SDK：新版本（Firebase、微信 SDK 等主流 SDK）已自带清单；老版本 SDK 联系升级；
- **报错表现**：上传后收到邮件 `ITMS-91053`（缺必需理由）/ `ITMS-91056`（清单无效）/ `ITMS-91061`（隐私影响型 SDK 缺清单）等——按邮件列出的 API 类别补声明即可（错误码以邮件原文为准）；
- **生效时间**：官方 Tech Note TN3181 载明，2024 年 11 月 12 日起提交审核的 App 必须包含有效的隐私清单。

> 不用背理由码：按邮件提示补、或查苹果官方 Required Reason API 文档。关键是**知道有这道门**，别等上传被拒才第一次听说。

---

## 九、从零到上架全景 SOP（新手按序执行）

> 前面的章节是"知识点"，这一节是"执行线"。**第一次上架，照着下面的步骤顺序走**；卡住了再回到对应章节查原理和排障。

### 9.0 全景步骤总览（收藏这张）

```
第 0 步 账号 → 第 1 步 开发者中心建档 → 第 2 步 工程签名配置
    → 第 3 步 App Store Connect 建 App 记录 → 第 4 步 元数据
    → 内购配置 / ICP 备案（付费或国内上架时，见 9.7 / 9.8）
    → 第 5 步 打包上传 → 第 6 步 TestFlight 自测
    → 第 7 步 提审 → 第 8 步 发布
```

### 9.1 第 0 步：账号（Apple Developer Program）

**四种账号对比（先选对，别买错）**：

| 类型 | 费用 | 能力 | 限制 |
|---|---|---|---|
| 免费（个人 Apple ID） | 0 元 | **只能真机调试** | 签出的 App **7 天后过期**要重新签、最多 3 台设备、**不能上架**、不能用推送/内购等能力 |
| 个人开发者 | $99/年 | 完整能力 + 上架 | 商店页开发者显示**个人姓名** |
| 公司开发者 | $99/年 | 完整能力 + 上架 | 注册需**邓白氏码（D-U-N-S Number）**，审核更久；商店显示公司名 |
| 企业账号（Enterprise） | $299/年 | **不能上架 App Store**，只能企业内部分发（In-House） | 申请门槛高，苹果严控 |

**注册入口**：`developer.apple.com/programs` → **Enroll（加入）** → 用 Apple ID 登录（先开双重认证）→ 选择个人/公司 → 实名信息 → 付款 → 等待开通（个人 1~2 天；公司要先拿邓白氏码，可能要几周）。

**新手建议**：
- 学习阶段用**免费账号**真机调试足够，上架前再买 $99；
- 给公司做 App **一开始就用公司账号**——后期换账号主体，Bundle ID、Keychain、深链接、推送全部要联动迁移，成本极高；
- 一个 $99 账号可以发多个 App，不限数量。

### 9.2 第 1 步：开发者中心建档

`developer.apple.com` → Certificates, Identifiers & Profiles → 注册 **App ID（Identifiers）**：精确包名 + 勾选需要的 Capability（推送等）。操作细节见**附录 A**。个人签名的可以跳过这步（Xcode 自动签名会代劳），CI 手动签名必须做。

### 9.3 第 2 步：工程签名配置

- target → Signing & Capabilities：选 Team、确认无红字；
- `General → Identity`：填 Bundle ID、Version、Build；
- Info.plist：按**第八节 8.1 清单**填齐权限文案等；
- 有 Capability 需求（推送/ associated domains）在这里添加。

### 9.4 第 3 步：App Store Connect 创建 App 记录

`appstoreconnect.apple.com` → **我的 App（My Apps）→ 左上角 + → 新建 App（New App）**：

- **名称**：商店显示名，**全局唯一**（被占用只能改名或加后缀）；
- **主要语言**：界面语言；
- **Bundle ID**：下拉选择（只有开发者中心已注册的才会出现，这就是 9.2 的意义）；
- **SKU**：内部标识，一般填包名，用户看不到。

### 9.5 第 4 步：元数据清单（提审前必须全部填完，否则提交按钮是灰的）

在 App 的 `App Store` 标签页逐项填写：

| 项目 | 要求 | 新手易漏 |
|---|---|---|
| 截图（Screenshots） | 6.9 英寸 iPhone 必填；支持 iPad 还要 13 英寸 iPad 截图 | 必须是真机实拍效果，功能不符会被拒 |
| 描述（Description） | ≤4000 字 | — |
| 关键词（Keywords） | ≤100 字符，逗号分隔 | 影响商店搜索 |
| **隐私政策 URL（Privacy Policy URL）** | **必填**，必须是可访问的网页 | **没有网址提不了审**——提前在公司官网/GitHub Pages 建一个 |
| 支持 URL（Support URL） | 必填 | 可用官网或客服页 |
| 分类 / 年龄分级问卷 | 按实际内容选 | 分级问卷如实答，虚报会被拒 |
| **App Privacy（隐私标签）** | 数据收集问卷 | **要和实际 SDK 收集行为对上**（三方 SDK 常收集设备标识符/使用数据），填少了被拒、填错了虚假申报 |
| 审核信息（App Review Info） | 演示账号/联系方式 | **给审核员一个能登录的测试账号**，否则功能进不去直接被拒 |

### 9.6 第 5~6 步：打包上传 + TestFlight 自测

打包上传走**第六节**三条路线任选（新手用 6.1 Xcode 手动）。

上传后**先别急着提审**：在 TestFlight 里装到真机完整跑一遍——每个权限弹窗、登录、支付入口、弱网场景都点一遍。**审核员就是普通用户，你自己都没跑通的版本，提审大概率被拒。**

### 9.7 内购（IAP）与收款协议（付费 / 订阅类 App 必读）

**收款前提（比创建商品更早！）**：App Store Connect → **协议、税务和银行业务（Agreements, Tax, and Banking）** → 签署 **Paid Applications Agreement（付费应用协议）** + 填写银行与税务信息。**协议没签，付费下载和内购都无法提审、无法收款**——新手最常在这一步卡住。

**四种内购商品类型**：

| 类型 | 适用 | 特点 |
|---|---|---|
| 消耗型（Consumable） | 金币、体力、道具 | 用完即耗 |
| 非消耗型（Non-Consumable） | 永久解锁、去广告 | 绑定 Apple ID，必须提供"恢复购买" |
| 自动续期订阅（Auto-Renewable） | 会员、VIP | 到期自动扣款，需配置订阅组 |
| 非续期订阅（Non-Renewing） | 课程、季卡（自定义时长） | 服务端自行管理到期 |

**创建位置**：App Store Connect → 我的 App → 选中 App → 左侧 **功能（Features）→ App 内购买项目 / 订阅 → + 创建**：
- **Product ID**：全局唯一，**创建后不可删除（只能停用）、不可复用**——命名前想清楚，建议带版本前缀（`com.team.app.gold100`）；
- 定价基于**价格等级（Price Tier）**，各国货币按等级自动换算；
- **商品也要过审**：提审版本时，该版本用到的内购商品要一起提交（状态为"准备提交"），过审后商品才可用。

**代码侧要点**：
- 用 **StoreKit 2**；本地调试用 **StoreKit Configuration File**（Xcode → `File → New → File → StoreKit Configuration File`，模拟购买不真实扣款，也不用等审核）；
- 真实环境测试用**沙盒账号**：App Store Connect → `用户和访问 → Sandbox → 测试员`（用沙盒 Apple ID 在真机上测试购买，不扣真钱）；
- **红线**：数字内容/功能解锁必须走 IAP，不得接入支付宝/微信等第三方支付、不得跳转外部购买（Guideline 3.1.1，见 9.10 被拒表）。

### 9.8 中国大陆上架：ICP 备案（国内团队必看）

**政策**：2023 年 9 月起，**中国大陆区 App Store 上架/更新必须提供 App 备案号（ICP 备案）**——没有备案，中国区无法过审（港澳台及海外区不受影响）。

**流程（App 备案由云服务商代办）**：
1. 在阿里云/腾讯云等国内云服务商的备案系统（"ICP/IP 地址/域名信息备案管理系统"）发起 **App 备案**；
2. 提交主体信息（公司营业执照 / 个人身份证）+ App 信息（**名称、Bundle ID、平台**）→ 服务商初审 → 通信管理局审核（数天到数周）；
3. 获得**备案号**（形如 `粤ICP备XXXXXXXXXX号-XXA`）；
4. 填写位置：App Store Connect → App 信息/App 隐私详情 → **中国大陆 App 备案号** 字段。

**注意**：备案绑定 **Bundle ID**——换包名等于重新备案；App 使用的**域名也要在同一主体下备案**；备案周期要预留进发版计划（首次备案可能要 2~4 周）。

### 9.9 第 7 步：提审

版本页选择 Build（9.4 的 App 记录里）→ 补完 9.5 的元数据 → **提交以供审核**。

- **审核时长**：目前一般 **24~48 小时**（大版本节点会慢）；
- **状态流转**：等待审核 → 正在审核 → 通过/被拒，全程 App Store Connect 站内 + 邮件通知；
- 审核备注里写清演示账号、特殊功能的操作路径（蓝牙/定位等需要环境的，说明测试方法）。

### 9.10 被拒了怎么办（审核实务）

**不要慌，被拒是常态**。流程：App Store Connect → 该版本页 → **Resolution Center（解决方案中心）** → 看拒绝理由 → 修复或回复说明 → 重新提交。

| 高频拒绝条款 | 含义 | 处理 |
|---|---|---|
| Guideline 2.1 | 崩溃 / 主要功能不可用 | 用崩溃日志修复，**换新 Build 重新提** |
| Guideline 2.3.1 | 截图/描述与实际功能不符 | 换真实截图，元数据如实展示 |
| Guideline 4.2 | 最低功能（网页套壳、信息量过低） | 增加原生功能与差异化价值 |
| Guideline 4.3 | 换皮 / 与现有 App 重复 | 在 Resolution Center 说明差异化，必要时改功能 |
| Guideline 5.1.1 | 权限文案缺失或不合理 | 补齐 Info.plist 权限描述（见 8.1） |
| Guideline 5.1.1(v) | 支持账号注册的 App 未提供**账号删除**入口 | 加"删除账号"功能（2022 起强制要求） |
| Guideline 5.1.2 | 数据收集与隐私标签不符 | 修正 App Privacy 问卷或实际收集行为 |
| Guideline 5.2.3 | 使用未授权的第三方素材 / 侵权 | 删除或取得授权 |
| Guideline 3.1.1 | 数字内容绕过苹果内购 | 接 IAP 或移除相应支付方式（见 9.7） |

> 条款全文动态更新，提审前浏览官方页面：`developer.apple.com/app-store/review/guidelines`——本表列的是高频稳定项，不替代官方最新版。

**加急审核（Expediting Review）**：紧急 bug 修复或活动时效需要插队时，在开发者支持页面提交加急申请（说明理由，次数别滥用，一般 24h 内出结果）。

**和审核员沟通**：Resolution Center 可以文字回复申诉；表达"改了什么、在哪一版、如何验证"，比争辩条款更有用。

### 9.11 第 8 步：发布方式（过审 ≠ 立刻上架）

版本页"版本发布"三个选项：

- **手动发布（Manually release this version）**：过审后自己点"发布"——配合运营窗口期用；
- **过审自动发布（Automatically release）**：审核通过即上架；
- **定时自动发布**：指定日期上架。

**分阶段发布（Phased Release，推荐大版本用）**：7 天内按 1% → 2% → 5% → 10% → 20% → 50% → 100% 逐步放量，期间发现重大 bug 可**暂停放量**（已更新的用户不受影响），给修复争取时间。

> **上架后提醒**：App Store 搜索索引有数小时延迟；**版本回滚不存在**——线上出问题只能提新版，所以分阶段发布 + TestFlight 预演才更重要。

---

## 十、业务视角：JD（岗位描述，Job Description）为什么要求这些知识

| 知识点 | 支撑的业务场景 | 对应岗位画像 |
|---|---|---|
| Mach-O 结构、dyld 加载 | 启动优化（二进制重排 Binary Layout 优化、减少缺页中断 Page Fault）、包体积优化（App Thinning / 符号裁剪）、热修复（Hotfix） | 大厂平台组/框架组、电商 App |
| 静态/动态库、符号冲突 | SDK 接入排查、自研组件封装发布 | SDK 团队、中间件团队 |
| 证书、描述文件、重签名 | 多渠道出包（Channel Build）、企业内部分发、灰度测试（Beta Testing）、CI 打包自动化 | 互娱、金融、有发行业务的公司 |
| 签名校验、Entitlements | 安全加固（Hardening：反调试 Anti-Debug、防二次打包 Re-package） | 安全合规、金融类 App |
| dSYM / 符号表 | 线上崩溃治理（Crash Analysis：Bugly/Firebase 堆栈还原 Symbolication） | 所有有崩溃率 KPI 的团队 |

JD 写这条的真实含义：招一个**自己不被出包问题卡住、还能帮别人解卡**的人——知识体系完整（报错即定位），并且能写 CI 脚本、Strip 脚本、出包文档，把故障一次性消灭而不是每次救火。

> 游戏方向的岗位（Unity/Unreal 出包、渠道 SDK、引擎桥接）对应的业务与面试视角，见姊妹篇《iOS 游戏出包实战（引擎导出 Xcode 工程）》。

---

## 十一、面试口述钩子

**主线链路（背熟，能一口气讲下来）**：

> 逐 4KB 页 SHA256 哈希（Page Hash）→ 汇总成 CodeDirectory → 私钥签名（CodeDirectory + Entitlements）→ CMS 块写入 `__LINKEDIT` → 描述文件四要素（证书 / App ID / 设备 / 权限）内嵌为 `embedded.mobileprovision` → 装机时 installd 逐项校验。

**案例 STAR 结构（讲一个就够，同时证明知识+推动力）**：

- **S 现象（Situation）**：上传后收到 ITMS-90426 邮件（或 duplicate symbol 构建失败）；
- **T 定位（Task）**：`lipo -info` 确认 slice → 追到具体 SDK 的 framework；
- **A 行动（Action）**：推动 SDK 方发 xcframework（或跨团队协调删掉一份重复静态库）；
- **R 根治（Result）**：上架前 Strip 脚本进了 CI，此后同类问题零复发。

**备选踩坑案例**：
1. Duplicate Symbol 的跨团队推动过程；
2. 企业 Profile 过期导致已装用户集体打不开的应急处理；
3. 用 fastlane match 把"祖传打包机"改造成标准 CI 流水线。

---

## 附录：实操入口地图（新手遇到问题去哪里动手）

> 前文讲了原理，这一节回答"**具体在哪个界面点哪个按钮**"。新手排障 90% 的困难不是不懂原理，而是不知道去哪里操作。

### A. Apple Developer 网站（developer.apple.com）

入口：`developer.apple.com/account`，登录后进入 **Certificates, Identifiers & Profiles**（证书、标识符与描述文件管理后台，俗称"开发者中心"）。左侧分四栏，对应四类操作：

| 左侧栏 | 管什么 | 典型操作 |
|---|---|---|
| Certificates（证书） | 证书的创建/吊销/下载 | 见下方详细流程 |
| Identifiers（标识符） | App ID（Bundle ID）注册、Capability 开关 | 见下方详细流程 |
| Devices（设备） | 注册测试设备的 UDID | 见下方详细流程 |
| Profiles（描述文件） | 手动创建/重新生成 profile | 见下方详细流程 |

**高频操作步骤**：

1. **创建证书（Create Certificate）**：`Certificates → 点 + 号` → 选 `Apple Development` 或 `Apple Distribution` → 按提示上传 CSR 文件（CSR 在 Mac 的"钥匙串访问"里生成，见 C 节）→ 下载 `.cer` 文件 → 双击导入钥匙串（会自动和本机私钥配对）；
2. **注册 App ID（Register an Identifier）**：`Identifiers → + → App IDs → App` → 填 Description 和 Bundle ID → 选 `Explicit`（精确包名，商店 App 必选）或通配符 → 在下方列表**勾选需要的 Capability**（如 Push Notifications）→ Register；
3. **注册测试设备（Register a Device）**：`Devices → +` → 填设备名称和 UDID（UDID 获取：连 Mac 后在 Xcode `Window → Devices and Simulators` 里直接复制，或访达里点 iPhone"序列号"区域切换显示）→ Register。**注意：注册 UDID 后必须重新生成 profile 才会包含新设备**；
4. **手动创建描述文件（Create a Profile）**：`Profiles → +` → 选类型（`iOS App Development` 开发 / `Ad Hoc` 测试分发 / `In-House` 企业 / `App Store` 商店）→ 选 App ID → 选证书 → 选设备（商店类型无此步）→ 命名 → Generate → 下载；
5. **profile 过期/失效处理**：Profiles 列表里状态为 `Expired`/`Invalid` 的会标黄 → 点进去 Edit（重新勾选证书/设备）→ Generate 重新生成 → 重新下载并替换本地和 CI 里的旧文件；
6. **创建推送 Key（.p8，服务端推送用）**：`Keys → +` → 填名称 → 勾选 **Apple Push Notifications service (APNs)** → Configure → 下载 `.p8` 文件交给服务端。**Key 只能下载一次**，妥善保存；每个账号最多 2 个推送 Key（比管理各 App 的推送证书简单，服务端方案首选）。

### B. Xcode（出包和签名的主战场）

> 菜单名差异提示：Xcode 13+ 的设置入口是 `Xcode → Settings...`，旧版叫 `Xcode → Preferences...`（快捷键都是 `Cmd+,`）。

| 要做的事 | 位置 | 操作说明 |
|---|---|---|
| 登录 Apple 账号 | `Xcode → Settings → Accounts → 左下角 +` | 一切签名的前提；登录后 Team 自动出现 |
| 管理本机证书 | `Settings → Accounts → 选中账号 → Manage Certificates` | 右下角 + 可创建开发/分发证书；右键可删除（revoke 需去开发者中心） |
| 配置签名（最常用！） | 选中项目 → 选中 **target** → `Signing & Capabilities` 标签 | **自动签名**：勾 `Automatically manage signing` + 下拉选 Team，报错信息会直接显示在这块区域；**手动签名**：取消勾选 → 在 Signing Certificate 里选证书，下方 Provisioning Profile 选 profile |
| 添加 Capability（推送等） | `Signing & Capabilities → 左上角 + Capability` | 双击弹出面板，添加 Push Notifications / Associated Domains 等；Xcode 会自动改 `.entitlements` 文件并同步开发者中心的 App ID 配置 |
| 改 Bundle ID / 版本号 | target → `General → Identity` | Bundle Identifier、Version（对外版本号）、Build（内部构建号，每次上传必须+1） |
| 查看设备 UDID / 真机状态 | `Window → Devices and Simulators` | 真机调试、复制 UDID、看设备上的 App 都在这里 |
| framework 引入与签名方式 | target → `General → Frameworks, Libraries, and Embedded Content` | 每个第三方 framework 的 Embed 列必须正确：**动态库选 `Embed & Sign`，静态库选 `Do Not Embed`**（错一个就装机失败） |
| 排除模拟器架构 | target → `Build Settings → 搜 Excluded Architectures` | 对应 `EXCLUDED_ARCHS`，模拟器 slice 编译问题的临时方案 |
| Archive 出包 | ① Scheme 无需手动切配置（Archive 固定使用 Release）② 设备目标选 **Any iOS Device (arm64)** ③ `Product → Archive` | Archive 必须选真机目标，选了模拟器会灰掉或报错 |
| 导出 ipa（选分发方式） | Archive 完成自动弹出 Organizer → `Distribute App` | 选 `App Store Connect` / `Ad Hoc` / `Enterprise` / `Development` → 按向导导出；这里 Xcode 会自动做最终签名 |
| 查看历史 archive / dSYM | `Window → Organizer → Archives 标签` | 每个 archive 右键 `Show in Finder` 可拿到 `.xcarchive`（内含 dSYM） |
| 编译玄学问题第一步 | `Settings → Locations` 看 Derived Data 路径 → 手动删除 | 派生数据（Derived Data）缓存损坏会导致各种诡异编译错，先删再编 |

### C. macOS 系统工具

| 要做的事 | 位置 | 操作说明 |
|---|---|---|
| 生成 CSR 文件 | `钥匙串访问 Keychain Access → 证书助理 Certificate Assistant → 从证书颁发机构请求证书` | 填邮箱选"存储到磁盘"，生成的 `.certSigningRequest` 上传给开发者中心；**私钥此刻已生成在本机钥匙串里** |
| 导出 `.p12`（迁移证书+私钥） | 钥匙串访问 → 选中证书（展开能看到私钥）→ 右键"导出"→ 选 .p12 格式 | 设密码后发给同事/导入 CI；**只能由当初生成 CSR 的那台 Mac 导出** |
| 解开描述文件 | 终端：`security cms -D -i embedded.mobileprovision` | 对照第四节四要素排查 |
| 解开 ipa | 改后缀 `.zip` 解压，或 `unzip App.ipa` | 见第五节 |

### D. 新手排障"动手顺序"速查（收藏这张）

遇到签名/安装/上传问题，按顺序检查，90% 的问题能在这条链路上定位：

```
① Xcode → Signing & Capabilities
   （报错红字直接写原因：无证书 / Team 不对 / Bundle ID 不匹配 → 就地修复）

② Xcode → Settings → Accounts → Manage Certificates
   （证书是否缺失/过期/只有证书没私钥 → 缺私钥去要 .p12）

③ developer.apple.com → Profiles
   （profile 是否过期/设备是否包含/状态是否 Invalid → Edit 重新生成）

④ 终端解开 embedded.mobileprovision 对照四要素
   （security cms -D -i：过期时间/设备名单/证书/entitlements 逐项核对）

⑤ 仍失败 → Archive 后 Organizer 里 Distribute App 重新导出
   （排除旧包残留；上传后看 App Store Connect 邮件反馈）
```

> **重要习惯**：手动签名的团队，profile 更新后要**同时替换**本地 Xcode、CI 机器（match 仓库）两处的文件，只换一处就会出现"我本地好好的、CI 就是失败"的经典错位。

---

## 官方与参考链接（信息核验于 2026-11-19）

> 本文档的关键事实均以下列官方来源为准；政策类信息会变动，使用前建议重查原页面。

**账号与设备**

- Apple Developer Program 费用与权益：https://developer.apple.com/programs/enroll/
- 设备注册上限（每产品族 100 台/会员年）：https://developer.apple.com/help/account/devices/devices-overview
- WWDR 中间证书与轮换（2023-02-07 过期 / 新版至 2030-02-20）：https://developer.apple.com/help/account/certificates/wwdr-intermediate-certificates/

**上架与审核**

- App Store 审核指南（Guideline 条款全文）：https://developer.apple.com/app-store/review/guidelines/
- 账号删除要求（5.1.1(v)，2022-06-30 生效）：https://developer.apple.com/news/?id=12m75xbj 与 https://developer.apple.com/support/offering-account-deletion-in-your-app/
- 中国大陆 ICP 备案（App 信息页要求）：https://developer.apple.com/cn/help/app-store-connect/reference/app-information/app-information
- 提审状态（含"缺少 ICP 备案号 / 缺少游戏注册编号"）：https://developer.apple.com/help/app-store-connect/reference/app-information/app-and-submission-statuses
- TestFlight 构建 90 天有效期：https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview
- 分阶段发布等 App Store Connect 操作：https://developer.apple.com/help/app-store-connect/

**构建与合规**

- Xcode 14 Release Notes（Bitcode 弃用）：https://developer.apple.com/documentation/xcode-release-notes/xcode-14-release-notes
- 隐私清单与 ITMS-910xx 排查（TN3181，2024-11-12 起强制）：https://developer.apple.com/documentation/technotes/tn3181-debugging-invalid-privacy-manifest
- 隐私影响型第三方 SDK 清单：https://developer.apple.com/support/third-party-SDK-requirements/
