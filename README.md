# 我的 AI 编程 Skill

个人 Trae skill 仓库——收录两个 skill：`beginner-tech-doc`（技术问题 → 搜索验证 → 新手友好文档）和 `spec-grill-me`（方案拷问与批量决策收敛）。

## beginner-tech-doc 是什么

触发后，agent 会**先用网络搜索验证事实**（版本、政策、菜单路径、错误码等易过时信息），再生成一份**新手友好、可照着实操**的中文技术文档：

- 比喻地图开路，全局直觉先行
- 专业名词中英对照 + 大白话解释
- 每个实操步骤写明"在哪个界面点哪个按钮"
- 报错三件套：现象原文 → 原理 → 按序解法
- 全景 SOP（含第 0 步前置条件：账号/费用/资质）
- 支持增量更新已有文档（章节编号与交叉引用自动同步）
- 交付前 7 项卡点自检，保证新手不在任何一步卡住

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

## 文件结构

```
beginner-tech-doc/
├── SKILL.md                 # 技能指令（工作流 + 内容规则 + 卡点自检）
└── assets/
    └── doc-template.md      # 文档标准结构模板

spec-grill-me/
└── SKILL.md                 # 技能指令（批量追问 → 建议列表 → 确认收敛）

docs/                                # 本仓库 skill 产出的知识文档（含官方来源清单）
├── iOS签名与出包知识体系.md           # 通用 App 打包上架知识体系
└── iOS游戏出包实战（引擎导出Xcode工程）.md  # 游戏出包专项（Unity/Unreal/Cocos）
```

## 知识文档（docs/ 目录）

`beginner-tech-doc` skill 实际产出的成果，关键事实均经网络核验（核验日期 2026-11-19，各文档文末附官方与参考链接）：

- **iOS 签名与出包知识体系**——Mach-O / 证书 / 签名 / 描述文件 / ipa / 出包三条路 / 事故排查 / 上架全景 SOP
- **iOS 游戏出包实战**——Unity / Unreal / Cocos 引擎导出与游戏特有签名场景
