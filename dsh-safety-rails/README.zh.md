# 🛡️ DSH Safety Rails — AI 编码代理的"先确认再动手"护栏

> 让 AI 编码代理在改动任何文件前**必须先说明方案、等待你确认**，用双层机制保证它无法绕过。

[![适用](https://img.shields.io/badge/适用-DeepSeek%20Harness%20%7C%20类%20AGENTS.md%20工具-blue)](#适用范围)

---

## 背景：为什么要做这个

用 AI 编码工具（DSH / Claude Code / Cursor 等）时，很多人踩过同一个坑：

> **需求还不明确时，AI 已经"先斩后奏"开始改文件了。**
> 改错方向后，又得在后续会话里要求它回退——**浪费 token、浪费时间、还容易把代码改乱。**

（真实经历：用某编码工具做公司项目，需求模糊时它直接开改，后来追加要求才发现方向错了，折腾回退。）

本仓库提供的，是一套**让 AI 强制"先说明、再确认、后动手"**的配置模板，适用于 DeepSeek Harness（DSH），其思路对任何支持"系统提示 + 用户目录规则"的编码代理都通用。

---

## 核心思想：一句话

> **"先确认，再动手"** —— 用户的要求 ≠ 用户的同意。

AI 出问题的根源在于：它把"用户让我做 X"理解为"用户授权我直接做 X"。但需求往往模糊——用户说的"做 X"可能根本不是 AI 理解的"做 X"。

所以规则的核心是**第 2 条**：

> **用户的"要求执行某操作"本身不是同意**：即使用户直接说"创建XX文件""修改XX"，也必须先展示具体改动方案并等待确认；仅当用户在同一消息中明确使用"不用确认""直接执行""直接改"等豁免字样时，才可免确认执行。

---

## 双层机制（软 + 硬）

| 层 | 机制 | 效果 |
|---|---|---|
| **软层①** | `AGENTS.md` 用户规则（自动注入每个会话） | 模型看到规则，自觉遵守 |
| **软层②** | 自定义 agent preset（规则写入系统提示 persona） | 即使不使用 AGENTS.md 加载路径，规则也在系统提示里 |
| **硬层** | 权限预设 `workspace-write` | **写文件时系统强制弹窗批准**——模型想绕过也绕不过 |

为什么需要硬层：模型对"提示词规则"可能服从性不足（实测中，deepseek-v4-flash 在明确用户指令下曾无视规则直接创建文件）。**系统级批准才是最可靠的保证**。

---

## 文件清单

| 文件 | 放到哪 | 作用 |
|---|---|---|
| `templates/AGENTS.md` | `~/.dsh/AGENTS.md` | 全局规则（主载体，实时生效） |
| `templates/standard-rules-agent.cordis.yml` | `~/.dsh/.agent-presets/standard-rules/agent.cordis.yml` | 含规则的自定义 preset |
| `templates/cordis.patch.yml` | `~/.dsh/profiles/web/cordis.patch.yml` | 把默认 preset 指向 standard-rules |
| `templates/settings.yaml` | `~/.dsh/settings.yaml` | 权限预设 `workspace-write` |

---

## 完整规则文本（7 条）

```markdown
## 文件与代码改动确认

1. 执行任何文件创建、修改、删除操作（代码、配置、脚本、文档）前，必须先向用户说明：
   改哪个文件、具体改什么、目的是什么，然后停下等待用户明确同意，再执行。
2. 用户的"要求执行某操作"本身不是同意：即使用户直接说"创建XX文件""修改XX"，
   也必须先展示具体改动方案并等待确认；仅当用户在同一消息中明确使用
   "不用确认""直接执行""直接改"等豁免字样时，才可免确认执行。
3. 批量确认：涉及多个文件的改动，一次性列出完整清单确认；
   执行中需要清单外的新改动时，重新确认。
4. 免确认例外：① 只读操作（查看/搜索/分析）；② 运行时系统自产文件（日志、缓存、会话数据）；
   ③ 为完成已确认任务所需的临时中间产物。
5. 意图不清先问：对改动意图、影响范围、恢复方式不确定时，先提问澄清，不猜测执行。
6. 高风险操作（删除文件、覆盖已有配置、批量修改）额外说明影响范围和是否可回滚。
7. 本规则对所有会话生效；会话内用户明确豁免时不适用。
```

### 设计要点

- **第 2 条是灵魂**：化解"用户要求≠同意"的歧义，这是 AI 先斩后奏的最大根源
- **第 3 条批量确认**：大任务一次确认，避免逐文件反复打断
- **第 4 条只读例外**：查看/搜索/分析无需确认，日常使用零打扰
- **第 5 条不确定性处理**：意图不清时提问而非猜测
- **第 7 条豁免**：你说"直接做"时放行，兼顾效率

---

## 快速开始（DSH）

### 安装配置

1. **复制规则**（最简单，只做这一步就生效）：
   ```bash
   cp templates/AGENTS.md ~/.dsh/AGENTS.md
   ```
   新会话即自动加载（当前会话部分场景也实时生效）。

2. **（可选）系统硬拦截**：编辑 `~/.dsh/settings.yaml`，加入：
   ```yaml
   permission:
     defaultPreset: workspace-write
   ```
   之后 AI 尝试写文件时，系统会**弹窗要求你批准**——批准才执行。

3. **（可选）preset 层规则**（双保险）：
   ```bash
   # 复制官方 standard preset 为自定义 preset
   cp -r <DSH安装目录>/config/agent-presets/standard ~/.dsh/.agent-presets/standard-rules
   # 用 templates/standard-rules-agent.cordis.yml 替换其中的 agent.cordis.yml
   # 再在 ~/.dsh/profiles/web/cordis.patch.yml 中追加：
   #   - id: agent-presets
   #     config:
   #       default: standard-rules
   ```
   > ⚠️ **重要**：Web 界面每个会话挂载 agent preset（默认 `standard`），preset 的 persona 会**遮蔽**全局 persona。只写 AGENTS.md 或全局 persona 时，规则可能不生效——必须让预设的 persona 包含规则（本模板已包含）。

### 验证

新建会话，发一句：
> 在当前目录创建一个 test.txt，内容为 hello world

- ✅ **生效**：AI 先停下，说明"改哪个文件、什么内容、目的"，等你确认才动手
- ❌ **未生效**：直接创建 → 检查 AGENTS.md 是否生效、preset 是否正确

---

## 适用范围

| 工具 | 适配方式 |
|---|---|
| **DeepSeek Harness (DSH)** | 按本仓库模板配置（AGENTS.md + preset + 权限） |
| Claude Code / Cursor 等 | 规则文本放各自的项目规则文件（如 `CLAUDE.md`、`.cursorrules`），权限机制按各自文档 |
| 自制 AI 编码代理 | 把规则文本加入系统提示；在工具执行层加"写入前确认"机制 |

核心不依赖任何 DSH 特有功能：**规则文本 + 文件权限批准**是通用思路。

---

## ⚠️ 安全提醒

- **永远不要提交** `~/.dsh/.credentials.yaml`（含 API 密钥）到任何仓库
- 不要提交 `sessions/`（历史对话）、`storages/` 等个人数据
- 本仓库的 `.gitignore` 已内置这些排除项
- 分享你的 `settings.yaml` 前，去掉 `agent-default-model` 等个人路由配置

---

## 已知局限

- 软层（提示词规则）依赖模型的服从性——不同模型、不同场景下可能失效；**硬层（权限批准）才是可靠保证**
- `workspace-write` 下写工作目录外的文件会被拒绝（或用批准弹窗），需要按需调整权限
- preset persona 遮蔽机制是 DSH 特有行为，其他工具无此问题

## License

MIT
