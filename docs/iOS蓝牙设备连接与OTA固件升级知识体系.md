# iOS 蓝牙设备连接与 OTA 固件升级知识体系（新手友好版）

> 覆盖：蓝牙（BLE）连接后的工程实践 / 断开与重连状态机 / 指数退避 / OTA 固件升级全流程 / 与硬件配合的协议约定与时序 / Nordic DFU 库集成 / 升级后连不上的四层排查 / 最小可跑 DFU 升级 SOP / 排障与速查
> 阅读方式：每个概念先讲"是什么（带比喻）"，再讲"为什么需要它"，最后讲"怎么用 / 怎么排查"。专业名词首次出现均做中英对照，并给一句大白话。
> 信息核验日期：2026-11-25。动态信息（库版本、工具下载入口）以官方页面为准，文末附链接。
> 姊妹篇：广播、扫描、系统唤醒与后台状态恢复，见《iOS 数字车钥匙与蓝牙远程唤醒知识体系》。

---

## 目录

- [0. 先用一组比喻建立全局](#0-先用一组比喻建立全局)
- [一、背景：连接之后才是主战场](#一背景连接之后才是主战场)
- [二、断开与重连：用状态机管住意外](#二断开与重连用状态机管住意外)
- [三、OTA 固件升级是什么](#三ota-固件升级是什么)
- [四、与硬件配合：约定与时序](#四与硬件配合约定与时序)
- [五、现成方案：Nordic DFU 库](#五现成方案nordic-dfu-库)
- [六、实操：最小可跑 DFU 升级（SOP）](#六实操最小可跑-dfu-升级sop)
- [七、常见报错与排查](#七常见报错与排查)
- [八、高频速查表](#八高频速查表)
- [九、踩坑清单](#九踩坑清单)
- [附录：实操入口地图](#附录实操入口地图)
- [官方与参考链接](#官方与参考链接)

---

## 0. 先用一组比喻建立全局

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 固件 | Firmware | 设备大脑里装的程序 | OTA 要更新的对象 |
| 空中升级 | OTA（Over-The-Air） | 给设备寄"系统更新包裹" | 不接线，无线更新固件 |
| 升级模式 | DFU（Device Firmware Update） | 设备的"装修模式" | 只收固件、不干业务的状态 |
| 引导程序 | Bootloader | 出厂焊死的"BIOS" | 决定从哪个分区启动，永不升级 |
| A/B 双分区 | A/B Partition | 新旧两套装修方案 | 新固件刷进 B 区，失败回退 A 区 |
| 数据包 / 帧 | Packet / Frame | 快递一件一件送 | 数据按小包收发、逐包确认 |
| 校验 | CRC（Cyclic Redundancy Check） | 收货前核对装箱单 | 验证数据在传输中有没有出错 |
| 确认 | ACK（Acknowledgement） | 签收回执 | 设备确认收到一包数据 |
| 单包上限 | MTU（Maximum Transmission Unit） | 一辆车一趟能装多少货 | 决定单个包最多带多少字节 |
| 断连重连 | Reconnection | 快递员跑单后重新接单 | 按策略恢复连接，而不是重头再来 |
| 指数退避 | Exponential Backoff | 失败后翻倍等一会儿再试 | 防止重连风暴拖垮电量和链路 |
| 连接状态机 | State Machine | 交通信号灯 | 让连接流程只走合法的几条路 |
| 服务缓存 | Service Cache | 快递员的旧地址本 | iOS 记着设备的服务表，固件变了要对账 |
| 服务与特征 | Service / Characteristic | 设备的"服务项目表"和每个项目下的"数据项" | BLE 数据读写的组织方式 |
| 初始化包 | Init Packet | 固件的"身份证" | 写明适配的型号、版本、签名与校验值 |
| 协议栈 | SoftDevice | Nordic 芯片出厂自带的蓝牙底座 | 芯片里负责蓝牙底层通信的固件 |
| 无按钮升级 | Buttonless DFU | 没有按键也能被 App 指挥进"装修模式" | 设备在应用运行时远程触发升级模式 |
| 连接间隔 | Connection Interval | 连上后双方多久聊一次天 | 间隔越大越省电，但数据来得越慢 |

串起来就是一句话：**连上设备只是开始——数据按包确认、断开按状态机重连、固件靠 OTA 更新，三件事全靠"约定、校验、确认"三个词撑起来。**

---

## 一、背景：连接之后才是主战场

### 1.1 从扫描到能干活，中间有五步

以 OBD 诊断 App 为例：手机连接诊断盒子、读出故障码，完整链路是五步。

| 步骤 | App 侧动作 | 用到的 API |
|---|---|---|
| ① 扫描 | 按服务 UUID / 广播名过滤，发现设备 | `scanForPeripherals(withServices:)` |
| ② 连接 | 发起连接，等待回调 | `connect(_:options:)` |
| ③ 发现服务 | 列出设备提供的服务和特征 | `discoverServices` / `discoverCharacteristics` |
| ④ 订阅与读写 | 订阅 notify（设备有新数据时主动推给 App 的机制）、发诊断指令 | `setNotifyValue` / `writeValue` |
| ⑤ 业务交互 | 解析数据帧、更新界面、跑状态机 | 自研协议层 |

广播和唤醒链路的细节（判据、后台限制、系统状态恢复）不在本文展开，见姊妹篇。

### 1.2 一条总根源事实：断开后，一切失效

苹果官方文档对断开回调的描述里有一句容易被忽略的话：连接断开后，设备的所有服务、特征、描述符都会失效（invalidated）。翻译成大白话：**每次断连，快递员的地址本就作废了**。重连成功不代表一切照旧——服务要重新发现、通知要重新订阅，这决定了后面所有重连代码的写法。

### 1.3 本文范围

本文只讲连接建立之后的工程实践：链路断了怎么办（第二章）、设备固件怎么升级（第三、四、五章）、怎么动手跑通一次（第六章）、出问题怎么排查（第七章）。示例语境是 OBD 诊断盒，机制对智能硬件、物联网设备通用。

---

## 二、断开与重连：用状态机管住意外

### 2.1 断开是常态，不是异常

设备会断电、车主会走远、手机系统会因为内存压力回收资源。诊断设备插在车上，车辆熄火休眠时蓝牙链路就会断。把断连当成 bug 修，代码里很快会长出一堆互相打架的布尔标志；当成常态设计，才是正确的起点。

### 2.2 分清两种断开：主动和意外

断开回调是 `centralManager(_:didDisconnectPeripheral:error:)`。`error` 参数的官方语义：

- **`error == nil`**：断开是 App 自己调 `cancelPeripheralConnection(_:)` 主动发起的，不需要重连。
- **`error != nil`**：意外断开（设备关机、超出范围、链路丢失），按重连策略处理。

这一行判断是整个重连逻辑的第一道岔路口。

### 2.3 断开后要做的四件事

无论哪种断开，都要按顺序收尾：

1. **切状态**：状态机从 `connected` 切到 `reconnecting` 或 `disconnected`，UI 同步提示（比如顶部横幅"连接已断开"）。
2. **清现场**：停掉轮询任务、取消实时数据订阅、清空数据拼装缓冲区。残留的半包数据会让重连后的每一帧都校验失败。
3. **决定是否重连**：主动断开不重连；意外断开按 2.5 的退避策略重试。
4. **UI 回到安全态**：数据页显示"最后值 + 时间"，别让旧数据假装还在实时更新。

### 2.4 重连三招与"重新开始"

苹果的重连最佳实践给了三条路径，按成本从低到高：

1. `retrieveConnectedPeripherals(withServices:)`——设备可能还连在系统上，先查这个再连；
2. `retrievePeripherals(withIdentifiers:)`——用登记过的设备标识直接找，省去扫描；
3. 重新扫描广播。

重连成功之后，还有两件事必须做：**重新发现服务**（`discoverServices`）、**重新订阅通知**（`setNotifyValue`）。这正是 1.2 那条事实的直接推论——重连后不重新订阅，是"收不到数据却以为设备坏了"的第一大原因。

### 2.5 指数退避与最大次数

意外断开后立刻重连、失败了立刻再连，就是重连风暴：设备真掉线时 App 耗电刷屏，链路里塞满注定失败的重试。指数退避的思路是每次失败后把等待时间翻倍。

```swift
func centralManager(_ central: CBCentralManager,
                    didDisconnectPeripheral peripheral: CBPeripheral,
                    error: Error?) {
    guard error != nil else { return }          // 主动断开，不重连
    state = .reconnecting                        // 状态机切换，UI 同步
    retryCount += 1
    guard retryCount <= maxRetry else {          // 次数用完，终止并上报
        state = .error
        return
    }
    let delay = Double(1 << min(retryCount, 5))  // 1s、2s、4s、8s……封顶 32s
    DispatchQueue.main.asyncAfter(deadline: .now() + delay) {
        // iOS 会缓存 CBPeripheral 对象，直接 connect，不必重新扫描
        central.connect(peripheral)
    }
}
```

### 2.6 连接等待要自己计时

`connect` 发出后，等待时长由系统掌控：设备不在广播时，这个请求可能长时间挂着，不保证及时回调。虽然系统侧存在连接超时的错误码（见 8.1），但工程上不能赌它必然触发——**自己起一个计时器（比如 10 秒），超时后调 `cancelPeripheralConnection` 并转入失败态**，才是可控的做法。

### 2.7 连接状态机速查

| 状态 | 含义 | 允许迁移到 |
|---|---|---|
| `disconnected` | 待机，没有待处理的连接请求 | `connecting` |
| `connecting` | `connect` 已发出，等待回调 | `connected` / `disconnected`（超时或失败） |
| `connected` | 链路在、服务已发现、订阅已生效 | `reconnecting` / `disconnected` |
| `reconnecting` | 意外断开，退避重试中 | `connected` / `error` |
| `error` | 重试超限、权限异常等不可用态 | `disconnected`（用户手动重试） |

纪律：状态变化收敛到一个入口函数，非法迁移直接断言并打日志。散落在各处的 `isConnected` 布尔标志，是这类模块腐化的开端。

---

## 三、OTA 固件升级是什么

### 3.1 是什么

OTA（Over-The-Air，空中升级）指不插数据线，通过无线链路把新固件传给设备，设备自己完成刷写。你的 iPhone 有"系统更新"按钮；诊断盒没有屏幕也没有应用商店，**App 就兼任它的"应用商店 + 快递员"**：从云端下载新固件，用 BLE 一包一包喂给设备，设备进入升级模式自己刷写重启。

### 3.2 为什么需要

硬件卖出去就收不回来了。诊断盒每年要支持新车型、新协议，只能靠 OTA 让用户手里的存量设备"永葆青春"。对厂商是修复成本最低的手段，对用户是"设备越用越新"的体验来源。

### 3.3 升级五步流程

| 步骤 | 谁负责 | 做什么 |
|---|---|---|
| ① 版本检查 | App | 读设备的版本号特征值，与云端最新版本比对，有新版提示用户 |
| ② 下载固件包 | App | 从后端下载固件文件（zip / bin），校验完整性（MD5——给文件算的"指纹"，两边指纹一致说明文件没坏） |
| ③ 分包传输 | App ↔ 设备 | 固件切成小包逐包发送，设备逐包 ACK，App 显示进度条 |
| ④ 刷写 | 设备 | 进入 bootloader 自刷固件，断连重启——硬件自主，App 只等待 |
| ⑤ 重启验证 | App | 重连后读新版本号比对，确认升级成功 |

### 3.4 帧与包：分包传输的基础

固件是几 MB 的文件，BLE 单次只能写很小的数据块。传输前要切包，切多大由 MTU 决定：BLE 规范里 ATT 层默认 MTU 是 23 字节，iOS 连接时会自动协商更大的值；**实际单包上限以 `peripheral.maximumWriteValueLength(for:)` 的返回为准**，别硬编码。

一帧升级数据通常长这样（具体以硬件协议文档为准）：

```
| 帧头 | 长度 | 序号 | 数据 …… | CRC |
```

- **序号**：设备用它确认"收到的是第几包"，是断点续传的依据；
- **CRC**：设备用它确认"这包没传坏"，传坏了回 NACK（Negative Acknowledgement，否认回执），App 重发同一包。

发送侧还有一个选择：`writeValue(_:for:type:)` 的 type 用 `.withResponse`（每包等回执，可靠但慢）还是 `.withoutResponse`（不等回执，快但要有序号和重传机制兜底）。OTA 传输量大的场景通常是"连续写多包 + 窗口式确认"。

---

## 四、与硬件配合：约定与时序

### 4.1 升级前，双方要约定什么

OTA 不是 App 单方面发数据，而是 App 和设备按一份事先约定好的剧本一问一答。硬件厂商会把这份剧本写成升级协议文档，拿到文档后先确认五件事都在：

| 约定项 | 内容 |
|---|---|
| 通道 | 用哪个服务 / 特征收发升级数据（通常是一个专用 DFU 服务） |
| 包结构 | 序号 + 长度 + 数据 + CRC 的帧格式 |
| 确认机制 | 逐包 ACK 还是攒一批窗口确认；NACK 表示这包错了要重发 |
| 头信息 | 开传前先报总包数、总大小、固件版本、MD5，设备预留存储 |
| 状态指令 | "进入升级模式"、"传输完成"、"重启刷写"这些指令的格式 |

### 4.2 七步对答时序

App 指挥节奏，设备只做应答；到了刷写阶段反过来，设备自主干活，App 只等待。

```
iOS App                    设备（固件）
   |                           |
   |-- ① 进入升级模式 --------->|
   |<-- 回 ready --------------|
   |-- ② 固件头信息 ----------->|   总包数 · 总大小 · MD5
   |<-- 回 OK，预留存储 --------|
   |-- ③ 数据包 #1 ----------->|
   |<-- ACK #1 ----------------|
   |      ……循环到 #N……        |   NACK / 超时则重发同一包
   |-- ④ 传输完成 ------------->|   设备做全量 CRC 校验
   |<-- 回校验通过 -------------|
   |-- ⑤ 重启刷写指令 --------->|
   |                           |   设备切 bootloader，自刷，断连重启
   |-- ⑥ 轮询重连 ------------->|
   |<-- 新版本号 --------------|   ⑦ 比对验证升级成功
```

对新手最重要的一句：**每包数据都要有"签收回执"（ACK）**。发送进度用"已 ACK 包数 / 总包数"计算——发出去不等于收到了。

### 4.3 iOS 侧的工程细节

- **状态机驱动**：`待机 → 校验版本 → 传输中 → 校验中 → 等待重启 → 重连验证 → 完成/失败`，每个阶段对应不同 UI，进度条文案跟着变。
- **每包超时重传**：ACK 等待一般 300ms–1s；超时重发同一包，连续失败 3 次就终止流程，别无限重试。
- **断点续传**：记住设备最后 ACK 的序号，断线重连后从下一包继续，不必从头传。
- **防呆**：传输中锁住页面操作，退出弹确认框。开传前过一遍检查清单：手机电量充足、蓝牙开启；**设备电量低于硬件协议规定的阈值就拒绝发起**（刷写中途掉电是最常见的升级失败诱因）；固件包 MD5 校验通过（失败先重新下载，仍失败查分发缓存是不是发了旧包）；确认设备存储空间足够——它对应系统错误码里的 `outOfSpace`（见 8.1）。
- **提速**：协商大 MTU、`.withoutResponse` 连续写多包（带序号）+ 窗口确认，而不是每包都等回执。

---

## 五、现成方案：Nordic DFU 库

### 5.1 什么时候用它

如果设备用的是 Nordic 芯片方案（nRF51 / nRF52 系列，带 SoftDevice 和 DFU Bootloader），不必手写第四章的整套协议，直接用官方库 NordicDFU。注意边界：**它是为 nRF5 SDK 设备设计的，不支持 nRF Connect SDK（NCS）/ MCUboot 方案**——那类设备要用 nRF Connect Device Manager 库。接手项目时先问清设备用哪套 SDK，再选库。

### 5.2 两个官方要求，提前知道少踩坑

- **设备必须实现 Service Changed 特征**（Generic Attribute 服务里，带 Indicate 属性）。没有它，iOS 会认为这台设备的服务永远不会变，切换到升级模式后不刷新服务缓存，升级后经常"连上了却找不到特征"。接硬件时先核对固件有没有配这个。
- **升级模式的广播会变样**：部分 Secure DFU 方案里，bootloader 会用"MAC 地址 +1"对外广播，手机看到的是一台"新设备"。所以扫描过滤要放宽，别死守原设备名。

### 5.3 集成与最小代码

三步集成：Podfile 写 `pod 'NordicDFU'` → 执行 `pod install` → 代码里 `import NordicDFU`。库也支持 SPM 和 Carthage。

```swift
import NordicDFU
import CoreBluetooth

final class OtaManager: NSObject, DFUServiceDelegate, DFUProgressDelegate {
    private var controller: DFUServiceController?

    /// target 是已连接的 CBPeripheral，zipURL 是固件包的本地路径
    func startUpgrade(peripheral: CBPeripheral, zipURL: URL) throws {
        let firmware = try DFUFirmware(urlToZipFile: zipURL)
        let initiator = DFUServiceInitiator()
        initiator.logger = self             // 日志：联调时与硬件对账
        initiator.delegate = self           // 状态与错误
        initiator.progressDelegate = self   // 进度
        controller = initiator.with(firmware: firmware).start(target: peripheral)
    }

    func dfuStateDidChange(to state: DFUState) {
        // .connecting → .starting → .enablingDfuMode → .uploading → .validating → .completed
        // 注意：.validating 仅 Legacy DFU 出现，Secure DFU 的校验由 bootloader 完成
    }

    func dfuError(_ error: DFUError, didOccurWithMessage message: String?) {
        // 转入失败态并上报；对照 7.6 的 DFUError 速查表
    }
}
```

`DFUServiceController` 还提供暂停、恢复、中止三个操作。进度回调里能拿到百分比和分片信息，直接绑定进度条。库默认用窗口确认（每 12 包回执一次），这正是 4.3 说的提速机制，参数可通过 `packetReceiptNotificationParameter` 调整。

---

## 六、实操：最小可跑 DFU 升级（SOP）

> 这一节是执行线，前面章节是知识点。照着做，可以跑通"手机给 Nordic 设备无线升级固件"。

### 6.0 全景步骤总览

```
第 0 步 前置准备（Mac / 真机 / Nordic 开发板 / 固件 zip 包）
  → 第 1 步 不写代码，用 nRF Connect 手动升级一遍
  → 第 2 步 集成 NordicDFU
  → 第 3 步 写最小 DFU 代码
  → 第 4 步 边界场景测试
  → 第 5 步 重启验证与验收
```

### 6.1 第 0 步：前置准备

| 准备项 | 要求 | 说明 |
|---|---|---|
| Mac | 能装当前 Xcode | Xcode 从 App Store 装 |
| iPhone | 真机，支持 BLE | 模拟器没有蓝牙硬件，一切 BLE 功能必须真机 |
| BLE 设备 | Nordic 开发板（nRF52 DK 等）或量产设备 | 固件须带 DFU Bootloader，见 5.1 |
| 固件包 | 硬件同事提供的 `.zip` DFU 包 | 内含新固件 + init packet 校验信息 |
| 工具 | nRF Connect（App Store） | Nordic 官方 App，自带 DFU 与日志 |
| 工程配置 | Info.plist 声明蓝牙权限 | 加 `NSBluetoothAlwaysUsageDescription` 写真实用途，否则一调蓝牙 API 直接崩（文末附官方链接） |
| 测试条件 | 无硬件条件时先向硬件同事要 | 开发板烧录 DFU Bootloader 由硬件侧完成，参考 Nordic 官方文档站（文末链接） |

### 6.2 第 1 步：不写代码，先手动升级一遍

先证明设备端没问题，再写 App 代码。先把固件 zip 传到手机（AirDrop 或"文件"App 都行）。打开 nRF Connect → **Scanner** 找到设备并连接 → 进入已连接界面 → 点顶部工具栏的 **DFU** 图标 → 选择固件 zip → **Upload** → 等待进度走完、设备重启。

这一步通过了，说明设备的 bootloader、固件包、升级服务都是好的；失败了先找硬件同事，别急着怀疑自己的代码。

### 6.3 第 2 步：集成 NordicDFU

Podfile 里加一行 `pod 'NordicDFU'` → 终端执行 `pod install` → 之后代码用 `import NordicDFU` 引入。见 5.3。

### 6.4 第 3 步：写最小 DFU 代码

如果工程里还没有"扫描并连接"这一段，先按 1.1 的五步链路把它补齐——本文档聚焦连接之后的升级环节。把 5.3 的 `OtaManager` 贴进工程，在设备连接成功后调用 `startUpgrade`。日志回调（`DFULoggerDelegate`）务必接上——联调时 App 日志和设备日志对时间戳，是定位丢包最有效的手段。

### 6.5 第 4 步：边界场景测试

每个场景都要造出来跑一遍：

| 场景 | 怎么造 | 预期 |
|---|---|---|
| 传输中断蓝牙 | 手机设置里关蓝牙再开 | 库进入错误态，App 可重试 |
| 设备中途断电 | 传输到 50% 拔掉设备 | App 报断连；设备重启后停在应用模式，可重新升级 |
| 升级中杀 App | 传输中上划强杀 | 设备侧仍在 bootloader 等待；重开 App 扫 DFU 广播恢复 |
| 升级中来电 | 打电话打断 | App 保持失败态，不展示错误的新版本 |

### 6.6 第 5 步：重启验证与验收

设备重启后重连，读版本号特征值，与升级目标版本比对。**验收标准不是"进度条走完"，而是"新版本号读出来了 + 业务指令正常"**。把这条用例放进回归组，固件每次改版全量跑一遍。

---

## 七、常见报错与排查

> 共同规律：连接类问题看回调链，传输类问题看确认回执，升级类问题看广播名和版本号。

### 7.1 connect 发出去，一直没回调

**现象**：`connect` 调了，`didConnect` / `didFail` 都迟迟不来，界面停在"连接中"。

**原理（新手版）**：连接请求由系统代管，设备不在广播时请求会长时间挂着；系统侧虽有连接超时错误码，但不能赌它必然触发。

**怎么办**：
1. 发起连接的同时自己起计时器（10 秒左右）。
2. 超时后调 `cancelPeripheralConnection(_:)`，转入失败态并允许重试。
3. 检查设备是否被别的手机占用（BLE 通常单连接），或换台手机交叉测试。

**预防**：把"连接超时计时器"做进连接管理器，所有连接都走它。

### 7.2 写数据丢包或失败

**现象**：诊断指令时灵时不灵；批量写入时设备收到的数据缺斤少两。

**原理（新手版）**：单包超过了设备实际支持的大小（MTU 协商结果比预期小），或 `.withoutResponse` 写太快，设备的接收缓冲跟不上。

**怎么办**：
1. 写之前用 `maximumWriteValueLength(for:)` 确认单包上限，按它切包。
2. `.withoutResponse` 模式下加序号，配合 `peripheralIsReady(toSendWriteWithoutResponse:)` 回调节流发送。
3. 还失败就把 type 换成 `.withResponse` 验证，再逐步提速。

**预防**：把切包逻辑收进一个传输模块，别在业务代码里到处拼字节。

### 7.3 订阅了 notify，却收不到数据

**现象**：连接正常，写指令有反应，但实时数据一条都收不到。

**原理（新手版）**：要么订阅本身没成功就发了指令，要么这是重连之后——断开后所有订阅失效（见 1.2），重连不重新订阅自然没数据。

**怎么办**：
1. 确认 `peripheral(_:didUpdateNotificationStateFor:error:)` 回调无错误后，再发业务指令。
2. 重连流程里固定包含：`discoverServices` → `discoverCharacteristics` → `setNotifyValue`，一步不省。
3. 检查特征是否真的支持 notify 属性。

**预防**：把"发现服务 + 订阅"封装成一个 `setupSession` 流程，首次连接和重连都调它。

### 7.4 升级后连不上（重点）

**现象**：OTA 进度走完、设备重启，之后 App 扫不到设备，或连上了找不到服务。

**原理（新手版）**：升级改变了设备的"运行形态"——它可能停在 bootloader 升级模式，广播名变成 DFU 前缀甚至换了地址（MAC +1）；也可能设备固件的服务表变了，而 iOS 还拿着旧的服务缓存对不上账。

**怎么办**：
1. **先交叉验证，分清是谁的问题**：用 nRF Connect 或 LightBlue 直接扫设备。第三方工具能看到设备 → 查 App 代码；看不到 → 查设备与固件。
2. **按四层排查**（见 8.3 的表）：广播层查广播名是否变了，放宽过滤条件，试试搜 DFU 前缀；连接层确认有超时兜底；服务层把 `didDiscoverServices` 打印的服务表与新固件协议文档逐条对照；数据层读版本号判断固件状态。
3. **确认固件包与设备匹配**：bootloader 拒刷最常见的原因是"包不对"。init packet 里的 device type、device revision、application version 和兼容 SoftDevice 版本必须与目标设备硬件一一对应（SDK 7+ 强制要求）；给 A 型号设备刷了 B 型号的包、或刷了被版本回退保护拦截的旧包，设备都会在校验阶段拒绝。把包的参数与设备型号逐项核对，拒刷错误对照 7.6 的速查表（remoteExtendedError 系列）。
4. **核对 Service Changed 特征**：设备没实现它（见 5.2）时，配对过的设备升级后 iOS 不会自动刷新服务缓存，表现为"连上了、服务是旧的"。让硬件同事在固件里补上，或引导用户删除配对后重连。
5. **走恢复流程**：设备停在 bootloader 时，App 扫描 DFU 广播、重新走一遍 DFU 把可用固件刷回去。产品上要留这个入口，并配"设备正在恢复模式"的明确提示。
6. **疑难杂症抓底层**：用 PacketLogger（Apple 官方蓝牙抓包工具，随 Additional Tools for Xcode 分发）看 HCI 层的连接失败原因码——很多 App 层看不到的错误在底层一目了然。

**预防**：升级前后都读版本号留档；升级日志全量记录并支持线上上报；扫描过滤同时匹配"应用模式名"和"升级模式名"；固件包按设备型号归档，包名带型号与版本标识。

### 7.5 升级中断，会不会把设备刷成砖

**现象**：传输到一半断电、断蓝牙，担心设备废了。

**原理（新手版）**：好的升级机制本身防砖——bootloader 是出厂写死、永不升级的"引导管理员"，任何固件都动不了它；很多方案还有 A/B 双分区，新固件刷进 B 区，校验通过才切换启动，失败仍从 A 区启动旧版本。

**怎么办**：设备真停在 bootloader（表现为只广播 DFU 名）时，用 7.4 的恢复流程重新刷固件即可，这不算砖。

**预防**：选型时把"bootloader 独立 + 失败回退"列为硬件方案的硬性要求；无按钮设备确认固件带 buttonless DFU 服务（SDK 13/14 起正式支持；SDK 12 的实验版本默认关闭且不安全，别用）。

### 7.6 升级中途失败：按 DFU 状态机定位（重点）

**现象**：进度条卡在某个百分比不动，或中途报错退出，不知道从哪查起。

**原理（新手版）**：DFU 库自己就是一台状态机，失败时先看"停在哪一态"，每个状态对应一类凶手。NordicDFU 的完整状态流转是 `.connecting → .connected → .starting → .enablingDfuMode → .uploading → .disconnecting → .completed`，每次跳变都会进 `dfuStateDidChange(to:)` 回调（见 5.3）。官方文档特别注明：`.validating`（设备校验固件）只在 Legacy DFU 出现，Secure DFU 的校验由 bootloader 在收尾时完成。

**怎么办**（按卡住的态对号入座）：

| 卡在哪一态 | 这一步在干什么 | 最可能的凶手 | 手段 |
|---|---|---|---|
| `.connecting` | 连接 DFU 目标 | 广播过滤没匹配升级模式名；设备被占用 | 见 8.3 广播层、连接层 |
| `.starting` | 初始化 DFU 操作：发现服务、读版本、订阅控制点 | 设备没实现 DFU 服务；服务缓存是旧的（缺 Service Changed，见 5.2） | 打印服务表对照；删配对重连；确认固件带 DFU Bootloader |
| `.enablingDfuMode` | 切入 bootloader 升级模式 | buttonless 前置条件不满足（未配对、固件没实现）；设备拒切 | 核对 7.5 的 buttonless 要求；抓设备侧 RTT 日志对账 |
| `.uploading` | 逐包上传固件 | 链路差丢包、重传循环；窗口确认参数过大 | 靠近设备；调小 `packetReceiptNotificationParameter`（默认 12） |
| `.validating`（仅 Legacy DFU） | 设备校验固件 | 包 CRC 不对；init packet 与设备不匹配（见 7.4 第 3 条） | 核对包的型号与版本；重新下载 |

**进度卡在固定百分比不动**：多半是确认回执丢了——App 等不到窗口确认，或固件把连接间隔设得过大。先靠近设备排除链路问题，再让硬件核对连接间隔参数，最后调小窗口确认包数。

**DFU 层常见错误速查**（NordicDFU 的 `DFUError`，经 `dfuError(_:didOccurWithMessage:)` 回调上报；注意与 8.1 的系统层 CBError 是两套，别混）：

| 错误码 | 官方含义 | 常见场景 |
|---|---|---|
| `bluetoothDisabled` | 蓝牙适配器未开启 | 手机蓝牙被关了 |
| `failedToConnect` / `deviceDisconnected` | 连不上 DFU 目标 / 目标意外断开 | 传输中途走出范围、设备掉电 |
| `deviceNotSupported` | 设备不支持 Legacy / Secure DFU 或 Buttonless 服务 | 固件没带 DFU Bootloader，或选错了库（见 5.1） |
| `readingVersionFailed` | 读 DFU 版本特征失败 | DFU 服务结构异常 |
| `enablingControlPointFailed` | 订阅控制点通知失败 | 服务缓存旧或特征权限不符 |
| `remoteButtonlessDFUNotBonded` | Buttonless 服务要求设备已配对 | 未配对就发起无按钮升级 |
| `remoteButtonlessDFUOperationFailed` | 跳转 bootloader 失败 | 设备拒绝切升级模式，查固件配置 |
| `bytesLost` | 已发送字节数与确认回执不一致 | 传输丢包，链路质量问题 |
| `crcError` | 设备回报 CRC 不匹配，重试 3 次仍失败 | 干扰严重，靠近设备再试 |
| `initPacketRequired` / `extendedInitPacketRequired` | 缺少 init packet | 固件包不完整（SDK 7+ 必需） |
| `remoteExtendedErrorHwVersionFailure` | 硬件版本与升级要求不匹配 | **包刷错型号了** |
| `remoteExtendedErrorFwVersionFailure` | 固件版本过低 | 回退保护生效，旧版本包被拒 |
| `remoteExtendedErrorSdVersionFailure` | 兼容的 SoftDevice 列表不含当前设备 | 包与设备底层软件不兼容 |
| `remoteExtendedErrorInsufficientSpace` | 设备空间不足 | 固件太大，对应系统层的 `outOfSpace` |
| `remoteExtendedErrorVerificationFailed` | 收到的固件哈希与 init packet 不符 | 传输损坏或包被改过，重新下载 |
| `fileInvalid` | 固件文件不受支持 | 拿错文件或格式用错入口 |

"拒刷"类错误的最终原因只有设备侧知道。让硬件同事用 J-Link RTT（Real Time Transfer，SEGGER 调试器往电脑实时输出设备日志的通道，见 8.4）抓 bootloader 日志，与 App 日志按时间戳对账：App 记"发了哪包、停在哪个态"，设备记"收到哪包、拒在哪一步"，两边一对，丢包和拒刷立刻现形。

**预防**：升级失败上报必须带上"DFU 状态 + DFUError + 卡住时的进度"，只报一句"升级失败"没法排查；升级前的电量、存储、包完整性检查按 4.3 的清单执行。

---

## 八、高频速查表

### 8.1 CBError.Code 常见错误码

系统侧错误在 `CBError` 域；设备侧返回的 ATT 层错误在 `CBATTError` 域，两者别混。

| 错误码 | 官方含义 | 常见场景 |
|---|---|---|
| `peripheralDisconnected` | 外设已断开 | 断连回调里最常见 |
| `notConnected` | 设备当前未连接 | 断开后仍去读写 |
| `connectionTimeout` | 连接超时 | 系统侧连接尝试超时 |
| `connectionFailed` | 连接失败 | 设备不在广播或被占用 |
| `connectionLimitReached` | 已达最大连接数 | 设备被别的手机占着 |
| `outOfSpace` | 设备空间不足 | OTA 写入阶段设备存储不够 |
| `operationCancelled` | 操作被取消 | 自己调用 cancel 后收到 |
| `invalidHandle` | 属性句柄无效 | 拿着断连前的旧特征引用操作 |
| `invalidParameters` | 参数无效 | UUID 或参数配置有问题 |
| `peerRemovedPairingInformation` | 对端删除了配对信息 | 设备清掉了配对记录 |

### 8.2 断连处理四件事

```
① 切状态：connected → reconnecting / disconnected，UI 同步提示
② 清现场：停任务、取消订阅、清拼帧缓冲区
③ 定策略：主动断开不重连；意外断开指数退避，封顶 32s，最多 N 次
④ 回安全态：数据页展示"最后值 + 时间"，不冒充实时
```

### 8.3 升级后连不上的四层排查

| 层 | 现象 | 凶手 | 手段 |
|---|---|---|---|
| 广播层 | 扫不到设备 | 固件停在 DFU 模式：广播名变 DFU 前缀或 MAC +1；过滤条件太死 | 放宽过滤搜 DFU 名；nRF Connect 交叉验证 |
| 连接层 | 有广播，connect 没回调 | 请求挂着等；设备被占用 | 自己计时，超时 cancel 重扫 |
| 服务层 | 连上了，找不到服务 | iOS 服务缓存未失效（缺 Service Changed 特征）；服务表变了 | 打印服务表对照新固件协议；删配对重连 |
| 数据层 | 数据不对、版本读不出 | 升级只成功一半，停在 bootloader | 读版本号判断；走恢复流程重新 DFU |

### 8.4 工具速查

| 工具 | 来源 | 用途 |
|---|---|---|
| nRF Connect | Nordic 官方，App Store | 手动 DFU、扫广播、看服务表，含日志 |
| LightBlue | Punch Through，App Store | 交叉验证广播与服务结构 |
| PacketLogger | Apple 官网，Additional Tools for Xcode（Hardware 目录） | HCI 层抓包，看 App 层看不到的失败原因码 |
| J-Link RTT Viewer | SEGGER 官网，随 J-Link 软件包分发 | 看设备侧日志（bootloader / 固件的 RTT 输出），与 App 日志对账 |
| Console.app | macOS 自带 | 看系统蓝牙守护进程日志 |

---

## 九、踩坑清单

1. 把断连当 bug 修，用一堆布尔标志代替状态机——标志迟早互相打架。
2. 断连后不清现场：旧的轮询任务还挂着，重连后回调错乱。
3. 重连成功后不重新发现服务、不重新订阅 notify——收不到数据还以为设备坏了。
4. 赌系统会给 connect 超时回调——请求可能无限期挂着，必须自己计时。
5. 意外断开后无限重连——没有指数退避和次数上限，设备不在时 App 耗电刷屏。
6. 进度条按"已发送包数"算——发出去不等于收到，要按已 ACK 的包数。
7. 升级中允许用户跳页、杀 App——升级断在半路，返工的是自己。
8. 设备固件没实现 Service Changed 特征——升级后 iOS 还用旧服务缓存，连上找不到特征。
9. 下载的固件包不校验 MD5 就开传——传坏了设备自校验失败，白等一轮。
10. 只在模拟器上测蓝牙——没有真实蓝牙栈，BLE 功能一律真机验证。
11. 扫描过滤死守旧设备名——升级模式下广播名和地址都可能变，永远扫不到"新"设备。
12. 拿 CBError 的码去解释设备侧的 ATT 错误——两个域的错误码含义不同（见 8.1）。
13. 设备电量不足也发起升级——刷写中途掉电触发回退保护，又得从头来一遍（见 4.3）。
14. 固件包与设备型号不匹配也硬刷——init packet 的硬件版本对不上，bootloader 直接拒刷（见 7.6）。

---

## 附录：实操入口地图

| 要做的事 | 位置（界面 / 命令） | 操作说明 |
|---|---|---|
| 手动验证设备 DFU | nRF Connect → Scanner → 连接设备 → DFU 图标 → 选 zip → Upload | 先证明设备端没问题，再写 App |
| 集成 DFU 库 | Podfile 加 `pod 'NordicDFU'` → 终端 `pod install` | 代码里 `import NordicDFU` |
| 抓蓝牙底层包 | Apple 官网下载 Additional Tools for Xcode → Hardware 目录 → PacketLogger | iOS 设备与 Mac 用数据线连，Live Capture |
| 看设备服务表 | LightBlue / nRF Connect → 连接设备 → Services | 对照新固件协议文档逐条核对 |
| 看系统蓝牙日志 | Console.app → 左侧选 iPhone → 过滤 `bluetoothd` | 连接失败原因、链路线索 |
| 模拟意外断连 | 设备断电 / 走出范围 / 设置里关蓝牙 | 分层复现，别用 Xcode 调试器连着测 |
| 模拟升级中断 | 传输 50% 时拔设备 / 强杀 App | 验证恢复流程是否走通 |

**排障"动手顺序"速查**：

```
① 先分清是谁的问题：nRF Connect / LightBlue 直接扫设备
② 广播层：搜得到吗？广播名 / 地址变了吗（DFU 前缀、MAC+1）
③ 连接层：connect 有回调吗？自己的超时计时器在跑吗？
④ 服务层：服务表打出来对过新固件协议吗？Service Changed 有没有？
⑤ 数据层：版本号读出来是多少？停在 bootloader 就走恢复流程
```

---

## 官方与参考链接

- [centralManager(_:didDisconnectPeripheral:error:)](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/centralmanager(_:diddisconnectperipheral:error:))（error 的两种取值；断开后服务与特征失效）
- [CBError.Code](https://developer.apple.com/documentation/corebluetooth/cberror-swift.struct/code)（错误码表）
- [Best Practices for Interacting with a Remote Peripheral Device](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/BestPracticesForInteractingWithARemotePeripheralDevice/BestPracticesForInteractingWithARemotePeripheralDevice.html)（重连三招）
- [Core Bluetooth Background Processing for iOS Apps](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html)（恢复后从 discoverServices 重新开始）
- [Apple Bluetooth 工具页](https://developer.apple.com/bluetooth/)（PacketLogger 随 Additional Tools for Xcode 分发）
- [Nordic IOS-DFU-Library（GitHub）](https://github.com/NordicSemiconductor/IOS-DFU-Library)（pod 'NordicDFU'；nRF51/52 兼容范围；Service Changed 要求；MAC +1 行为）
- [NordicDFU 官方文档](https://nordicsemi.github.io/IOS-DFU-Library/documentation/nordicdfu)（DFUFirmware / DFUServiceInitiator / 三个 delegate 的用法）
- [nRF Connect Device Manager（GitHub）](https://github.com/nordicsemi/IOS-nRF-Connect-Device-Manager/)（NCS / MCUboot 设备的升级库，与 NordicDFU 适用范围不同）
- [NSBluetoothAlwaysUsageDescription](https://developer.apple.com/documentation/BundleResources/Information-Property-List/NSBluetoothAlwaysUsageDescription)（蓝牙权限键，不声明则调用即崩）
- [Nordic 官方文档站](https://docs.nordicsemi.com/)（nRF5 SDK、bootloader 与 DFU 示例，动态内容以此为准）
