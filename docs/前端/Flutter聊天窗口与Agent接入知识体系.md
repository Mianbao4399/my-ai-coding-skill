# Flutter 聊天窗口与 Agent 接入知识体系（新手友好版）

> 覆盖：聊天窗口为什么特殊 / 消息协议（前端视角）/ WebSocket 接入与断线重连 / 流式渲染 / 三种卡片 Widget / 下单确认交互 / 状态管理选型 / 从零搭建 SOP / 报错排查 / 速查与踩坑
> 阅读方式：每个概念先讲"是什么（带比喻）"，再讲"为什么需要它"，最后讲"怎么用 / 怎么排查"。专业名词首次出现均做中英对照，并给一句大白话。
> 信息核验日期：2026-11-27（Flutter 3.50 与 Dart 3.14 经官方发布信息核验；依赖包版本以 pub.dev 页面为准，文末附链接）。
> 项目总纲见《车务 Agent 项目总览与路线图》；协议与后端行为见《Agent 服务核心设计与实现知识体系》。姊妹篇（上一代 iOS 项目知识）：蓝牙连接与 OTA、签名与出包等，见本目录其余文档。

---

## 目录

- [0. 先用一组比喻建立全局](#0-先用一组比喻建立全局)
- [一、聊天窗口为什么是 App 里最难的页面](#一聊天窗口为什么是-app-里最难的页面)
- [二、消息协议（前端视角）：七种消息怎么渲染](#二消息协议前端视角七种消息怎么渲染)
- [三、WebSocket 接入：对讲机的接法](#三websocket-接入对讲机的接法)
- [四、消息模型与流式渲染](#四消息模型与流式渲染)
- [五、卡片 Widget：三种卡怎么画](#五卡片-widget三种卡怎么画)
- [六、下单确认交互：confirm 流程](#六下单确认交互confirm-流程)
- [七、状态管理怎么选](#七状态管理怎么选)
- [八、实操全流程（SOP）：从零搭聊天页](#八实操全流程sop从零搭聊天页)
- [九、常见报错与排查](#九常见报错与排查)
- [十、高频速查表](#十高频速查表)
- [十一、踩坑清单](#十一踩坑清单)
- [附录：实操入口地图](#附录实操入口地图)
- [官方与参考链接](#官方与参考链接)

---

## 0. 先用一组比喻建立全局

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| 组件 | Widget | 乐高积木 | Flutter 里一切界面皆积木 |
| 状态 | State | 积木当前拼到哪一步 | 数据变了界面跟着变 |
| 有状态组件 | StatefulWidget | 会自己记事的积木 | 消息气泡需要记录"已拼到第几段" |
| 流 | Stream | 传送带上的包裹 | 服务端推送的数据一件件到达 |
| 长连接 | WebSocket | 双向对讲机 | 不用每次喊话都重新拨号 |
| 心跳 | Heartbeat Ping | 定期喊一声"还在吗" | 及时发现断线，别等消息丢了才知道 |
| 退避重连 | Backoff Reconnect | 没人接就隔久一点再拨 | 防止疯狂重拨拖垮服务端 |
| 消息模型 | Message Model | 快递的标准化面单 | 七种 JSON 消息解析成统一的 Dart 对象 |
| 反序列化 | Deserialization | 按面单拆包裹 | JSON 字符串还原成 Dart 对象 |
| 列表构建器 | ListView.builder | 滚到哪装到哪的流水线 | 长消息列表只渲染屏幕内的项 |
| 深链 | Deep Link | 卡片上的"直达传送门" | 点配件卡直接跳商城详情页 |
| 佚位键 | Key | 每块积木的身份证 | 流式更新时帮 Flutter 认出哪条消息是哪条 |

串起来就是一句话：**聊天页 = 一条对讲机长连接 + 一条消息解析流水线 + 一根滚不到头的列表，全部围绕"边收边画"三个字。**

---

## 一、聊天窗口为什么是 App 里最难的页面

普通页面是"请求一次 → 渲染一次"；聊天页是"持续接收 → 不停重绘"，难在三件事叠加：

1. **三种内容混排**：流式文字气泡、灰色过程提示、结构化卡片同住一个列表，每种的布局、生命周期、点击行为都不同；
2. **高频局部刷新**：`text_delta` 一秒可能来十几段，整列表无脑 setState 会卡；
3. **连接不可靠**：切后台、进电梯、弱网都会断线，断线期间的体验决定用户是否觉得"这功能坏了"。

对应的三条设计底线：消息统一抽象成一个模型再分流渲染；流式更新只刷新正在打字的那条气泡；断线重连 + 消息缓冲是必修功能而不是优化项（M2 验收项）。

---

## 二、消息协议（前端视角）：七种消息怎么渲染

协议合同见总览篇第三章。前端关心的是**每种的渲染动作**：

| type | 渲染动作 | 要点 |
|---|---|---|
| `text_delta` | 追加到"正在生成"的气泡 | 高频消息，必须局部刷新 |
| `tool_status` | 灰色小字插入列表 | "正在查询故障记录…"，无需交互 |
| `card` | 按 `cardType` 分流到三种卡片 Widget | 承载点击跳转，数据要校验 |
| `confirm` | 弹出确认框 | 必须带 `confirmId`，超时要失效 |
| `error` | 气泡内红字 + 重试按钮 | 重试即重发上一条 user_message |
| `user_message` | 自己发的，右侧气泡 | 本地先渲染，勿等回显 |
| `confirm_result` | 不渲染 | 只更新确认框状态 |

三种卡片的 schema 要点：

| cardType | 必须字段 | 交互 |
|---|---|---|
| `fault`（故障记录卡） | 故障码、描述、发生时间、严重度 | 点击展开冻结帧详情 |
| `knowledge`（知识解读卡） | 结论、严重等级、建议措施、出处 | "出处"必须展示，抑制幻觉 |
| `part`（配件商品卡） | 名称、品牌、价格、适配 VIN、buyUrl | 点"去购买"跳 `buyUrl` 深链 |

**前端是协议的第一道守门员**：解析时字段缺失就渲染降级样式（如"数据异常"占位卡）并上报埋点，不要让坏数据崩掉整个页面。

---

## 三、WebSocket 接入：对讲机的接法

### 3.1 是什么与用什么包

WebSocket 是双向长连接：拨通一次（握手升级），之后双方随时互发消息。Flutter 官方生态用 `web_socket_channel` 包（版本以 pub.dev 为准）。连接时把用户 token 放在查询参数里做鉴权透传：

```dart
final channel = WebSocketChannel.connect(
  Uri.parse('ws://10.0.2.2:8000/ws/chat?token=$userToken'),
  // Android 模拟器里 10.0.2.2 才是宿主机 localhost
);
channel.stream.listen(onServerMessage, onError: onServerError);
```

### 3.2 连接生命周期：心跳、重连、缓冲

裸连接在线上一定掉，生命周期管理是接入的真正难点：

```
连接中 → 已连接 ──收发消息──► 运行中 ──心跳超时/报错──► 断开
                    ▲                            │
                    └──── 指数退避重连(1s→2s→4s…上限30s) ◄┘
```

三个必备件：

- **心跳（heartbeat）**：每 30 秒发一次 ping 消息，两个周期没收到任何下行（含 pong）判定假在线，主动断开重连；
- **退避重连（backoff reconnect）**：断开后从 1 秒开始翻倍重试，上限 30 秒；连上后计时器归零。避免弱网下疯狂重拨；
- **发送缓冲（outbox buffer）**：断线期间用户发的消息先入本地队列，重连成功后按序补发。用户感受是"发出去就走了"，不感知断线。

页面生命周期也要接：App 进后台暂停心跳，回前台立刻检查连接并补发缓冲。

---

## 四、消息模型与流式渲染

### 4.1 消息模型：一个基类分流七种消息

按面单拆包裹：服务端的每条 JSON 先解析成统一的 `ChatMessage` 家族，列表渲染时按子类型分流。Dart 里用密封类（sealed class）表达这个家族最合适：

```dart
sealed class ChatMessage { String get id; }

class TextMsg extends ChatMessage {        // 文字气泡（含流式生成中的）
  final String content;
  final bool streaming;                    // true = 还在蹦字
  TextMsg({required this.content, required this.streaming, required super.id});
}

class CardMsg extends ChatMessage {        // 卡片
  final String cardType;                   // fault / knowledge / part
  final Map<String, dynamic> payload;
  CardMsg({required this.cardType, required this.payload, required super.id});
}

class StatusMsg extends ChatMessage {      // tool_status 灰色提示
  final String text;
  StatusMsg({required this.text, required super.id});
}
```

解析层职责单一：不认识的 `type` 丢弃并埋点，认识的转成对应子类。**渲染层永远只面对 Dart 对象，不面对原始 JSON**。

### 4.2 text_delta 的合并策略

服务端按"段"推送，前端要按"条消息"合并：收到 `text_delta` 时，若其 `messageId` 对应的气泡已在列表尾部，只更新该气泡的 `content` 并标 `streaming=true`；收到该消息的结束信号才置 `streaming=false`。关键约束：**只 rebuild 这一格，不 rebuild 整个列表**（给每条消息套独立的 Consumer/局部 State，佚位键用服务端的 messageId，保证 Flutter 能认出"还是同一条"）。

### 4.3 列表与自动滚动

用 `ListView.builder`（滚到哪构建哪），`reverse: true` 是聊天页惯例——列表反转后天然从底部开始，新消息落底且无需手动滚动锚定，还避开了键盘弹出时的滚动错乱。流式期间的自动跟随做成可选：用户一旦手动上滑回看历史，就暂停自动跟随并浮出"回到底部"按钮。

---

## 五、卡片 Widget：三种卡怎么画

卡片是协议里"给 UI 的数据"，与气泡混排在一个列表里。三张卡的实现要点：

| 卡 | 结构 | 易错点 |
|---|---|---|
| fault | 顶部故障码大字 + 严重度色块 + 可展开详情 | 严重度颜色只用设计规范定义的几档，别自己发散 |
| knowledge | 结论段 + 建议列表 + 底部"出处"链接 | 出处为空时显示"暂无出处"而不是隐藏（让空洞可见） |
| part | 图 + 名称品牌 + 价格 + 两颗按钮（去购买/问详情） | 价格格式化统一走工具函数；buyUrl 用深链路由注册 |

"问详情"按钮的价值容易被低估：它把卡片操作拉回对话流——点击后本地插入一条 `user_message`（"详细说说这个配件"），Agent 自然接话。**卡片和对话互为入口**，这是聊天式购物体验的核心。

---

## 六、下单确认交互：confirm 流程

`confirm` 消息的交互流程是个小型状态机：

```
收到 confirm → 弹底部确认框(展示商品/金额) → 用户选择
   ├─ 确认 → 上行 confirm_result{confirmId, approved:true} → 等 Agent 下发订单结果
   └─ 取消 → 上行 confirm_result{confirmId, approved:false}
任何分支都要处理：confirm 超时(如 60s)自动失效并提示重新发起
```

两条红线：确认框**必须展示金额与商品明细**，不允许只弹"是否继续"；确认框期间禁止重复弹出第二张（以 confirmId 去重）。前端不自己判断"这个操作危不危险"，服务端说危险才弹。

---

## 七、状态管理怎么选

聊天页的状态就三样：消息列表、连接状态、待确认项。规模判断：**一套 ChangeNotifier（或 Provider）足够撑起整个聊天模块**，不必上重型框架。推荐拆分：

- `ChatSessionNotifier`：持有消息列表，唯一写入口是"收到消息 / 发出消息"两个方法；
- `ConnectionNotifier`：连接状态机（连接中/在线/重连中/离线），驱动顶栏的状态提示；
- 页面 Widget 只读不写。

如果项目已有既定的状态管理方案（Riverpod / Bloc），聊天模块跟随项目惯例优先于本篇推荐——**一致性比局部最优更重要**。

---

## 八、实操全流程（SOP）：从零搭聊天页

> 本节是"执行线"。目标是 M2 的前端部分：真机上与 Agent 服务完成"查故障 → 解读 → 配件卡"全流程。

### 8.0 全景步骤总览（收藏这张）

```
第 0 步 [环境检查] → 第 1 步 [建项目加依赖] → 第 2 步 [消息模型与解析]
→ 第 3 步 [连接管理器] → 第 4 步 [气泡列表] → 第 5 步 [卡片]
→ 第 6 步 [确认流程] → 第 7 步 [联调]
```

### 8.1 第 0 步：环境检查

装好 Flutter SDK 后先体检（核验时最新稳定版为 Flutter 3.50 + Dart 3.14，以官方发布页为准）：

```powershell
flutter doctor          # 体检：列出缺什么（Android SDK / Xcode / 证书）
flutter --version       # 确认版本
```

`flutter doctor` 打叉的项按提示逐个补齐，Android 开发补 Android Studio 与 SDK，iOS 开发补 Xcode（macOS 专属）。

### 8.2 第 1 步：建项目加依赖

```powershell
flutter create vehicle_assistant
cd vehicle_assistant
flutter pub add web_socket_channel provider
```

### 8.3 第 2 步：消息模型与解析

写 4.1 节的 sealed class 家族 + `ChatMessage parse(Map json)` 工厂函数。先用单元测试喂七种样例 JSON（含一种未知 type 和一种缺字段卡片），全绿再往下走——解析器是整页的地基。

### 8.4 第 3 步：连接管理器

按 3.2 节状态机实现 `ChatConnection`：连接、心跳、退避重连、发送缓冲。对着 Agent 服务还没起的情况下，先用 `echo` 服务（如 websocket.org 的测试端点或本地 echo 脚本）把状态机跑熟。

### 8.5 第 4 步：气泡列表

`ListView.builder` + `reverse: true`，先只支持 TextMsg 与 StatusMsg。联调 Agent 服务（本地起 uvicorn），验证流式蹦字与局部刷新。此时按 M1 协作约定，前端可以先行——Agent 服务哪怕只有 mock 工具也能配合联调。

### 8.6 第 5 步：卡片

实现三种卡片 Widget 并接入 `card` 分流。与设计对齐卡片的视觉规范；价格、日期格式化统一走工具函数。

### 8.7 第 6 步：确认流程

按第六章状态机实现确认框与 confirm_result 上行，联调商城"创建订单"危险工具（M3 与后端约好联调窗口，注意用测试环境）。

### 8.8 第 7 步：联调验收

对照 M2 完成标志逐项过：真机弱网（开飞行模式再恢复）验证重连与缓冲；连发 20 条消息验证列表流畅；杀进程重进验证会话恢复。

---

## 九、常见报错与排查

共同规律：**先分清是"连不上"还是"连上但消息不对"**——前者查连接层，后者查解析层，别混着调。

### 9.1 事故一：WebSocket 连接被拒

**现象**：

```text
WebSocketChannelException: WebSocket connection failed
```

**原理（新手版）**：对讲机没拨通。Android 模拟器里 `localhost` 不是宿主机（要用 10.0.2.2）；iOS 模拟器用宿主机 IP；真机必须同网段或公网地址；明文 ws 在 iOS 上默认被 ATS（App Transport Security，苹果的强制加密策略）拦截。

**怎么办**：
1. 核对地址：模拟器用 10.0.2.2（Android）/ 宿主机 IP（iOS），真机用局域网 IP；
2. Android 检查 `AndroidManifest.xml` 的网络权限与明文流量配置（debug 才允许 ws）；
3. iOS 用 https/wss 或在 debug 配置里加 ATS 例外。

**预防**：环境地址（dev/test/prod）做成构建配置注入，不硬编码。

### 9.2 事故二：JSON 解析异常

**现象**：

```text
type 'Null' is not a subtype of type 'String'
```

**原理**：消息字段和模型定义不一致——服务端改了字段名或返回了 null，反序列化在拿值时炸了。

**怎么办**：
1. 把原始 JSON 打进日志，对照协议文档定位差异字段；
2. 解析器对可空字段显式判空并给默认值；
3. 与后端确认是协议变更还是 bug，改完同步更新协议文档版本号。

**预防**：解析层单元测试覆盖七种消息 + 异常样例（未知 type / 缺字段 / null）；协议字段变更必须三端评审（总览篇第七章）。

### 9.3 事故三：流式输出时页面卡顿

**现象**：蹦字阶段掉帧，低端机更明显。

**原理**：每个 text_delta 都 rebuild 了整个消息列表，或整列表没有 itemBuilder 懒加载。

**怎么办**：
1. 确认列表用 `ListView.builder` 且每条消息是独立的最小 rebuild 单元；
2. delta 合并做节流：同一气泡的更新每帧最多应用一次（收到多次 delta 攒一帧）；
3. 长文本气泡生成中先截断显示尾部（如"……"保留最后 500 字），生成完毕再放全文。

**预防**：性能验收写进 M2 清单：低端真机连发 20 条消息不掉帧。

### 9.4 事故四：键盘遮挡与溢出报错

**现象**：输入框聚焦时键盘盖住输入框，或控制台红字 `A RenderFlex overflowed`。

**原理**：键盘弹出压缩了可用高度，固定布局被挤断。

**怎么办**：外层包 `resizeToAvoidBottomInset` 默认行为并确认 Scaffold 生效；输入栏用 `SafeArea` + `viewInsets` 适配；列表被压缩后确认仍锚定底部（reverse 列表天然满足）。

**预防**：聊天页布局验收必须包含"键盘弹出 / 表情面板切换 / iOS 底部安全区"三个场景。

### 9.5 事故五：断线后消息"失踪"

**现象**：弱网下用户发的消息没回复，也没报错，像石沉大海。

**原理**：消息发出后连接恰好断了，没有发送缓冲，消息既没到服务端也没回补。

**怎么办**：
1. 确认连接管理器的发送缓冲（outbox）已实现并在重连后补发；
2. 每条上行消息带客户端消息 ID，服务端做幂等，避免补发导致重复回复；
3. 界面上把"发送中 / 已送达 / 失败可重发"状态画出来，让消息去向可见。

**预防**：把 3.2 节的三件套（心跳/退避/缓冲）作为 M2 验收项逐条打钩。

---

## 十、高频速查表

**七种消息渲染速查**：

| type | 渲染 | 局部刷新 | 可交互 |
|---|---|---|---|
| text_delta | 追加到生成中气泡 | 必须 | 无 |
| tool_status | 灰色小字 | 否 | 无 |
| card | 三种卡片 Widget | 否 | 点击/深链 |
| confirm | 底部确认框 | — | 确认/取消 |
| error | 红字气泡+重试 | 否 | 重试 |
| user_message | 右侧气泡 | 否 | 长按撤回(可选) |
| confirm_result | 不渲染 | — | 无 |

**连接管理三件套参数默认值**：

| 件 | 默认值 | 说明 |
|---|---|---|
| 心跳间隔 | 30s | 两周期无下行判假在线 |
| 重连退避 | 1s 起翻倍，上限 30s | 连上归零 |
| 确认框超时 | 60s | 超时失效需重新发起 |

**常用依赖**（版本以 pub.dev 为准）：

| 包 | 用途 |
|---|---|
| web_socket_channel | WebSocket 客户端 |
| provider / riverpod | 状态管理（跟随项目惯例） |
| flutter_markdown | 渲染含格式的助手回复（按需） |

---

## 十一、踩坑清单

1. **不要自己渲染 Markdown 来"升级"气泡**。一期协议里助手回答是纯文本流；如果决定上 Markdown，必须同步约定"卡片消息才是结构化内容的唯一出口"，两条路并行会打架。
2. **text_delta 千万别 setState 整页**。卡顿事故三的根因十有八九是这个。
3. **卡片数据不校验直接用会闪退**。payload 是 Map，字段缺失判空降级，前端是协议守门员。
4. **消息 ID 必须用服务端的 messageId 做佚位键**。自己用列表下标当 key，流式更新时 Flutter 会认错积木，出现重复气泡或状态串位。
5. **真机与模拟器网络行为不一样**。模拟器全绿不代表真机能连，M2 联调必须上真机 + 弱网场景。
6. **确认框不做 confirmId 去重会双重下单**。服务端有幂等兜底，但前端仍要在入口拦住重复弹窗。

---

## 附录：实操入口地图

| 要做的事 | 位置 | 操作说明 |
|---|---|---|
| 环境体检 | 终端 `flutter doctor` | 打叉项按提示修复 |
| 加依赖 | 终端 `flutter pub add <包名>` | 版本以 pub.dev 页面为准 |
| 起本地 Agent 服务 | agent 服务目录 `uvicorn main:app --reload --port 8000` | 联调前先起 |
| 模拟器访问宿主机 | Android 用 10.0.2.2，iOS 用宿主机局域网 IP | localhost 在 Android 模拟器里指向模拟器自身 |
| 看原始消息流 | 连接管理器里加日志打印每条收发的 JSON | 排查协议问题的第一现场 |
| 弱网测试 | 真机开飞行模式数秒再恢复 / 开发者选项限速 | M2 验收必过场景 |

**排障"动手顺序"速查**（收藏）：

```
① 先分诊：连不上（看连接日志/地址/权限）还是消息不对（看原始 JSON）
② 解析层单测喂异常样例定位字段差异
③ 性能问题先确认是否整页 rebuild
④ 弱网问题必上真机复现
⑤ 最后才怀疑服务端（用 wscat 直连对比）
```

---

## 官方与参考链接

- Flutter 官网（安装与环境）：https://flutter.dev
- Flutter 发布信息（版本动态以官方为准，核验时 3.50 / Dart 3.14）：https://docs.flutter.dev/release/whats-new
- Flutter 3.50 发布说明：https://flutter.dev/blog/whats-new-in-flutter-3-50
- Dart 官网：https://dart.dev
- web_socket_channel（pub.dev，版本以页面为准）：https://pub.dev/packages/web_socket_channel
- provider（pub.dev）：https://pub.dev/packages/provider
- Flutter 列表与性能官方文档：https://docs.flutter.dev/perf
