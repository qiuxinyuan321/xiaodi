<div align="center">

# 小弟 (xiaodi)

**Claude Code 的执行引擎 —— 让大哥指挥，小弟干活**

Claude Code 控制 Codex CLI，像遥控器操作机器人一样

[![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=for-the-badge&logo=anthropic)](https://claude.ai)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-0.118.0-green?style=for-the-badge&logo=openai)](https://github.com/openai/codex)
[![Token Savings](https://img.shields.io/badge/Opus_Token-↓70--85%25-orange?style=for-the-badge)]()
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)]()

</div>

---

## 核心理念

Claude Code (Opus) 是**大脑**，Codex CLI 是**双手**。

大脑负责思考——分析需求、制定计划、审查结果。双手负责干活——编辑文件、运行命令、执行测试。大脑很贵（Opus token），双手很便宜（Codex token）。把贵的资源用在刀刃上，这就是小弟的设计哲学。

```
                    ┌──────────────────────┐
                    │       你的任务        │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Claude Code (Opus)   │
                    │                      │
                    │  "我来分析需求，      │
                    │   写好详细计划"       │
                    └──────────┬───────────┘
                               │ 结构化计划
                    ┌──────────▼───────────┐
                    │    Codex CLI (小弟)    │
                    │                      │
                    │  "收到！马上干活！     │
                    │   改文件、跑命令"      │
                    └──────────┬───────────┘
                               │ 执行结果
                    ┌──────────▼───────────┐
                    │  Claude Code (Opus)   │
                    │                      │
                    │  "让我检查一下...      │
                    │   嗯，干得不错！"      │
                    └──────────┴───────────┘
```

## 为什么不直接全用 Opus？

| | 纯 Opus | 小弟模式 |
|:---|:---:|:---:|
| Opus Token | 30,000 - 80,000 | **5,000 - 10,000** |
| Codex Token | — | 10,000 - 50,000 (便宜) |
| Opus 节省 | — | **70-85%** |
| 适用场景 | 所有任务 | 多文件实现类任务 |

Opus 的价值在于**推理能力**，不在于搬砖。让 Opus 花大量 token 去逐行写代码，就像让建筑师亲自搬砖——太浪费了。

## 怎么控制的？

Claude Code 通过 `codex exec` 命令**非交互式**地控制 Codex CLI：

```bash
# Claude Code 把计划写成文件，然后一条命令交给 Codex 执行
codex exec -C "/你的/项目目录" --color never -o "结果.txt" "计划内容"
```

Codex CLI 就像一个听话的终端 —— 接收指令，执行完毕，汇报结果。**不需要 git 仓库**，任何目录都能用。

## 三个阶段

### Phase 1: Claude Code 规划

Claude Code 分析你的需求，生成一份结构化计划：

```markdown
## CODEX EXECUTION PLAN

### Context
- Working directory: F:\my-project
- Tech stack: TypeScript, React
- Test command: npm test

### Objective
给 UserService 添加 Redis 缓存层

### Success Criteria
- [ ] getById() 先查 Redis 再查数据库
- [ ] 缓存 TTL 300 秒
- [ ] 所有现有测试通过

### Implementation Steps
1. 安装 ioredis → npm ls ioredis 验证
2. 创建 src/cache/redis-client.ts → 文件存在即通过
3. 修改 src/services/user-service.ts → grep "redis" 验证
4. 运行 npm test → 全部通过
```

Claude Code 同时判断任务复杂度，决定用哪个 Codex profile：
- **简单任务**（单文件、文档）→ `speed` profile（快但推理弱）
- **复杂任务**（多文件、跨模块）→ `auto-max` profile（慢但推理强）

### Phase 2: Codex 执行

Claude Code 把计划交给 Codex，然后**双轨捕获**执行过程：

```bash
codex exec \
  -C "$WORKDIR" \           # 项目目录（任意目录）
  -p "$PROFILE" \           # speed 或 auto-max
  --color never \           # 防止 ANSI 污染
  --json \                  # JSONL 事件流 → events.jsonl
  -o "summary.txt" \        # 最终结论
  "$(cat plan.md)"          # 计划内容
```

| 输出 | 内容 | 作用 |
|:---|:---|:---|
| `events.jsonl` | 每条命令、每次编辑的完整事件流 | 发现隐藏的失败 |
| `summary.txt` | Codex 的最终自述 | 快速了解执行情况 |

### Phase 3: Claude Code 审查

Claude Code 不会轻信 Codex 说"我做完了"，而是**独立验证**：

**在 git 仓库中：**
```bash
git diff HEAD  # 最可靠的地面真相
```

**在任意目录中：**
```bash
# 对比执行前后的文件快照
diff snapshot-before.txt snapshot-after.txt
# 然后直接读取变更的文件内容来审查
```

**两种模式都会检查：**
- Codex 的自述报告（summary.txt）
- 事件流中的失败命令（非零退出码）

审查结论：
- **PASS** → 完工！
- **PARTIAL / FAIL** → 生成增量补丁，让 Codex 修正（最多重试 3 次）

## 智能重试

失败时**不会完整重跑**，而是基于当前状态生成最小修正指令：

```
Phase 3 发现问题 → 生成 PATCH → Codex 增量修正 → 再次审查
                                     ↑                    │
                                     └────── 重试 ────────┘
                                          (最多 3 次)
```

第 4 次仍失败？升级给你手动处理。

## 快速开始

### 1. 安装 Codex CLI

```bash
npm install -g @openai/codex
```

### 2. 配置 Codex 全自动模式

编辑 `~/.codex/config.toml`：

```toml
approval_policy = "never"    # 非交互执行必须
# sandbox_mode 根据你的安全需求选择：
# "workspace-write" — 只能写项目目录（推荐）
# "danger-full-access" — 无限制
```

### 3. 安装技能到 Claude Code

```bash
git clone https://github.com/qiuxinyuan321/xiaodi.git
cp -r xiaodi ~/.claude/skills/codex-executor
```

### 4. 使用

在 Claude Code 中说：

```
/codex 给这个项目添加单元测试

交给 codex：重构 UserService 的错误处理

用 codex 执行这个计划
```

## 适用场景

| 场景 | 用小弟？ | 原因 |
|:---|:---:|:---|
| 多文件功能开发 | ✅ | 大量编辑工作适合委托 |
| 代码重构 / 迁移 | ✅ | 步骤清晰，执行量大 |
| 批量添加测试 | ✅ | 模式化工作，Codex 擅长 |
| Bug 修复（已定位） | ✅ | 修复步骤可规划 |
| 架构设计讨论 | ❌ | 全程需要 Opus 推理 |
| 一行代码修改 | ❌ | 三阶段开销太重 |
| 交互式调试 | ❌ | Codex exec 不支持交互 |

## FAQ

**Q: 必须是 git 仓库才能用吗？**

不是。任何目录都能用。git 仓库下审查更方便（`git diff`），非 git 目录用文件快照对比也能工作。

**Q: Codex 和 Claude Code 用的是同一个 API 吗？**

不是。Claude Code 用 Anthropic API (Opus)，Codex 用 OpenAI API（或你配置的 AI Router）。两套独立的 API，独立计费。这正是省钱的关键——把 bulk 工作量转移到更便宜的一侧。

**Q: 执行出错怎么办？**

自动重试最多 3 次，每次用增量补丁而非完整重跑。3 次后仍失败会通知你手动处理。

**Q: 和直接用 Codex 有什么区别？**

直接用 Codex = 让小弟自己决定做什么、怎么做。
用小弟技能 = Claude Code 这个大哥画蓝图、验收成果，Codex 只负责施工。
**质量由 Opus 把关，成本由 Codex 承担。**

## License

MIT

---

<div align="center">

**大哥动动嘴，小弟跑断腿** 🏃

*Claude Code controls Codex CLI — brain thinks, hands work.*

</div>
