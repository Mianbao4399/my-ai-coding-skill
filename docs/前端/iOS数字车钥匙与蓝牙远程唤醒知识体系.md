# iOS 数字车钥匙知识体系（新手友好版）

> 覆盖：iOS 做车钥匙的三条路线 / CCC 与 Apple Wallet CarKey / NFC·UWB·BLE 三通道分工 / BLE 广播与后台唤醒 / CoreBluetooth 状态恢复 / TN3115 重启规则 / 最小可跑 Demo / 排障与速查
> 阅读方式：每个概念先讲"是什么（带比喻）"，再讲"为什么需要它"，最后讲"怎么用 / 怎么排查"。专业名词首次出现均做中英对照，并给一句大白话。
> 信息核验日期：2026-11-20（4.7 节姊妹篇指引新增于 2026-11-25）。动态信息（车型支持、认证清单、审核政策）以官方页面为准，文末附链接。
> 姊妹篇：连接建立之后的断连重连、OTA 固件升级等设备工程实践，见《iOS 蓝牙设备连接与 OTA 固件升级知识体系》。

---

## 目录

- [0. 先用一组比喻建立全局](#0-先用一组比喻建立全局)
- [一、先看清地图：iOS 上做车钥匙的三条路](#一先看清地图ios-上做车钥匙的三条路)
- [二、CCC 数字钥匙与 Apple Wallet CarKey](#二ccc-数字钥匙与-apple-wallet-carkey)
- [三、三条通道的距离分工：NFC / UWB / BLE](#三三条通道的距离分工nfc--uwb--ble)
- [四、BLE 基础：只有两个角色](#四ble-基础只有两个角色)
- [五、iOS 后台模型：App 是怎么"死"的](#五ios-后台模型app-是怎么死的)
- [六、状态恢复三件套：立遗嘱、收遗产、对账](#六状态恢复三件套立遗嘱收遗产对账)
- [七、TN3115：系统什么时候愿意叫醒你](#七tn3115系统什么时候愿意叫醒你)
- [八、唤醒全链路：从一帧广播到一次解锁](#八唤醒全链路从一帧广播到一次解锁)
- [九、实操：最小可跑 Demo（SOP）](#九实操最小可跑-demosop)
- [十、常见问题与排查](#十常见问题与排查)
- [十一、高频速查表](#十一高频速查表)
- [十二、能力清单与选型](#十二能力清单与选型)
- [十三、踩坑清单](#十三踩坑清单)
- [附录：实操入口地图](#附录实操入口地图)
- [官方与参考链接](#官方与参考链接)

---

## 0. 先用一组比喻建立全局

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 数字车钥匙 | Digital Car Key | 把实体钥匙电子化后装进手机 | 让手机能解锁、启动、共享车辆 |
| 安全元件 | Secure Element（SE） | 银行保险柜 | 存密钥，App 读不到也写不进 |
| 钱包 | Wallet / PassKit | 官方的钥匙串 | 系统托管车钥匙、卡券、登机牌 |
| CCC 标准 | Car Connectivity Consortium | 车厂和手机厂的"普通话标准委员会" | 让大家用同一套协议对话 |
| 近场通信 | NFC | 贴一下 | 厘米级、几乎不耗电、天然抗中继 |
| 超宽带 | UWB | 室内 GPS / 声呐 | 米级精测距，能测"飞行时间" |
| 低功耗蓝牙 | BLE | 门口的对讲机 | 数十米广播，负责远场叫醒 |
| 广播 | Advertising（ADV） | 车在门口喊"我在这里" | 无连接、便宜、帧小 |
| 信号强度 | RSSI | 听声音大小猜远近 | 只能粗判距离，防不住中继 |
| 飞行时间 | Time of Flight（ToF） | 掐表算回声 | 信号放大不改变时间，可防中继 |
| 中继攻击 | Relay Attack（Two-Thief Attack） | 两个小偷隔着一条街传话 | 把车与钥匙的认证对话原样延长 |
| 被动无钥匙进入 | Passive Keyless Entry（PKE） | 走到车边，车自己来问 | 车主动发起认证，钥匙自动应答 |
| 状态恢复 | State Preservation and Restoration | 生前立遗嘱，系统当遗嘱执行人 | App 死了，请求还能被系统代管 |
| 重启规则 | TN3115 | 遗嘱什么时候生效的官方说明书 | 规定哪些状态能唤醒、哪些不能 |
| 主机卡模拟 | HCE（Host Card Emulation） | 手机扮演一张卡 | 让读卡器把手机当卡读 |
| 应用协议数据单元 | APDU | 读卡器和卡之间的电报报文 | NFC 读卡 / 卡模拟的指令格式 |
| MFi 认证 | Made for iPhone | Apple 的合作认证门票 | 决定你能不能碰 CarKey |
| 授权文件 | Entitlement | 按 App 签发的通行证 | 没有它，框架初始化就会失败 |

把这些串起来就是一句话：**手机靠近车，车先用 BLE 广播敲门，系统替可能已经"死了"的 App 应门，App 醒来建立加密通道，UWB 测距确认意图，NFC 贴靠完成最后认证。**

---

## 一、先看清地图：iOS 上做车钥匙的三条路

### 1.1 三条路分别是什么

"手机当车钥匙"在 iOS 上不是一个 API，而是三种完全不同的工程形态。

**路线 A：CCC 标准 + Apple Wallet CarKey（系统托管）。** 车钥匙被写进 iPhone 的安全元件（Secure Element，SE——手机里独立的一块安全芯片，密钥进去就出不来），由 Wallet 托管。无感解锁由系统层处理，App 只负责管理界面、配对引导和远程控车。这条路要求车企与 Apple 走 MFi（Made for iPhone，Apple 面向配件与车厂的合作认证计划）合作，普通开发者拿不到资格。

**路线 B：车企自研 BLE App 钥匙（App 就是钥匙）。** 密钥存在 App 沙盒或服务器，App 用 CoreBluetooth 扫描车辆广播、发起连接、走私有协议认证。特斯拉 Phone Key 是典型代表。这条路谁都能做，但 App 被系统回收、被用户强杀，钥匙就失灵。

**路线 C：远程控车（4G / 云通道）。** 手机发 HTTPS 请求到车企云，云再下发指令到车。它解决的是"人在远方"的场景，不是"走近自动开"。很多 App 把 B 和 C 混在一起做，但底层是两套东西。

### 1.2 三条路的硬指标对照

| | A：CCC + Wallet | B：自研 BLE App 钥匙 | C：远程控车 |
|---|---|---|---|
| 钥匙存哪 | iPhone 安全元件（SE） | App 沙盒 / 服务器 | 服务器 |
| 靠近无感 | 系统层处理，可不开 App | 依赖 App 后台存活 | 不适用 |
| 断网可用 | 可用（近场交易不走网络） | 通常可用 | 不可用 |
| 低电量模式可用 | CCC 2.0 起支持（车型 / 机型有差异） | 不支持 | 不支持 |
| 抗中继 | UWB 测距（CCC 3.0） | RSSI 只能缓解，防不住 | 不涉及 |
| 开发门槛 | 车企 + Apple MFi 认证 | 任何人都能做 | 任何人都能做 |
| 典型例子 | 宝马、沃尔沃、极氪、蔚来等（车型差异大） | 特斯拉 Phone Key | 各车企 App 的远程开关锁 |

### 1.3 最容易误解的一件事

在 App 里存个 token（令牌，一段代表身份的字符串）、再写一套蓝牙协议，这不等于"手机钥匙"，更进不了 Wallet。它只是"自研钥匙"。两者在安全等级、责任主体、系统权限上都不是一回事。

> **新手常见疑问：把一张普通 Pass 加进 Wallet，能不能当车钥匙？**
> 不能。普通 Pass 是静态卡券（登机牌、会员卡那类），没有安全认证通道，安全模型完全不成立。车钥匙在 Wallet 里是独立的 CarKey 凭证类型，只有车企能签发。

### 1.4 能力边界一览

| 想做 | 普通开发者 | 车企（MFi） |
|---|---|---|
| App 内 BLE 无感解锁、远程控车 | 可以 | 可以 |
| Core NFC 读卡、读车辆 NFC 卡 APDU | 可以 | 可以 |
| 钥匙管理 UI（分享状态、有效期、权限展示） | 可以（最终动作交 Wallet） | 可以 |
| 自建一张能加进 Wallet 的车钥匙 | 不可以 | 可以 |
| 调用 CarKey 的 UWB 安全测距通道 | 不可以 | 可以（车辆与系统层集成，不是公开 API） |
| 后台静默读 NFC | 不可以 | 不可以 |

---

## 二、CCC 数字钥匙与 Apple Wallet CarKey

### 2.1 CCC 是什么

CCC（Car Connectivity Consortium，车连接联盟）是车厂、手机厂、芯片厂一起定"手机当车钥匙"标准的组织。董事会成员包括 Apple、BMW、福特、通用、Google、本田、现代、奔驰、NXP、松下、三星、Thales、小米等，目前成员超过 300 家（CCC 2026-01 新闻稿）。它的产出是 CCC Digital Key 规范，不是某一家的私有协议。

版本演进有四个关键节点：

| 版本 | 时间 | 射频 | 关键能力 |
|---|---|---|---|
| Release 1.0 | 2018-06 | NFC | 基础版，贴靠解锁 |
| Release 2.0 | 2020-05 | NFC | 可规模化架构，支持低电量模式 |
| Release 3.0 | 2021-07 | UWB + BLE，保留 NFC | 无感靠近、位置感知、防中继 |
| Version 4 | 当前（2026-11）互操作测试中 | 沿用 3.0 射频 | 多级分享、车队 / 租赁、账号级钥匙管理 |

Release 3.0 明确：UWB 提供安全且精确的距离测量，让车能定位已认证的手机；NFC 保留是为了向后兼容和低电量模式。也就是说，**三条通道不是替代关系，是分工关系**。

### 2.2 时间线

| 时间 | 发生了什么 |
|---|---|
| 2018-06-20 | CCC Digital Key 1.0 发布 |
| 2020-05-04 | CCC Digital Key 2.0 定稿（NFC，支持低电量模式） |
| 2020-06 | BMW 宣布首个支持 iPhone 数字钥匙；WWDC20 Session 10006 面向车企介绍 Car Keys |
| 2021-07-13 | CCC Digital Key 3.0 定稿（UWB + BLE） |
| 2024-07-17 | BMW Group 与 NXP 成为首批获得 CCC Digital Key 认证的厂商 |
| 2025 | 全年认证 115 项车辆与模块产品，首批中国车企（蔚来、小鹏、极氪等）通过；3 月起认证范围扩展到 BLE 和 UWB |
| 2026 | CCC Digital Key Version 4 处于互操作测试阶段，重点转向分享、车队、租赁 |

这里要记住一句话：**标准定稿 ≠ 你的 iOS 版本支持 ≠ 你的车型支持**。三个条件同时满足，用户才能真正用上。

### 2.3 Apple Wallet CarKey 怎么落地

钥匙写进 SE，由 Wallet 托管，不在 App 沙盒、不在 Keychain、也不在车企服务器。配对时车主出示一次性配对密码，双方用 SPAKE2+ 和 NIST P-256 曲线建立安全通道，再通过 GlobalPlatform SCP03 把钥匙数据传进 SE。翻译成大白话：双方在不直接把密码发出去的前提下，互相证明"我们知道同一个密码"，然后开一条加密隧道把钥匙送进保险柜。这些细节 Apple Platform Security 文档里有完整描述。

开发侧要知道的是：Apple 确实有一个公开文档的 **CarKey 框架**（iOS 16+），提供 `CarKeyRemoteControl`、`CarKeyRemoteControlSession` 等类，用来对 Wallet 里已配置的车辆做锁车、开天窗这类远程无钥匙操作。但它的门槛写在文档第一段：App 必须有 `com.apple.developer.carkey.session` 授权，而申请这个授权的前提是"已加入 MFi Program 的汽车制造商"。

所以准确的说法不是"没有公开 SDK"，而是：**框架文档公开，使用资格不公开**。普通开发者能读到 API，但拿不到 entitlement（授权文件，Apple 按 App 签发的权限凭证，没有它连框架都初始化不了）。

### 2.4 钥匙分享是怎么走的

不是"发个二维码让对方扫"，而是 Wallet 里的钥匙条目通过信息 App 直接分享。接收方可以选权限（只能解锁 / 可启动 / 可开后备箱）和有效期（一次性 / 限时 / 长期），车主可以随时回收。

对 iOS 开发的意义：App 负责"管理界面"和"拉新引导"，真正的钥匙分发是系统做的。你在 App 里另做一套分享逻辑，和 Wallet 那套是并行的两条线，接管不了。

### 2.5 两个"UWB"别混

CarKey 的 UWB 跑在系统层，用 iPhone 的 U 系芯片，做的是 CCC 3.0 的安全测距。第三方 App 拿不到这条通道。

开发者能用的 UWB 官方框架是 **NearbyInteraction**，它面向第三方配件（`NINearbyAccessoryConfiguration`），能做相对距离测量。但它不是 CCC 车钥匙协议，测距结果也不参与 Wallet 的钥匙认证。拿它做"辅助测距"可以，拿它冒充 CarKey 的抗中继能力不行。

### 2.6 iPhone 的 HCE 能力

HCE（Host Card Emulation，主机卡模拟）是让手机扮演一张卡、被读卡器读取的能力。iPhone 从 iOS 17.4 起支持，但要同时满足两个门槛：

- 系统版本：iOS 17.4 及以上，使用 Core NFC 的 `CardSession`。
- 地区与资质：仅限欧洲经济区（EEA）的用户和开发者，且需要 Apple 审批的 HCE entitlement。

官方列出的场景包括店内支付、车钥匙、闭环交通、企业工牌、酒店钥匙、活动门票。

单次模拟最长 60 秒，超时会收到 `maxSessionDurationReached`。需要 `com.apple.developer.nfc.hce` 与 `com.apple.developer.nfc.hce.iso7816.select-identifier-prefixes` 授权。

所以 iPhone 能不能当卡，取决于"版本 + 地区 + 资质"是否同时满足。做方案评估时先确认目标市场和资质，再决定要不要走这条路。

---

## 三、三条通道的距离分工：NFC / UWB / BLE

### 3.1 为什么需要三条

因为它们解决的是三个不同距离上的问题，物理特性决定了谁干不了谁的活。

NFC 工作在厘米级，必须贴上去。距离太短，攻击者很难在中间做手脚，所以它天然适合"最后一步的认证与开门"，而且几乎不耗电。

UWB 工作在米级，靠测量信号往返的飞行时间算距离。它能判断"人正在靠近哪个车门"，也能防中继，但功耗和成本都比 NFC 高。

BLE 能覆盖数十米，广播便宜，适合"远远地先把 App 叫醒"。但它的 RSSI（Received Signal Strength Indicator，接收信号强度）只能粗判距离，人体遮挡、手机放裤兜、车型天线位置都会让数值抖得厉害。

### 3.2 分工矩阵

| 通道 | 典型半径 | 主要任务 | 功耗 | 抗中继 | iOS 侧由谁负责 |
|---|---|---|---|---|---|
| BLE | 数十米 | 远场广播、叫醒 App、建立安全通道 | 低 | 弱（RSSI 可伪造） | App 的 CoreBluetooth + 系统状态恢复 |
| UWB | 米级 | 精测距、判断意图、防中继 | 中 | 强（ToF 测距） | CarKey 走系统层；开发者的 NearbyInteraction 不参与认证 |
| NFC | 厘米级 | 贴靠认证、完成开门 / 启动 | 极低 | 强（距离太短） | Wallet 系统层；或 Core NFC 读卡 / EEA 的 CardSession |

半径为定性示意，具体数值以各车企方案和 CCC 规范为准。

### 3.3 接力顺序

一次完整的靠近，通常是三层接力：

1. **BLE 远场叫醒**：车端周期广播，命中判据后系统拉起 App，建立安全通道。
2. **UWB 中场定意图**：测距确认人在靠近哪个门、要不要提前解锁。
3. **NFC 贴靠收尾**：贴上去完成最后认证，开门或启动。

谁先到站谁干活。不是每次都要三层全走一遍，车型方案不同，裁剪也不同。

### 3.4 中继攻击（Relay Attack）：为什么 RSSI 防不住，ToF 可以

中继攻击（Relay Attack，也叫双贼攻击 Two-Thief Attack）不破解密钥，也不重放旧报文。它做的是把车和钥匙之间的认证对话，实时延长到两个相距很远的地点。攻击者只是"传话筒"。

先看被转发的是什么。被动无钥匙进入（Passive Keyless Entry，PKE）的认证流程是"挑战—应答"（Challenge-Response）：

1. 车端发起认证挑战（Challenge，一个随机数）。传统遥控钥匙由车端的低频信号（LF，Low Frequency，约 125 kHz）唤醒；BLE 手机钥匙则由车端广播或连接触发。
2. 钥匙或手机收到挑战后，用内置密钥计算应答（Response）。传统钥匙通过 UHF（Ultra High Frequency，特高频，315 / 433 / 868 / 915 MHz）发回，手机则通过 BLE 发回。
3. 车验证应答正确，才解锁或允许启动。

挑战每次都是新的随机数，所以录下上一次的应答没用，中继必须是实时的。

中继攻击把这条链路拆成两段：

1. 车主在屋内，钥匙或手机放在身边。攻击者把"钥匙旁设备"放在能连上它的位置，把"车旁设备"放在车门附近。
2. 车旁设备接收车端发出的认证挑战（BLE 手机钥匙则是连接请求与后续挑战），通过 4G、Wi-Fi 或专用射频转发给钥匙旁设备。
3. 钥匙旁设备把挑战原样发给钥匙；钥匙以为是车在问，算出应答。
4. 钥匙旁设备把应答转发回车旁设备，车旁设备再发给车。车验证通过，解锁。

所以，被捕获的不是某个"解锁指令"，而是**车与钥匙之间的认证挑战和应答**。车主不需要按任何键：被动进入系统里，触发点是车旁的人拉门把手或靠近车辆，车主动发起挑战，钥匙自动应答。如果是主动按键遥控，按键发出的指令也可能被中继，但经典的中继攻击针对的是"靠近自动解锁"的被动链路。对 BLE 手机钥匙，手机需要处于可连接、可应答的状态（App 还能被系统唤醒、蓝牙开着）；用户强杀 App 或关掉蓝牙，中继链就拿不到应答。

这也解释了为什么 RSSI 防不住：RSSI 只衡量"信号听起来多强"，攻击者加一个放大器，车看到的信号强度反而更高。它不衡量"信号跑了多久"。ToF 测的是往返飞行时间，中继链路再快也会引入额外的处理和传输延迟，测出来的距离就会变远，车可以拒绝解锁。这就是 CCC 3.0 用 UWB 做安全测距的核心原因。

自研 BLE 钥匙只能做缓解：用户确认弹窗、二次认证、异常位置告警。要做真正的抗中继，绕不开 CCC + Wallet + UWB。

### 3.5 如果走 Core NFC 读卡，配置只有四步

Core NFC 不是车钥匙主线，但做诊断、读车辆 NFC 卡时会用到。工程上漏一项就调不通。

1. Info.plist 加 `NFCReaderUsageDescription`，写清楚读什么、为什么读。
2. 打开 NFC 能力：Xcode → Signing & Capabilities → + Capability → Near Field Communication Tag Reading，带上 `com.apple.developer.nfc.readersession.formats` 授权。
3. `NFCTagReaderSession` 的 `pollingOption` 里加 `.iso14443`，并把 AID 写进 Info.plist 的 `com.apple.developer.nfc.readersession.iso7816.select-identifiers`。AID 不在白名单里，标签根本不会被系统识别出来。
4. 会话必须在前台发起，系统会弹扫描面板；一次会话大约 60 秒，多次 APDU 要在同一次会话里完成。

APDU 的字段和 iOS 参数是一一对应的：

| APDU 字段 | iOS 参数 | 含义 |
|---|---|---|
| CLA | `instructionClass` | 指令类别 |
| INS | `instructionCode` | 指令码 |
| P1 / P2 | `p1Parameter` / `p2Parameter` | 参数 |
| Lc + Data | `data` | 数据长度与内容 |
| Le | `expectedResponseLength` | 期望响应长度 |

典型流程是先"选应用"（`00 A4 04 00` + AID），再发业务指令。返回 `90 00` 才算成功，`6A82` 表示卡上找不到这个应用，不是通信失败。

还有两条限制：App 在后台时 NFC 会话不可用；模拟器不能测 NFC，必须真机。

另外，NDEF 和 APDU 是两套用法。读普通标签用 `NFCNDEFTag`，做安全认证类交互才走 `NFCISO7816Tag`。车钥匙场景属于后者；用 NDEF 写的"开门链接"是明文噱头，没有双向认证，别把它当安全方案。

---

## 四、BLE 基础：只有两个角色

### 4.1 外围与中心

BLE（Bluetooth Low Energy，低功耗蓝牙）里只有两个角色。

**外围（Peripheral）**是车端或钥匙端设备，周期性向外广播。**中心（Central）**是手机 App，用 `CBCentralManager` 监听广播、按判据筛选、发起连接。

比喻：车站在门口喊"我在这里，我是你的车"，手机是听见喊声的人。喊话不需要对方回应，所以便宜；真要办事，才握手进屋。

### 4.2 广播和连接是两件事

广播（Advertising）是无连接的，收发一帧就完事，帧里可以带服务 UUID、状态标志、厂商自定义数据。连接（Connection）要维持链路、定期交换数据，费电但能双向通信。

远场唤醒走广播，因为需求是"用最低的成本一直喊"。真正解锁时才建立连接，走 GATT（Generic Attribute Profile，蓝牙里读写特征值的标准协议）。

### 4.3 广播帧里有什么

一帧广播能带的东西不多，够用的有：

- 设备地址（通常是随机化的，不能当唯一身份）
- 服务 UUID（Universally Unique Identifier，通用唯一识别码；比如广播常用的 `FEAA`（Eddystone 格式），或标准服务 `180F`（电池服务））
- Flags（可发现、可连接等标志）
- Manufacturer Data（厂商自定义字节，常用来放身份或状态）
- Tx Power（发射功率，手机据此估算距离）

iOS 在后台扫描时有两个行为变化：`CBCentralManagerScanOptionAllowDuplicatesKey` 会被忽略，同一个设备的多次发现会合并成一次；如果所有扫描的 App 都在后台，系统会拉长扫描间隔，发现设备变慢。

iOS 26 起还有一个例外：启动 Live Activity 的 App 可以保留前台级蓝牙权限，详见 5.7。

### 4.4 唤醒判据：三道题

广播唤醒要同时答对三道题。

**判据**：广播里的身份信息（服务 UUID、厂商数据、标志位）必须和 App 生前挂起的扫描过滤器精确匹配。写宽了天天醒，写窄了叫不醒。别把别人家的车当成自家门。

**安全**：唤醒只是敲门，认证才是进门。广播帧永远不携带可仿造的敏感凭据，握手、挑战应答、密钥校验都在加密通道里完成。

**功耗**：后台扫描由系统调度，扫描间隔和窗口受平台约束。判据越准、广播越克制，系统的代管成本越低，唤醒延迟和续航也越好看。

这三道题互相牵制：判据收紧会推高延迟，广播放宽会吃掉功耗。工程调优就在这张三角里做。

### 4.5 广播能被伪造吗

能。广播是明文、无认证的，任何人都能复制一帧。所以它只能负责"敲门"，不能负责"证明身份"。真正的身份认证必须放到连接后的加密通道里。这也是为什么"广播里塞个密钥"的方案从根上就不成立。

### 4.6 广播、扫描、连接的时间尺度

BLE 的"检测"不是每隔固定几秒发一次信号，而是三层时间尺度叠加：

| 环节 | 谁控制 | 规范允许范围 | 车钥匙常见做法 |
|---|---|---|---|
| 广播间隔（Advertising Interval） | 外围设备（车端或手机） | 20 ms–10.24 s | 几十毫秒到几百毫秒，兼顾响应与功耗 |
| 扫描间隔 / 扫描窗口（Scan Interval / Window） | 中心设备（车端或手机） | 2.5 ms–10.24 s | iOS 不暴露给 App；前台较连续，后台由系统拉长 |
| 连接间隔（Connection Interval） | 双方协商 | 7.5 ms–4 s | 15–30 ms 常见 |

体感上的"靠近就开"通常有 0.5–2 秒延迟，取决于广播 / 扫描占空比、RSSI 滤波和认证耗时。不存在一个固定的"每隔 X 秒检测一次"。

iOS 在后台还有两个限制：`allowDuplicates` 被忽略，多次发现合并成一次，App 拿不到连续 RSSI；如果所有扫描 App 都在后台，系统会拉长扫描间隔。所以后台靠近检测更依赖连接事件或系统唤醒，而不是高频广播采样。

车端的具体参数（广播间隔、RSSI 阈值、滤波窗口）属于车厂协议，通常不公开。调优时以车端文档和实测为准。

### 4.7 连接之后：断连、重连与固件升级

广播把 App 叫醒、连接建立，只是链路的前半段。日常使用的大头在连接之后：链路会断（车主走远、车端休眠、系统回收资源），断开后服务与特征全部失效，重连必须重新发现服务、重新订阅；设备固件靠 OTA 无线升级，升级模式下广播名甚至设备地址都可能变化，扫描过滤要提前考虑。

这些"连接后工程"的完整实践——断开与重连状态机、指数退避、OTA 固件升级全流程、升级后连不上的排查——单独成篇，见《iOS 蓝牙设备连接与 OTA 固件升级知识体系》。

---

## 五、iOS 后台模型：App 是怎么"死"的

### 5.1 三个状态

常规 App 在 iOS 上只有三种状态：

**Active（前台运行）**：CPU、网络、蓝牙全权在握。

**Suspended（挂起）**：退到后台几秒后冻结在内存里，不执行任何代码，但状态原样保留。

**Terminated（终止）**：被系统移出内存，或者崩溃。App 层面已经死了，没有任何代码在跑。

比喻：前台是在岗，挂起是趴在工位睡着，终止是人被清出办公楼、工位也没了。

### 5.2 为什么系统要杀 App

内存压力、后台资源公平、省电。用户开了一堆 App，系统必须挑一些清掉。你的车钥匙 App 不会因为"很重要"就被豁免。

### 5.3 例外规则：挂起不等于失联

挂起之前，App 通过 CoreBluetooth 留下的扫描请求和连接请求，会由系统蓝牙守护进程（`bluetoothd`）代管。车端广播命中时，系统替 App 收下这封信，再按恢复机制决定要不要把 App 拉起来处理。

核心在于：**蓝牙事件唤醒的不是进程，是它生前挂起的请求。**

### 5.4 两个蓝牙后台模式

要在后台继续做蓝牙的事，必须在工程里声明后台模式。两个模式对应两个角色：

| 模式 | 中文 | 作用 | 典型场景 |
|---|---|---|---|
| `bluetooth-central` | 中心角色后台运行 | 后台继续扫描、发现外围、收发 GATT 数据 | 车钥匙 App、手环、传感器网关 |
| `bluetooth-peripheral` | 外围角色后台运行 | App 在后台对外广播、可被中心设备发现和连接 | 把手机变成信号源、点对点传输 |

Xcode 里的入口：选中 target → **Signing & Capabilities** → 左上角 **+ Capability** → 搜 **Background Modes** → 勾选 **Uses Bluetooth LE accessories**（对应 `bluetooth-central`）或 **Acts as a Bluetooth LE accessory**（对应 `bluetooth-peripheral`）。

声明不是开关，是资格证。没声明的 App，后台蓝牙行为会被系统直接掐断。

### 5.5 Info.plist 里的权限键

iOS 13 及以后，访问蓝牙必须在 Info.plist 里写 `NSBluetoothAlwaysUsageDescription`，否则一调 API 就崩。如果 deployment target 早于 iOS 13，还要额外加 `NSBluetoothPeripheralUsageDescription`。

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>靠近车辆时自动解锁</string>
<key>UIBackgroundModes</key>
<array>
    <string>bluetooth-central</string>
</array>
```

文案要写清楚真实用途，审核会看。写"提升用户体验"这种空话容易被拒。

### 5.6 被唤醒后只有大约 10 秒

Apple 的后台编程指南给出的经验值是：App 被蓝牙事件唤醒后，大约有 10 秒时间完成任务，理想情况是尽快处理完、让系统重新挂起。后台执行时间过长会被系统限流甚至杀掉。

所以恢复路径里不要拉接口、写大文件、初始化全家桶。只做最小重建，重活丢给下一次前台或后台任务。

### 5.7 iOS 26 的两个新变化

**Live Activity**（实时活动，锁屏或灵动岛上持续显示的状态卡片）：App 在进入后台前启动 Live Activity，并且已经实例化了 `CBManager`，就能在后台保留前台级蓝牙权限。适合需要持续扫描的场景，但会带来额外的 UI 和生命周期成本。

**AccessorySetupKit**：iOS 18 起提供的配件配对框架。TN3115 注 5 写明，iOS 26 起只有用 AccessorySetupKit 配置蓝牙配件的 App，才会在用户强杀后被系统重新启动。这是强杀规则的一个例外。

---

## 六、状态恢复三件套：立遗嘱、收遗产、对账

CoreBluetooth 的状态保存与恢复（State Preservation and Restoration）只有三个关键 API。可以把它理解成"生前立遗嘱，系统当执行人，重启后对账"。

### 6.1 第一件：立遗嘱

初始化 `CBCentralManager` 时，在 options 里传 `CBCentralManagerOptionRestoreIdentifierKey`，值是一个稳定的字符串。系统用这个 UID 识别"这个 manager 的请求要代管"，所以它在 App 的各次执行中必须保持一致。

每次启动都要传，不是只传一次。

### 6.2 第二件：收遗产

实现 `centralManager(_:willRestoreState:)`。系统为了处理挂起的扫描或连接而重启 App 时，这是第一个被调用的方法，早于 `centralManagerDidUpdateState`。

回调的 `dict` 里带三类东西：

| Key | 内容 |
|---|---|
| `CBCentralManagerRestoredStatePeripheralsKey` | App 停止时已连接或有待连接的外围列表 |
| `CBCentralManagerRestoredStateScanServicesKey` | 当时正在扫描的服务 UUID |
| `CBCentralManagerRestoredStateScanOptionsKey` | 当时的扫描参数 |

这是恢复的唯一入口。错过它，App 就算被拉起来，也不知道自己该干什么。

### 6.3 第三件：冷启动对账

App 被彻底重启时，需要重建恢复标识。这里不能用 `UIApplicationLaunchOptionsBluetoothCentralsKey`：在 scene-based App（用 Scene 管理界面的 App，新建工程默认就是）里，`launchOptions` 启动时永远是 `nil`，拿不到这个 key；而且它在 iOS 26 已经弃用。

正确做法是：自己生成一个稳定的 UID，持久化（比如 `UserDefaults`），每次启动都把它传给 `CBCentralManager` 的初始化 options。

### 6.4 恢复的是请求清单，不是内存现场

系统恢复的是"有哪些 manager、哪些挂起的扫描、哪些连接"，不是你内存里的变量、UI 状态、业务状态机。变量不会被恢复，对象需要重建。

所以 `willRestoreState` 更像"重新接管"，不像"恢复现场"。业务状态机要自己重建，当前处于"已配对 / 已连接 / 待认证 / 待解锁"哪一步，要么持久化，要么从协议重新问出来。

### 6.5 最小代码

```swift
final class WakeManager: NSObject, CBCentralManagerDelegate {
    private var central: CBCentralManager!

    override init() {
        super.init()
        // 恢复标识跨启动必须稳定；scene-based App 拿不到 launchOptions
        let key = "centralRestoreUID"
        let uid = UserDefaults.standard.string(forKey: key) ?? UUID().uuidString
        UserDefaults.standard.set(uid, forKey: key)
        central = CBCentralManager(
            delegate: self,
            queue: .main,
            options: [CBCentralManagerOptionRestoreIdentifierKey: uid]
        )
    }

    // 系统重启 App 时，第一个被调用的方法
    func centralManager(_ central: CBCentralManager,
                        willRestoreState dict: [String: Any]) {
        // 以回调参数为准重建状态机，别假设内存对象还活着
        let restoredPeripherals =
            dict[CBCentralManagerRestoredStatePeripheralsKey] as? [CBPeripheral]
        _ = restoredPeripherals // 后续用它重建连接与业务状态
    }

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        guard central.state == .poweredOn else { return }
        central.scanForPeripherals(withServices: [CBUUID(string: "FEAA")])
    }
}
```

这段只解决"能被叫醒 + 知道被叫醒"两件事。连接后的 GATT 交互、认证握手、业务状态机，都要另写。

### 6.6 四个常见错误

- 恢复标识每次启动都用新 `UUID()`，等于每次换遗嘱编号，系统对不上账。
- 只恢复了扫描，没重建业务状态机，App 处于"半醒"状态：进程在跑，解锁流程卡住。
- 在 `willRestoreState` 里拉接口、写文件、初始化全套 SDK，十秒预算瞬间烧完。
- 假设连接还在。内存里没有任何活对象，一切以回调参数为准。

---

## 七、TN3115：系统什么时候愿意叫醒你

### 7.1 规则表

Apple Technical Note TN3115 专门回答"什么状态下 App 会被蓝牙事件重新启动"。2025-09-15 更新的版本是八行：

| App 或设备状态 | 会被重新启动吗？ |
|---|---|
| App 在内存中挂起 | 不需要重启，直接激活（注 1） |
| App 已从内存移除 | 会 |
| App 崩溃 | 会 |
| 用户强制退出 App | 不会（注 5） |
| 在设置里切换蓝牙开关 | 不会（注 2） |
| 在控制中心切换蓝牙按钮 | 会（注 5） |
| 切换飞行模式 | 会（注 3） |
| 设备重启 | 会（注 4） |

五条附注值得逐条看：

- 注 1：挂起的 App 不需要 relaunch，系统直接激活它。
- 注 2：用户可能在设置里关掉蓝牙电源，这种情况下不会重启。
- 注 3：只有当飞行模式没有把蓝牙一起关掉时才成立。
- 注 4：如果设备需要密码解锁，重启后第一次解锁之前，App 不会被重启。
- 注 5：iOS 26 / iPadOS 26 起，只有用 AccessorySetupKit 配置蓝牙配件的 App 才会被重新启动。这条同时挂在"强杀"和"控制中心切换蓝牙"两行，工程上按最保守的理解做测试。

另外还有一个总前提：**只有当 App 当时确实在等待某个蓝牙事件**（扫描、连接、订阅通知，外围角色还包括正在广播），并且对应事件真的发生了，系统才会重启和恢复它。没有挂起的请求，就没有唤醒。

### 7.2 强杀：唤醒资格被关闭

TN3115 规定：用户从多任务界面上划强杀 App 后，系统不会重启它。这等于亲手剥夺了 App 的唤醒资格。此后不管车端怎么广播，系统都不会再拉起这个 App，直到用户手动重新打开一次。注 5 补充的是 iOS 26 的例外：用 AccessorySetupKit 配对的配件 App 仍可被重启。

> **新手常见疑问：是不是要一直把 App 开着，不能关？**
> 不是。"关闭 App"要分情况：回到桌面、锁屏、切到别的 App，都只是进入后台或挂起，系统仍可在蓝牙事件到来时唤醒；系统因内存压力回收，也能按状态恢复规则重启。只有"从多任务界面上划强杀"才会剥夺唤醒资格。强杀后手动打开一次即可恢复。

特斯拉官方排障文档的要求是 "Ensure the Tesla app is running in the background"，指的是后台运行，不是前台常开。特斯拉是否使用 CoreBluetooth 状态恢复没有公开，具体行为以它的 App 为准；但 iOS 的强杀规则对所有 App 都一样。

强杀剥夺的是"系统因蓝牙事件重启 App"的资格，不等于车端立刻、永久失去所有响应。车端可能有最近认证的缓存，车门可能还没锁，手机与车之间也可能还有系统级蓝牙连接。具体到特斯拉，实现未公开；如果"开门"和"启动"是一起成功、一起失败，重点查三件事：App 当时是不是真的被强杀、车端会话缓存有没有过期、手机与车的蓝牙链路质量如何。

代码里没有"绕过"的办法。系统规则对所有 App 一视同仁，所谓"保活黑科技"在这个场景下无效，还可能带来审核风险。

### 7.3 产品对策只有一条

教育用户。在设置页、首次使用引导（onboarding）、以及"功能失灵"的诊断页里，用大白话解释"强杀 = 钥匙暂时失效"，并给出恢复动作：重新打开 App 一次。

客服侧也要统一话术。用户说"钥匙不好使"，常见原因是强杀过 App。先问"最近有没有从后台把 App 划掉"，比让用户重装 App 有效得多。

### 7.4 测试用例直接从这张表来

TN3115 的八行就是八条测试用例。工程上至少覆盖四种前置态：

| 前置态 | 怎么造 | 预期 |
|---|---|---|
| 挂起 | 打开 App，回桌面，锁屏，等 30 秒 | 直接被激活，不需要重启 |
| 从内存移除 | Xcode 里 Stop 结束进程（不是上划强杀） | 后台重启，触发 `willRestoreState` |
| 崩溃 | 代码里 `fatalError()` 或手动崩溃 | 同"从内存移除" |
| 用户强杀 | 多任务界面上划 | 不重启，这是正确行为 |

---

## 八、唤醒全链路：从一帧广播到一次解锁

### 8.1 五步流水线

1. **车端**：ECU（Electronic Control Unit，车上的电子控制单元）按固定间隔向外广播，帧里带钥匙判据（服务 UUID、厂商数据、标志位）。
2. **系统**：`bluetoothd` 扫描并匹配 App 生前挂起的请求。
3. **系统**：按 TN3115 规则把 App 拉起来，后台运行，不带 UI。
4. **App**：`willRestoreState` 重建 manager、扫描清单和业务状态机。
5. **App**：发起连接，走协议握手认证，完成解锁或启动。

### 8.2 第 2、3 步完全在系统层

这两步 App 代码零参与。你能控制的只有两件事：生前登记什么请求，身后如何接管。

这也是为什么链路测试要按"系统视角"设计，而不是按 App 视角。你在前台点按钮能跑通，不代表 App 死后能被叫醒。

### 8.3 时序图

```
车端 ECU            bluetoothd              你的 App
   |                   |                       |
   |-- 广播 ADV ------>|                       |
   |                   |-- 匹配挂起请求         |
   |                   |-- relaunch（后台）---->|
   |                   |                       |-- willRestoreState
   |<------------------ 建立连接 ---------------|
   |<------------------ 认证握手 ---------------|
   |------------------- 解锁结果 -------------->|
```

### 8.4 唤醒成功不等于解锁成功

进程起来只是第一步。恢复后的第一次业务动作成功，才算这条链路通了。很多团队把"看到 App 被拉起"当成验收标准，结果上线后用户还是开不了门。

---

## 九、实操：最小可跑 Demo（SOP）

> 这一节是执行线，前面的章节是知识点。照着做，可以跑通"App 死后被广播唤醒"。

### 9.0 全景步骤总览

```
第 0 步 前置准备（Mac / 真机 / 账号 / BLE 外设）
  → 第 1 步 新建 Xcode 工程
  → 第 2 步 打开 Background Modes
  → 第 3 步 配 Info.plist 权限
  → 第 4 步 写最小状态恢复代码
  → 第 5 步 造一个 BLE 外设
  → 第 6 步 四种前置态逐个测
  → 第 7 步 用 Console.app 看 relaunch 日志
  → 第 8 步 断言恢复后的第一次业务动作
```

### 9.1 第 0 步：前置准备

| 准备项 | 要求 | 说明 |
|---|---|---|
| Mac | 能装当前 Xcode | Xcode 从 App Store 装 |
| iPhone | 真机，支持 BLE | 模拟器没有蓝牙硬件，测不了唤醒 |
| Apple ID | 免费账号即可 | 免费账号可以真机调试，签名有效期较短；付费 Developer Program 的年费和权益以 Apple 官网为准 |
| BLE 外设 | 第二台 iPhone 或开发板 | 第二台 iPhone 装 nRF Connect，用来模拟车端广播 |
| 日志工具 | macOS 自带 Console.app | 看 `bluetoothd` 的 relaunch 记录 |

### 9.2 第 1 步：新建工程

Xcode → **File → New → Project** → **iOS** → **App** → Interface 选 SwiftUI 或 Storyboard 都行 → Language 选 Swift。记下 Bundle Identifier，后面看日志要用。

### 9.3 第 2 步：打开 Background Modes

选中 target → **Signing & Capabilities** → 左上角 **+ Capability** → 搜 **Background Modes** → 勾 **Uses Bluetooth LE accessories**。

对应到 Info.plist 就是 `UIBackgroundModes` 数组里的 `bluetooth-central`。如果你还要让手机当外设，再勾 **Acts as a Bluetooth LE accessory**。

### 9.4 第 3 步：配 Info.plist 权限

打开 Info.plist → 右键 **Add Row** → 选 **Privacy - Bluetooth Always Usage Description**，填一句真实用途。也可以直接加前面 5.5 节那段 XML。

### 9.5 第 4 步：写最小状态恢复代码

把 6.5 节的 `WakeManager` 贴进工程，在 App 启动时创建它并强引用（别让它被释放）。然后补 `centralManager(_:didDiscover:advertisementData:rssi:)` 回调，打印设备名和 RSSI，确认判据能命中。

`scanForPeripherals(withServices:)` 在后台必须带服务 UUID，前台可以传 `nil`。Demo 里用 `FEAA` 只是举例，实际要和车端协议对齐。

### 9.6 第 5 步：造一个 BLE 外设

第二台 iPhone：App Store 装 **nRF Connect** → 切到 **Advertiser** → 添加一个 Service UUID（和代码里的 `FEAA` 一致）→ 点 **Start**。

没有第二台手机就用开发板（nRF52、ESP32 都行），烧一个广播示例。广播间隔先设 100–500ms，别设太快，省电也更接近真实车端。

### 9.7 第 6 步：四种前置态逐个测

| 序号 | 前置态 | 操作 | 预期 |
|---|---|---|---|
| 1 | 挂起 | 打开 App，回桌面，锁屏，等 30 秒 | App 被激活，不需要重启 |
| 2 | 从内存移除 | Xcode 里点 Stop 结束进程，然后断开调试器 | 后台重启，日志出现 `willRestoreState` |
| 3 | 崩溃 | 代码里触发 `fatalError()` | 同 2 |
| 4 | 用户强杀 | 多任务界面上划 | 不重启，这是正确行为 |

测试时尽量别连着 Xcode 调试器。调试器会影响系统的挂起和回收行为，有些场景根本造不出来。至少要在拔掉调试器后用 Console.app 观察一遍。

### 9.8 第 7 步：看日志

打开 **Console.app** → 左侧选中你的 iPhone → 搜索 `bluetoothd`，或者直接过滤你的进程名。也可以在 Xcode 里 **Window → Devices and Simulators → Open Console**。

正常命中时，日志大概长这样（示意）：

```
[bluetoothd] match found: peer=VEH-xxxx adv-key=match
[bluetoothd] app relaunched (bg) restore-id=com.team.key
[CB] willRestoreState: centrals=1 scanning=1
```

看到 relaunch 但没有 `willRestoreState`，先查恢复标识有没有传、是不是稳定。看到 `willRestoreState` 但业务没动，查状态机重建。

### 9.9 第 8 步：断言终点

验收标准不是"进程起来了"，而是"恢复后的第一次业务动作成功"：连接建立、认证完成、解锁指令发出、车端回执收到。

把这条用例放进真机回归组，每次协议改版全量跑一遍。

### 9.10 第 9 步（可选）：iOS 26 的两个新变化

AccessorySetupKit 与 Live Activity 的作用和限制见 5.7。两者都有额外接入成本，先把基础方案跑通，再评估是否接入。

---

## 十、常见问题与排查

> 故障分布有规律：权限问题在启动时爆，扫描问题在前台正常后台失灵，唤醒问题在日志里没有 relaunch，恢复问题在 relaunch 之后断片，业务问题在认证环节卡住。按这个顺序查，比乱试快。

### 10.1 问题一：强杀后怎么都不唤醒

**现象**：用户反馈"钥匙不好使"，车端广播正常，Console 里没有任何 relaunch 记录。

**原理（新手版）**：TN3115 规定，用户强制退出 App 后系统不会重启它。系统不是没收到广播，是拒绝为这个 App 启动。注 5 的 iOS 26 例外只覆盖 AccessorySetupKit 配对的配件 App，普通 App 不适用。

**怎么办**：
1. 让用户手动打开 App 一次，唤醒资格恢复。
2. 在 App 里做"诊断"页，检测并提示"App 曾被强制退出"。
3. 客服话术统一：先问有没有划掉 App，再谈其他。

**预防**：首次使用引导（onboarding）和设置页常驻说明；不要承诺"永远无感"，承诺"打开一次即恢复"。如果是有时能开、有时不能，见 10.11。

### 10.2 问题二：模拟器上永远测不出唤醒

**现象**：模拟器里 UI 正常，真机上不弹蓝牙权限；或者模拟器能跑但靠近毫无反应。

**原理（新手版）**：Simulator 没有真实蓝牙硬件栈，CoreBluetooth 在模拟器上的行为不可靠，唤醒测试更无从谈起。

**怎么办**：
1. 换真机。
2. 用真外设（第二台手机或开发板）。
3. 把唤醒用例从模拟器的持续集成（CI）里摘出去，单列真机用例。

**预防**：项目一开始就把"蓝牙功能只在真机验证"写进测试规范。

### 10.3 问题三：App 被回收后不恢复

**现象**：退后台一段时间再靠近，App 完全没反应，日志里也没有 `willRestoreState`。

**原理（新手版）**：要么初始化时没传 `CBCentralManagerOptionRestoreIdentifierKey`，要么每次启动都用新的 `UUID()`，系统对不上账。

**怎么办**：
1. 检查 `CBCentralManager` 的初始化 options。
2. 把 UID 存进 `UserDefaults`，每次启动读同一个。
3. 确认 `willRestoreState` 真的实现了，且 delegate 没被提前释放。

**预防**：把"恢复标识是否稳定"写进代码评审（code review）清单。

### 10.4 问题四：后台扫描扫不到

**现象**：前台能看到设备，一退后台就没了。

**原理（新手版）**：后台扫描间隔被拉长，`allowDuplicates` 被忽略，系统对服务 UUID 过滤更严格。前台那套"实时刷 RSSI"的逻辑在后台不成立。

**怎么办**：
1. `scanForPeripherals` 带上服务 UUID。
2. 不要依赖 `allowDuplicates` 做实时判断。
3. iOS 26 可评估 Live Activity 方案，但不是常规解。

**预防**：前台和后台各写一套判据测试，别只测前台。

### 10.5 问题五：一启动就崩

**现象**：

```
This app has crashed because it attempted to access privacy-sensitive
data without a usage description. The app's Info.plist must contain
an NSBluetoothAlwaysUsageDescription key with a string value explaining
to the user how the app uses this data.
```

**原理（新手版）**：Info.plist 缺 `NSBluetoothAlwaysUsageDescription`。iOS 13 起访问蓝牙必须声明用途，否则系统直接终止进程。

**怎么办**：
1. Info.plist 加 key，写真实用途。
2. deployment target 早于 iOS 13 的，再加 `NSBluetoothPeripheralUsageDescription`。

**预防**：建工程模板时就带上，别等第一次调 API 才想起来。

### 10.6 问题六：唤醒后被系统杀掉

**现象**：App 被拉起，跑了一半没了；Console 里能看到 jetsam（iOS 因内存压力回收进程的机制）或 CPU 异常。

**原理（新手版）**：被唤醒后大约只有 10 秒预算。在 `willRestoreState` 里拉接口、初始化 SDK、写大文件，很容易超时。

**怎么办**：
1. 恢复路径只做最小重建：manager、扫描清单、状态机。
2. 重活丢到下一次前台，或用短任务异步处理。
3. 给恢复路径做耗时打点，超过 2 秒就要排查。

**预防**：把 `willRestoreState` 当冷启动入口，按启动性能预算管理。

### 10.7 问题七：恢复了扫描，业务却卡住

**现象**：进程起来了，扫描也在跑，但解锁流程停在某个步骤。

**原理（新手版）**：恢复的是请求清单，不是内存现场。业务状态机没重建，App 不知道自己走到哪一步了。

**怎么办**：
1. 在 `willRestoreState` 里重建状态机。
2. 把关键阶段持久化，或者连接后重新问一遍车端。
3. 用"恢复后第一次业务动作"做验收。

**预防**：把 `willRestoreState` 和 `didFinishLaunching` 当成两个平等的冷启动入口，行为要一致。

### 10.8 问题八：NFC 的 6A82 被当成通信失败

**现象**：`sendCommand` 返回 `6A82`，代码把它当超时，反复重试。

**原理（新手版）**：`6A82` 在 ISO 7816 里表示"找不到应用或文件"，通常是 AID（Application Identifier，卡上应用的编号）不在卡上，或者 AID 配错。它不是通信故障，重试多少次结果都一样。

**怎么办**：
1. 核对 AID。
2. 检查 Info.plist 里的 AID 白名单（`com.apple.developer.nfc.readersession.iso7816.select-identifiers`）。
3. 把状态字按语义分流，别一律重试。

**预防**：把常用 SW1/SW2 码表放进排障文档（见 11.5）。

### 10.9 问题九：RSSI 靠近解锁忽灵忽坏

**现象**：有时很近才开，有时远处就开，手机放裤兜里干脆不开。

**原理（新手版）**：RSSI 受人体遮挡、手机朝向、车型天线位置影响极大。同一套阈值在不同场景抖动严重，这不是代码问题，是物理特性。

**怎么办**：
1. 多阈值 + 时间窗口，别用单点值。
2. 加二次确认或用户手势。
3. 要真正稳定，只能上 UWB / CCC。

**预防**：产品和测试都要接受"RSSI 方案有抖动"，别把它当 bug 修。

### 10.10 问题十：以为 NearbyInteraction 就是 CarKey 的 UWB

**现象**：想用 NearbyInteraction 做防中继车钥匙，发现协议对不上。

**原理（新手版）**：NearbyInteraction 是第三方配件测距框架，不是 CCC 车钥匙协议，测距结果也不参与 Wallet 的钥匙认证。

**怎么办**：
1. 自研方案里，NI 只能做辅助测距。
2. 要 CCC 3.0 级别的抗中继，必须走 Wallet + 车企 MFi。

**预防**：选型时先确认商务资格，再定技术方案。

### 10.11 问题十一：强杀后有时能开、有时不能

**现象**：用户坚称每次都上划杀掉了 App，但有时拉门能开、上车也能启动，有时完全没反应。能开门的时候，启动也正常。

**原理（新手版）**：强杀只决定"iOS 是否愿意因蓝牙事件重启 App"，不决定车端是否立刻清空会话。常见原因有三类：

1. 其实没杀成，或者后来又打开过。App 已经被系统回收、不在多任务列表里时，上划那一下不产生"用户强杀"标记；你在看电量、开空调、查位置时打开过 App，也会恢复唤醒资格。
2. 车端会话缓存。刚杀完 App 的短时间内，车可能还保留最近一次认证会话；过一两分钟就失效。
3. 蓝牙链路质量。手机放裤兜、包里、身体遮挡、2.4 GHz 干扰，都会让 BLE 连接时好时坏。这一条和 App 是否被杀无关，但在体感上很难区分。广播、扫描、连接的时间尺度见 4.6。

**怎么办**：
1. 先确认 App 真的在多任务列表里被划掉，而不是已经被系统回收。
2. 下车走远，等车自动锁车，等 1–2 分钟再靠近测试，排除车端缓存。
3. 把手机从口袋或包里拿出来，靠近门把手再试，排除 BLE 链路问题。
4. 检查蓝牙、后台 App 刷新、低电量模式是否影响了 App 的后台活动。
5. 如果"开门"和"启动"总是一起成功或一起失败，重点查 App 状态、车端缓存和蓝牙链路；只有"开门成功、启动失败"时，才需要怀疑两段认证的强度不同。

**预防**：不要强杀 App；保持蓝牙和后台 App 刷新开启；给 App 必要的后台权限；出现问题时先按"是否真强杀 → 车是否已锁 → 手机位置 → 蓝牙状态"的顺序排查。

---

## 十一、高频速查表

### 11.1 TN3115 重启规则

| 状态 | 会重启 | 备注 |
|---|---|---|
| 挂起在内存 | 不需要重启 | 直接激活 |
| 从内存移除 | 会 | — |
| 崩溃 | 会 | 崩溃不等于强杀 |
| 用户强杀 | 不会 | iOS 26 起 AccessorySetupKit App 例外 |
| 设置里关蓝牙 | 不会 | — |
| 控制中心关蓝牙 | 会 | 注 5 同时适用 |
| 飞行模式 | 会 | 前提是蓝牙没被一起关 |
| 设备重启 | 会 | 首次解锁前不会 |

### 11.2 后台模式对照

| 模式 | 角色 | 能做什么 | 典型用途 |
|---|---|---|---|
| `bluetooth-central` | 中心 | 后台扫描、发现、收发 GATT | 车钥匙、手环、传感器 |
| `bluetooth-peripheral` | 外围 | 后台广播、被连接、响应读写 | 手机当信号源 |
| 未声明 | 都不是 | 退后台即停 | 纯前台工具 |

### 11.3 状态恢复 API 与恢复字典

| 名称 | 作用 |
|---|---|
| `CBCentralManagerOptionRestoreIdentifierKey` | 初始化时登记恢复标识 |
| `centralManager(_:willRestoreState:)` | 系统重启 App 时的唯一恢复入口 |
| `CBCentralManagerRestoredStatePeripheralsKey` | 恢复的外围列表 |
| `CBCentralManagerRestoredStateScanServicesKey` | 恢复的扫描服务 UUID |
| `CBCentralManagerRestoredStateScanOptionsKey` | 恢复的扫描参数 |
| `CBCentralManagerOptionShowPowerAlertKey` | 蓝牙不可用时是否弹系统提示 |

### 11.4 三条路线选型

| 你的处境 | 建议路线 |
|---|---|
| 车企或有车企深度合作，要无感、防中继、没电也能开 | CCC + Apple Wallet CarKey |
| 普通开发者，要快速上线，能接受后台存活和 RSSI 抖动 | 自研 BLE + 服务器下发密钥 |
| 只要远程开关锁、提前开空调 | 4G / 云通道 |
| 在欧洲经济区，要做手机当卡被读卡器读 | `CardSession` + HCE entitlement |

### 11.5 ISO 7816 常见状态字

| SW1 SW2 | 含义 | 处理建议 |
|---|---|---|
| `90 00` | 成功 | 继续业务流程 |
| `6A 82` | 找不到应用或文件 | 核对 AID，不要重试 |
| `6A 86` | P1/P2 参数错误 | 检查指令参数 |
| `6D 00` | 不支持该指令 | 检查 INS |
| `67 00` | 长度错误 | 检查 Lc / Le |
| `69 85` | 使用条件不满足 | 检查前置状态 |

### 11.6 Info.plist 与授权速查

| 名称 | 类型 | 用途 |
|---|---|---|
| `NSBluetoothAlwaysUsageDescription` | Info.plist | 蓝牙权限文案，iOS 13+ 必须 |
| `NSBluetoothPeripheralUsageDescription` | Info.plist | 旧系统兼容（deployment target < iOS 13） |
| `UIBackgroundModes` | Info.plist | `bluetooth-central` / `bluetooth-peripheral` |
| `NFCReaderUsageDescription` | Info.plist | NFC 读卡权限文案 |
| `com.apple.developer.nfc.readersession.formats` | Entitlement | NFC 读卡会话 |
| `com.apple.developer.nfc.readersession.iso7816.select-identifiers` | Info.plist | ISO 7816 AID 白名单 |
| `com.apple.developer.carkey.session` | Entitlement | CarKey 框架，仅 MFi 车企 |
| `com.apple.developer.nfc.hce` | Entitlement | EEA 的 HCE 卡模拟 |

---

## 十二、能力清单与选型

### 12.1 开发者能落地什么

| 能做 | 说明 |
|---|---|
| App 内 BLE 无感解锁、远程控车 | 自研路线的主战场 |
| Core NFC 读卡、读 APDU | 诊断、读标签、读车辆 NFC 卡原始数据 |
| 钥匙管理 UI | 分享状态、有效期、权限展示；最终动作交 Wallet |
| 停车位置、行程、状态同步 | 车机与手机的辅助功能 |
| EEA 内的 HCE 卡模拟 | 需要 entitlement，场景含车钥匙 |

| 做不了 | 原因 |
|---|---|
| 自建一张能加进 Wallet 的车钥匙 | 签发权在车企 + Apple |
| 调用 CarKey 的 UWB 安全测距 | 系统层通道，第三方拿不到 |
| 后台静默读 NFC | 系统限制 |
| 绕过用户强杀 | TN3115 规则，无代码解 |

### 12.2 一句话选型

想要"贴一下就开、没电也能开、不怕中继"，只能走 CCC + Wallet，而且必须车企主导。想要"快、通用、不依赖车企认证"，就自研 BLE，接受后台存活和 RSSI 的抖动。两条路都能做，但别指望用 B 方案拿到 A 的安全等级。

---

## 十三、踩坑清单

1. 以为"App + 蓝牙"就等于数字钥匙，忽略了 SE 与 Wallet 托管的差别。
2. 以为 CCC 联盟会员 = 车支持 CCC 钥匙。会员、认证、车型落地是三件事。
3. 以为 iOS 完全没有 HCE，直接从安卓方案照搬。iOS 17.4 起在 EEA 可用，但需要 HCE entitlement；其他地区仍不行。
4. ISO 7816 场景忘记配 AID 白名单，标签死活扫不到。
5. 把 `6A82` 当通信故障反复重试。
6. NFC 会话超时没处理，握手到一半会话已失效（阅读器会话约 60 秒）。
7. 蓝牙钥匙没做状态恢复（state restoration），App 被回收后无感解锁时灵时不灵。
8. 恢复标识每次启动都换新，系统对不上账。
9. 只恢复扫描不重建状态机，App 半醒。
10. 用 RSSI 阈值做"靠近即解锁"，没做人体遮挡与机型差异的标定。
11. 把 NearbyInteraction 误当成 CarKey 的 UWB 能力。
12. 用普通 Pass（静态卡券）去仿车钥匙，安全模型不成立。
13. 在 `willRestoreState` 里做重活，十秒预算烧完被杀。
14. 连着 Xcode 调试器测唤醒，造不出真实的挂起和回收。

---

## 附录：实操入口地图

| 要做的事 | 位置（界面 / 命令） | 操作说明 |
|---|---|---|
| 新建 iOS 工程 | Xcode → File → New → Project → iOS → App | Interface 选 SwiftUI 或 Storyboard 均可 |
| 打开蓝牙后台模式 | target → Signing & Capabilities → + Capability → Background Modes | 勾 Uses Bluetooth LE accessories |
| 加蓝牙权限文案 | Info.plist → 右键 Add Row → Privacy - Bluetooth Always Usage Description | 写真实用途，审核会看 |
| 加 NFC 权限文案 | Info.plist → Privacy - NFC Scan Usage Description | 读卡场景必填 |
| 配 AID 白名单 | Info.plist → `com.apple.developer.nfc.readersession.iso7816.select-identifiers` | 数组，填 AID |
| 看真机日志 | macOS Console.app → 左侧选设备 → 搜索 `bluetoothd` | 也可用 Xcode → Window → Devices and Simulators → Open Console |
| 造 BLE 外设 | 第二台 iPhone → nRF Connect → Advertiser → 添加 Service UUID → Start | UUID 要和代码一致 |
| 模拟"从内存移除" | Xcode 工具栏 Stop | 不是多任务上划，上划是强杀 |
| 模拟"强杀" | iPhone 多任务界面 → 上划 App | 预期不会再被唤醒 |
| 查 App 是否被拉起 | Xcode → Debug → Attach to Process | 或直接看 Console 的 relaunch 日志 |

**排障"动手顺序"速查**：

```
① 先看权限：Info.plist 有没有 NSBluetoothAlwaysUsageDescription，设置里蓝牙开没开
② 再看扫描：前台能不能发现设备，service UUID 对不对
③ 再看唤醒：Console 里有没有 bluetoothd 的 relaunch 记录
④ 再看恢复：有没有 willRestoreState，restore identifier 稳不稳定
⑤ 最后看业务：恢复后第一次连接 / 认证 / 解锁有没有成功
```

---

## 官方与参考链接

- [TN3115: Bluetooth State Restoration app relaunch rules](https://developer.apple.com/documentation/technotes/tn3115-bluetooth-state-restoration-app-relaunch-rules)（重启规则原文，2025-09-15 更新）
- [CBCentralManagerOptionRestoreIdentifierKey](https://developer.apple.com/documentation/corebluetooth/cbcentralmanageroptionrestoreidentifierkey)（恢复标识与 scene-based App 说明）
- [Central Manager State Restoration Options](https://developer.apple.com/documentation/corebluetooth/central-manager-state-restoration-options)（恢复字典与示例代码）
- [centralManager(_:willRestoreState:)](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/centralmanager(_:willrestorestate:))（恢复入口）
- [Core Bluetooth Programming Guide: Background Processing for iOS Apps](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html)（后台模式、10 秒预算、扫描差异）
- [Core Bluetooth 框架首页](https://developer.apple.com/documentation/corebluetooth)（iOS 26 Live Activity 例外）
- [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification/)（广播、扫描、连接间隔的规范范围）
- [NSBluetoothAlwaysUsageDescription](https://developer.apple.com/documentation/BundleResources/Information-Property-List/NSBluetoothAlwaysUsageDescription)（蓝牙权限键）
- [CarKey 框架](https://developer.apple.com/documentation/carkey)（公开文档与 MFi 授权要求）
- [Core NFC CardSession](https://developer.apple.com/documentation/CoreNFC/CardSession)（EEA 的 HCE 卡模拟）
- [HCE-based contactless NFC transactions in the EEA](https://developer.apple.com/support/hce-transactions-in-apps/)（HCE 资质与申请入口）
- [AccessorySetupKit](https://developer.apple.com/documentation/accessorysetupkit)（iOS 18+ 配件配对）
- [NearbyInteraction](https://developer.apple.com/documentation/nearbyinteraction)（第三方 UWB 配件测距）
- [WWDC20 Session 10006: Introducing Car Keys](https://developer.apple.com/videos/play/wwdc2020/10006/)（CarKey 面向车企的介绍）
- [WWDC21 Session 10084: Explore UWB-based Car Keys](https://developer.apple.com/videos/play/wwdc2021/10084/)（UWB 车钥匙）
- [CCC Digital Key Release 3.0 新闻稿](https://carconnectivity.org/car-connectivity-consortium-publishes-digital-key-release-3-0-businesswire/)（2021-07-13 定稿）
- [CCC Digital Key Certification: BMW + NXP](https://carconnectivity.org/bmw-group-and-nxp-become-the-first-manufacturers-to-achieve-ccc-digital-key%EF%B8%8F-certification/)（2024-07-17 首批认证）
- [CCC Digital Key 认证 2025 年进展](https://carconnectivity.org/ccc-digital-key-certifications-surged-in-2025/)（新增认证清单）
- [CCC Digital Key Version 4 白皮书](https://carconnectivity.org/whitepaper/ccc-digital-key-v4/)（分享、车队、租赁方向）
- [BMW 与苹果联合推出适用于 iPhone 的数字钥匙（2020-06，中文）](https://www.press.bmwgroup.com/china/article/detail/T0310468ZH_CN/%E5%AE%9D%E9%A9%AC%E4%B8%8E%E8%8B%B9%E6%9E%9C%E8%81%94%E5%90%88%E6%8E%A8%E5%87%BA%E9%80%82%E7%94%A8%E4%BA%8Eiphone%E6%89%8B%E6%9C%BA%E7%9A%84%E6%95%B0%E5%AD%97%E9%92%A5%E5%8C%99?language=zh_CN)
- [Apple Platform Security: Car key security in iOS](https://support.apple.com/zh-cn/guide/security/secf64471c16/web)（配对、安全通道、共享）
- [Tesla Model Y 车主手册：钥匙](https://www.tesla.cn/ownersmanual/modely/zh_cn/GUID-E004FAB7-1C71-448F-9492-CACF301304D2.html)（Phone Key 行为）
- [Tesla 服务文档：Bluetooth 排障](https://service.tesla.com/docs/Public/diy/model3/en_us/GUID-D1B313C0-1FE6-4841-AEE2-661ACBDF3838.html)（要求 App 后台运行）
