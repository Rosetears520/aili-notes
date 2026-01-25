# AiLi Notes

**AiLi = AI Library**。这是我的个人 AI 资料库，用来分享好用、可复用的 **Prompts / Skills / Commands / Workflows**，实现更快写作、更稳编码、更高效的研究与交付。

目标很简单：把"零散好用的提示词和套路"变成一个 **可检索、可复用、可持续迭代** 的仓库。

---

## 目录结构

```
aili-notes/
├── prompts/           # 提示词库（Prompt Library）
│   ├── research/      # 研究类提示词
│   ├── writing/       # 写作类提示词
│   └── templates/     # 通用模板
├── skills/            # 技能库（AI AgentSkills）
│   ├── code-polish/                    # 代码精炼与标准化
│   ├── openspec-change-interviewer/    # OpenSpec 变更面试官
│   └── openspec-feature-list/          # OpenSpec 功能列表生成
├── commands/          # 命令库（AI AgentCommands）
│   └── coding/        # 编码相关命令
└── workflows/         # 工作流（暂未收录）
```

---

## 各部分详解

### Prompts（提示词库）

提示词库包含了可直接复制粘贴使用的 AI 提示词模板，覆盖研究、写作等场景。

#### Research（研究类）

| 文件 | 作用 |
|------|------|
| `文献检索.md` | 模拟专业文献检索专家团队，为研究课题生成 CNKI（中国知网）和 Web of Science 的高质量检索式。支持种子文献验证、多轮迭代优化，帮助研究者快速构建精准的文献检索策略。 |

#### Writing（写作类）

| 文件 | 作用 |
|------|------|
| `高保真润色模拟器.md` | 模拟顶级期刊编辑部和同行评审团队，对学术论文进行润色。包含逻辑架构、术语精确化、限制定位、黑名单词汇过滤等完整工作流，确保学术表达符合顶刊标准。 |
| `High-Fidelity Polishing Simulator.md` | 上述润色模拟器的英文版本。 |

#### Templates（通用模板）

| 文件 | 作用 |
|------|------|
| `AI明确需求.md` | AI 提示词优化专家与战略顾问。通过苏格拉底式对话帮助用户澄清需求，构建精准的提示词。支持 DETAIL（深度咨询）和 BASIC（快速优化）两种模式，适用于各类 AI 平台（ChatGPT、Claude、Gemini、DeepSeek、通义千问、豆包等）。 |

---

### Skills（技能库）

技能库包含 AI Agent的自定义技能定义文件（`SKILL.md`），用于扩展 Claude 的专业能力。

#### code-polish（代码精炼与标准化）

**作用：** 工程级代码重构与标准化专家。

**核心功能：**
- 简化控制流、降低嵌套深度、减少圈复杂度
- 标准化代码风格，提升可维护性
- 支持 Python、Go、C/C++、JS/TS 等多语言
- 自动发现项目编码规范（`.editorconfig`、`pyproject.toml`、`tsconfig.json` 等）
- 输出 Unified Diff 格式的补丁，确保可审计性

**使用场景：** 需要保持行为不变的前提下，改善代码结构和可读性。

#### openspec-change-interviewer（OpenSpec 变更面试官）

**作用：** 通过访谈用户来完善 OpenSpec 变更包的内容。

**核心功能：**
- 读取现有 `proposal.md`、`design.md`、`tasks.md`、`specs/` 文件
- 识别缺失或模糊的需求点
- 通过高信息增益问题澄清细节
- 将完善的内容写回文件
- 运行 `openspec validate` 进行验证

**使用场景：** OpenSpec 变更需求不完整时，通过结构化访谈补全信息。

#### openspec-feature-list（OpenSpec 功能列表生成）

**作用：** 从 `tasks.md` 解析生成 `feature_list.json`。

**核心功能：**
- 解析复选框任务（`- [ ]` / `- [x]`）
- 提取 `[#R<n>]` 引用标记
- 提取 `ACCEPT:` 和 `TEST:` 字段
- 自动推断功能分类（`functional` / `docs` / `testing` / `performance` / `maintenance`）
- 保留已有的 `passes` 状态

**使用场景：** 自动化 OpenSpec 功能清单的维护与追踪。

---

### Commands（命令库）

命令库包含 AI Agent的自定义命令定义文件。

#### monitor-openspec-codex（OpenSpec Codex 监控器）

**作用：** 监督 OpenSpec 变更的批量执行流程。

**核心功能：**
- 串行执行 `tasks.md` 中的未完成任务
- 每个任务 = 一个独立子代理 = 一次 Codex 运行
- 自动重试机制（默认 `MAX_ATTEMPTS=2`）
- 依赖阻塞保护（前置任务失败时停止后续任务）
- 持久化进度记录（`progress.txt`）
- 功能状态追踪（`feature_list.json`）

**工作流程：**
1. Supervisor 读取 `tasks.md`，选择下一个符合条件的任务
2. 启动 Worker 子代理，运行 Codex CLI
3. Worker 生成验证包（validation bundle）
4. Supervisor 执行验证，记录证据，更新状态
5. 根据结果继续或重试

**使用场景：** 自动化 OpenSpec 变更的端到端执行与监控。

---

## 使用方式

### Prompts（提示词）

直接复制对应文件内容，粘贴到任意 AI 助手（ChatGPT、Claude、DeepSeek 等）中使用。

### Skills（技能）

将 `skills/` 目录下的技能文件夹放入 AI Agent 的技能目录中，即可通过 `$skill-name` 语法调用。

### Commands（命令）

将 `commands/` 目录下的命令文件放入 AI Agent 的命令目录中，即可通过 `/command-name` 语法调用。

---

## 许可证

本项目采用 [LICENSE](LICENSE) 中定义的开源许可证。

---

## 贡献

欢迎提交 Issue 和 Pull Request，帮助完善这个 AI 资产库。
