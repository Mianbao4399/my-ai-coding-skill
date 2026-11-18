# 我的 AI 编程 Skill

个人 Trae skill 仓库——目前收录 `beginner-tech-doc`：一个"技术问题 → 搜索验证 → 新手友好文档"的技能。

## beginner-tech-doc 是什么

触发后，agent 会**先用网络搜索验证事实**（版本、政策、菜单路径、错误码等易过时信息），再生成一份**新手友好、可照着实操**的中文技术文档：

- 比喻地图开路，全局直觉先行
- 专业名词中英对照 + 大白话解释
- 每个实操步骤写明"在哪个界面点哪个按钮"
- 报错三件套：现象原文 → 原理 → 按序解法
- 全景 SOP（含第 0 步前置条件：账号/费用/资质）
- 交付前 7 项卡点自检，保证新手不在任何一步卡住

触发词：用户提出技术知识类问题，或要求"新手教程 / 知识体系文档 / 实操指南"。

## 安装（新电脑）

把 `beginner-tech-doc` 文件夹复制到本机 skills 目录，重启 Trae 即生效：

```powershell
# CN 版 Trae
git clone https://github.com/Mianbao4399/my-ai-coding-skill.git
Copy-Item -Recurse .\my-ai-coding-skill\beginner-tech-doc "$env:USERPROFILE\.trae-cn\skills\"

# 国际版 Trae（目录是 .trae 不是 .trae-cn）
Copy-Item -Recurse .\my-ai-coding-skill\beginner-tech-doc "$env:USERPROFILE\.trae\skills\"
```

## 更新

```powershell
git pull
Copy-Item -Recurse -Force .\beginner-tech-doc "$env:USERPROFILE\.trae-cn\skills\"
```

## 文件结构

```
beginner-tech-doc/
├── SKILL.md                 # 技能指令（工作流 + 内容规则 + 卡点自检）
└── assets/
    └── doc-template.md      # 文档标准结构模板
```
