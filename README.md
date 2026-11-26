# 我的 AI 编程 Skill

个人 Trae skill 仓库——收录两个 skill：`beginner-tech-doc`（技术问题 → 搜索验证 → 新手友好文档）和 `spec-grill-me`（方案拷问与批量决策收敛）。

## beginner-tech-doc 是什么

触发后，agent 会**先用网络搜索验证事实**（版本、政策、菜单路径、错误码等易过时信息），再生成一份**新手友好、可照着实操**的中文技术文档：

- 比喻地图开路，全局直觉先行
- 专业名词中英对照 + 大白话解释
- 每个实操步骤写明"在哪个界面点哪个按钮"
- 报错三件套：现象原文 → 原理 → 按序解法
- 全景 SOP（含第 0 步前置条件：账号/费用/资质/工具安装/首次启动/模块/磁盘/网络）
- 文档头写明适用版本与文档边界，经验性内容标注可靠性边界
- 业务内容只讲与技术交接相关的角色、流程与责任边界，点到为止
- 支持增量更新已有文档（章节编号与交叉引用自动同步）
- 交付前逐项卡点自检，保证新手不在任何一步卡住

触发词：用户提出技术知识类问题，或要求"新手教程 / 知识体系文档 / 实操指南"。

## spec-grill-me 是什么

当你带着一个**还没定完的方案 / 设计 / 计划**，要求"拷问我 / grill me / 帮我把方案想清楚"时触发。agent 会：

1. 先自己调查代码和文档，只把真正的决策点留给你（查得到的事实不问）；
2. 按依赖层级**批量提问**（每批 3–6 个），每题给 2–4 个选项 + 明确推荐 + 依据 + 代价 + 置信度；
3. 默认把推荐项作为"待确认建议"继续推演，不逐题等待回答；
4. 输出一份完整建议列表，你一次性确认或按编号修正——修正任一项会自动重算下游依赖项；
5. 只有你明确确认后，建议才升级为"已确认决策"。

适合：架构选型、方案评审前的决策收敛。不适合：直接执行任务、普通问答、只要一条评审结论。

## 安装（新电脑）

把 `beginner-tech-doc` 和 `spec-grill-me` 文件夹复制到本机 skills 目录，重启 Trae 即生效：

```powershell
# CN 版 Trae
git clone https://github.com/Mianbao4399/my-ai-coding-skill.git
Copy-Item -Recurse .\my-ai-coding-skill\beginner-tech-doc, .\my-ai-coding-skill\spec-grill-me "$env:USERPROFILE\.trae-cn\skills\"

# 国际版 Trae（目录是 .trae 不是 .trae-cn）
Copy-Item -Recurse .\my-ai-coding-skill\beginner-tech-doc, .\my-ai-coding-skill\spec-grill-me "$env:USERPROFILE\.trae\skills\"
```

## 更新

```powershell
git pull
Copy-Item -Recurse -Force .\beginner-tech-doc, .\spec-grill-me "$env:USERPROFILE\.trae-cn\skills\"
```

## 提交信息规范

提交信息用 `<类型>(<范围>): <摘要>`，摘要中文动词开头、不超过 40 个汉字，细节放正文。

| 类型 | 用途 |
|---|---|
| `docs` | 知识文档、README |
| `skill` | skill 规则、模板、安装说明 |
| `chore` | 仓库维护 |
| `fix` | 修正错误 |

仓库根目录提供了 `.gitmessage` 模板，本地启用：

```powershell
git config commit.template .gitmessage
```

示例：

```text
docs(数字车钥匙): 收录 iOS 数字车钥匙与蓝牙远程唤醒知识体系

- 覆盖三条路线、CCC/CarKey、NFC/UWB/BLE 分工
- 补充状态恢复、TN3115、中继攻击与强杀排查
```

## 文件结构

```
beginner-tech-doc/
├── SKILL.md                 # 技能指令（工作流 + 内容规则 + 卡点自检）
└── assets/
    └── doc-template.md      # 文档标准结构模板

spec-grill-me/
└── SKILL.md                 # 技能指令（批量追问 → 建议列表 → 确认收敛）

docs/                                # 本仓库 skill 产出的知识文档（按领域分文件夹，含官方来源清单）
├── 车务Agent项目总览与路线图.md        # 车务 Agent 项目跨领域总纲：架构 / 协议 / 里程碑 / 风险
├── 前端/                              # App 端
│   ├── Flutter聊天窗口与Agent接入知识体系.md   # 当前项目（Flutter 跨端）：WebSocket / 流式渲染 / 卡片
│   └── iOS*.md                        # 上一代 iOS 项目知识沉淀（7 篇）
├── agent/                             # Agent 服务（Python + FastAPI）
│   └── Agent服务核心设计与实现知识体系.md       # 核心循环 / 工具系统 / DeepSeek / SOP
└── 后端/                              # 业务接口侧
    └── 业务API工具化对接知识体系.md             # OBD / 知识库 RAG / 商城的工具化契约
```

## 知识文档（docs/ 目录）

`beginner-tech-doc` skill 实际产出的成果，关键事实均经网络核验（各文档文内标注核验日期，文末附官方与参考链接；经验性内容标注可靠性边界）。按领域分文件夹组织：

**根目录（跨领域总纲）**

- **车务 Agent 项目总览与路线图**——项目定位 / 三端架构 / 消息协议 / 关键决策 / 技术选型 / M1~M4 里程碑与任务分解 / 协作约定 / 风险清单

**前端/**

- **Flutter 聊天窗口与 Agent 接入知识体系**——当前项目（Flutter 跨端）：聊天页难点 / 七种消息渲染 / WebSocket 心跳退避缓冲 / 流式渲染 / 三种卡片 / confirm 流程 / 输出安全 / SOP 与排障
- **iOS 签名与出包知识体系**——Mach-O / 证书 / 签名 / 描述文件 / ipa / 出包三条路 / 事故排查 / 上架全景 SOP
- **iOS 游戏工程与出包知识体系**——四大引擎导出、运行原理、从零到真机 SOP、签名/崩溃/性能排查、研发与发行协作
- **iOS 与 RN 桥接知识体系**——双世界模型与线程、旧桥/新架构（JSI/TurboModule/Codegen）双语实操、0.75+iOS 15 版本适配、原生 SDK 封装实战与升级 SOP
- **iOS 数字车钥匙与蓝牙远程唤醒知识体系**——三条路线选型、CCC 与 Apple Wallet CarKey、NFC/UWB/BLE 三通道分工、CoreBluetooth 状态恢复与 TN3115 规则、最小可跑 Demo 与排障速查
- **iOS 蓝牙设备连接与 OTA 固件升级知识体系**——连接后工程实践 / 断开重连状态机 / 指数退避 / OTA（DFU）全流程 / Nordic DFU 集成 / 排障与速查
- **iOS 内存与性能调优知识体系**——内存布局 / ARC 与引用关系 / 循环引用与泄漏排查 / Memory Graph 与 Instruments 四件套 / 启动、卡顿、包体积优化 / 性能回归 SOP
- **iOS 开发语言与运行时知识体系**——OC / Swift / C++ 选型与混编 / OC Runtime（消息转发、Method Swizzling、KVO）/ Swift 派发与 Swift 6 并发 / 与 UIKit/SwiftUI 的协作

**agent/**

- **Agent 服务核心设计与实现知识体系**——当前项目（Python + FastAPI + DeepSeek）：核心循环与安全阀 / 工具系统 / LLM 抽象层 / 上下文管理 / FastAPI 服务化 / 响应速度优化 / 安全与防注入 / 从零 SOP / 报错排查

**后端/**

- **业务 API 工具化对接知识体系**——当前项目：三域接口最小契约 / 工具封装五条约定 / 鉴权透传与数据隔离 / 精确匹配 + RAG 双通道 / 下单安全 / 接入 SOP
