<div align="center">

# 小弟 (xiaodi)

**让大哥指挥，小弟干活**

Opus 规划 · Codex 执行 · Opus 审查

[![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=for-the-badge&logo=anthropic)](https://claude.ai)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-0.118.0-green?style=for-the-badge&logo=openai)](https://github.com/openai/codex)
[![Token Savings](https://img.shields.io/badge/Opus_Token-↓70--85%25-orange?style=for-the-badge)]()
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)]()

<br/>

*一个 Claude Code 技能插件，将昂贵的 Opus 推理用在刀刃上（规划+审查），把大量编码实现工作交给 Codex CLI——就像大哥画蓝图、小弟搬砖。*

</div>

---

## 为什么需要小弟？

使用 Claude Opus 编码时，大部分 token 消耗在**实际写代码**上——读文件、生成代码、运行命令、修 bug。这些工作需要大量 token，但并不都需要 Opus 级别的推理能力。

**小弟的解决方案：分工协作。**

```
┌─────────────────────────────────────────────────────────┐
│                     你的编码任务                          │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────┐
        │   Phase 1: Opus 规划      │  ← 大哥出谋划策
        │   分析需求 → 生成详细计划   ��     (~3K-5K tokens)
        └────────────┬─────────────┘
                     │ 结构化计划
                     ▼
        ┌──────────────────────────┐
        │   Phase 2: Codex 执行     │  ← 小弟埋头苦干
        │   codex exec 非交互执行    │     (~10K-50K cheap tokens)
        │   编辑文件 · 运行命令      │
        └────────────┬─────────────┘
                     │ git diff + 事件日志
                     ▼
        ┌──────────────────────────┐
        │   Phase 3: Opus 审查      │  ← 大哥验收成果
        │   三源交叉验证 → 判定结果   │     (~2K-5K tokens)
        └────────────┬─────────────┘
                     │
              ┌──────┴──────┐
              │             │
           PASS ✓       FAIL ✗
           完工！      增量修正 →
                     回到 Phase 2
                    (最多重试 3 次)
```

## Token 对比

| 模式 | Opus Token | 其他 Token | 总成本 |
|:---:|:---:|:---:|:---:|
| **纯 Opus** | 30,000 - 80,000 | — | $$$$$ |
| **小弟混合** | 5,000 - 10,000 | 10,000 - 50,000 (Codex) | $$ |
| **节省** | **↓ 70-85%** | — | — |

> Opus token 单价远高于 Codex 使用的模型（如 GPT-4.1 / GPT-5.4），实际成本节省可能更显著。

## 快速开始

### 前置条件

```bash
# 1. 安装 Codex CLI
npm install -g @openai/codex

# 2. 配置 Codex 为全自动模式 (~/.codex/config.toml)
approval_policy = "never"
sandbox_mode = "danger-full-access"  # 或 workspace-write

# 3. 确保工作目录是 git 仓库
git init  # 如果还不是
```

### 安装技能

将 `SKILL.md` 复制到你的 Claude Code 技能目录：

```bash
# 克隆仓库
git clone https://github.com/qiuxinyuan321/xiaodi.git

# 复制到 Claude Code 技能目录
cp -r xiaodi ~/.claude/skills/codex-executor
```

### 使用

在 Claude Code 中，对任何编码任务说：

```
/codex 帮我重构这个模块的错误处理
```

或者自然语言触发：

```
交给 codex 执行：给 UserService 添加缓存层
```

## 工作原理详解

### Phase 1: 大哥规划

Opus 分析你的需求，生成一份**结构化实施计划**，包含：

- **上下文**：工作目录、技术栈、约束条件
- **目标**：一句话描述 + 可验证的成功标准
- **步骤**：编号的实施步骤，每步含文件路径、变更内容、验证命令
- **约束**：不可触碰的文件和接口

计划会写入临时目录（`~/.codex/tmp/`），不污染你的项目。

```markdown
## CODEX EXECUTION PLAN

### Context
- Working directory: /home/user/my-project
- Tech stack: TypeScript, Express, PostgreSQL
- Test command: npm test

### Objective
Add Redis caching layer to UserService.getById()

### Success Criteria
- [ ] UserService.getById() checks Redis before hitting DB
- [ ] Cache TTL set to 300 seconds
- [ ] All existing tests pass

### Implementation Steps
1. Install ioredis dependency
   - Change: add to package.json
   - Verify: npm ls ioredis
2. Create src/cache/redis-client.ts
   - Change: Redis connection singleton
   - Verify: file exists with correct exports
3. Modify src/services/user-service.ts
   - Change: add cache check in getById()
   - Verify: grep "redis" src/services/user-service.ts
4. Run tests: npm test
   - Expected: all tests pass
```

### Phase 2: 小弟执行

Opus 自动判断任务复杂度，选择合适的 Codex profile：

| 复杂度 | Profile | 推理等级 | 适用场景 |
|:---:|:---:|:---:|:---|
| 简单 | `speed` | low | 单文件修改、文档、格式化 |
| 复杂 | `auto-max` | xhigh | 多文件、跨模块、含测试 |

然后通过 `codex exec` 非交互式执行：

```bash
codex exec \
  -C "$WORKDIR" \
  -p "$PROFILE" \
  --color never \
  --json \
  -o "summary.txt" \
  "$(cat plan.md)" \
  > events.jsonl 2>&1
```

**双轨输出捕获：**

| 文件 | 内容 | 用途 |
|:---|:---|:---|
| `events.jsonl` | 完整 JSONL 事件流（每条命令、每次编辑） | 发现隐藏失败 |
| `summary.txt` | Codex 最终结论 | 快速对齐 |

### Phase 3: 大哥验收

Opus 从**三个独立来源**交叉验证执行结果：

```
优先级 1: git diff HEAD          ← 地面真相：实际发生了什么
优先级 2: summary.txt            ← Codex 说发生了什么
优先级 3: events.jsonl 中的失败   ← 隐藏的错误和非零退出码
```

审查清单：

- [x] `git diff` 是否匹配计划中的实施步骤？
- [x] 成功标准是否已满足？
- [x] 事件流中有无失败命令（非零退出码）？
- [x] 测试命令是否出现且以 exit code 0 结束？

### 智能重试

审查不通过时，**不会完整重跑**——而是生成增量补丁：

```
审查发现 → 生成 PATCH_PROMPT → codex exec 增量修正 → 再次审查
```

- 基于当前 `git diff` 做最小修正
- 每次重试是独立的 `codex exec` 调用（无状态）
- 最多 3 次自动重试，之后升级给人工

## 架构一览

```
Claude Code (Opus)                    Codex CLI
┌─────────────────┐                  ┌─────────────────┐
│                 │   结构化计划       │                 │
│   Phase 1       │ ───────────────→ │   Phase 2       │
│   规划          │                  │   执行          │
│                 │   git diff +     │                 │
│   Phase 3       │ ←─────────────── │   events.jsonl  │
│   审查          │                  │   summary.txt   │
│                 │                  │                 │
│   ┌───────────┐ │   PATCH_PROMPT   │                 │
│   │ 重试判断   │ │ ───────────────→ │   增量修正       │
│   └───────────┘ │                  │                 │
└─────────────────┘                  └─────────────────┘
        │                                    │
        └── Opus: ~7K tokens ──────── Codex: ~30K tokens ──┘
```

## 适用场景

| 场景 | 推荐 | 原因 |
|:---|:---:|:---|
| 多文件功能开发 | ✅ | Codex 处理大量编辑，Opus 只做规划审查 |
| 重构 / 迁移 | ✅ | 步骤清晰可计划，执���量大 |
| 添加测试用例 | ✅ | 模式明确，适合自动执行 |
| Bug 修复（已定位） | ✅ | 修复步骤可规划，验证可自动化 |
| 架构设计讨论 | ❌ | 全程需要 Opus 推理，无法委托 |
| 一行代码修改 | ❌ | 三阶段开销大于收益 |
| 需要交互式调试 | ❌ | Codex exec 是非交互的 |

## 配置参考

### Codex `config.toml` 推荐配置

```toml
# ~/.codex/config.toml

# 非交互必须
approval_policy = "never"

# Profile: 快速任务
[profiles.speed]
model_reasoning_effort = "low"
approval_policy = "never"

# Profile: 复杂任务
[profiles.auto-max]
model_reasoning_effort = "xhigh"
approval_policy = "never"
```

### 触发关键词

| 关键词 | 语言 |
|:---|:---:|
| `/codex` | — |
| `delegate to codex` | EN |
| `交给codex` / `用codex执行` | CN |
| `save opus tokens` | EN |
| `hybrid workflow` | EN |

## FAQ

**Q: Codex 执行出错怎么办？**

小弟会自动重试最多 3 次，每次用增量补丁而非完整重跑。3 次后仍失败会升级给你处理。

**Q: 支持非 git 项目吗？**

目前不支持。审查阶段依赖 `git diff` 作为地面真相来验证变更。

**Q: Codex 会修改我的项目之外的文件吗？**

取决于你的 `sandbox_mode` 配置。建议使用 `workspace-write`（只允许写项目目录）而非 `danger-full-access`。

**Q: 和直接用 Codex 有什么区别？**

直接用 Codex 没有 Opus 级别的规划和审查。小弟的价值在于：Opus 的规划让 Codex 少走弯路，Opus 的审查确保质量——两头用最强大脑，中间用高效执行者。

## License

MIT

---

<div align="center">

**大哥动动嘴，小弟跑断腿** 🏃

*Built with [Claude Code](https://claude.ai/claude-code) + [Codex CLI](https://github.com/openai/codex)*

</div>
