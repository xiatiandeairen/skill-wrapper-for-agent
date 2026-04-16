---
name: skill-wrapper
description: Skill 质量保障工具。Use when user says "skill-wrapper", "lint skill", "检查skill", "模拟skill", "simulate", or invokes /skill-wrapper.
---

# Skill Wrapper

`/skill-wrapper {subcommand} [target]` → route → execute.

## Core Principles

1. **行为优先** — skill 质量 = AI 按指令执行时行为符合预期的程度。结构和格式是次要的。
2. **只暴露不修改** — 报告问题，不替代作者重写。作者对 skill 有最终判断权。
3. **指令精确性** — 模糊指令是 AI 行为偏离的首要原因。写行为不写意图，写条件不写期望。

## Definitions

| Term | Meaning |
|------|---------|
| 行为链 | AI 按 skill 规则执行时的决策序列：入口匹配 → 阶段选择 → 每步 gate 判定 → 产出 |
| dry-run | 纯推理模拟，不执行实际操作（不创建文件、不调用脚本） |
| 指令缺陷 | SKILL.md 中会导致 AI 执行行为不可预测或偏离预期的表述 |
| [AMBIGUOUS] | simulate 输出标记：规则冲突导致 AI 可能选择不同行为分支 |
| [MISSING] | simulate 输出标记：引用的 stage 文件不存在 |

## Subcommands

| Subcommand | Alias | Description |
|------------|-------|-------------|
| `lint` | `l` | 扫描 SKILL.md 指令质量缺陷，输出报告 |
| `simulate` | `sim` | 给定 skill + 模拟输入，dry-run 行为链 |

未匹配任何子命令时，显示此表并提示用户选择。

**lint** 是快速扫描（秒级，模式匹配），适合每次改完跑一下。**simulate** 是深度验证（分钟级，全链路推理），适合发布前或大改后。lint 发现的指令缺陷在 simulate 中表现为 [AMBIGUOUS] 或行为分支不确定。两者互补。

## Routing

```
/skill-wrapper {input}
1. 第一个词匹配子命令名或 alias？ → 进入对应流程
2. 输入为 .md 文件路径？           → 默认 lint 子命令
3. 否则                           → 显示帮助
```

---

## Subcommand: lint

扫描 SKILL.md 中会导致 AI 执行偏离的指令缺陷。不改文件，只报告。

### Rules

10 条检查规则，按严重度分级：

**Error（会导致 AI 行为不可预测）**

| # | 规则 | 检测方法 |
|---|------|---------|
| L1 | **模糊指令** — 使用"适当/尽量/大概/合理/视情况/roughly/approximately/maybe" | 关键词匹配 |
| L2 | **缺量化标准** — 条件判断无具体数字（"大量改动"而非">5 文件"） | 扫描 Rules/Workflow 中的条件句，检查是否含数字或枚举 |
| L9 | **Gate 无判定标准** — 跳过条件使用"视情况/if appropriate"等不可判定表达 | 扫描 Gate 描述，检查是否有可判定的 yes/no 条件 |

**Warning（可能导致 AI 行为漂移）**

| # | 规则 | 检测方法 |
|---|------|---------|
| L3 | **意图而非行为** — "确保高质量"而非"检查这 5 项" | 扫描动词，标记抽象动词（确保/优化/改善/提升/ensure/improve） |
| L4 | **隐式规则** — Rules/Workflow 中引用的概念术语未在 Definitions 中定义 | 提取 Rules/Workflow 中加粗/反引号/大写术语，检查 Definitions 是否有对应定义 |
| L6 | **混合关注点** — 一条规则管多件事（含"并/且/and"连接不同动作） | 扫描 Rules 中含并列连接词连接两个不同动作的条目 |
| L8 | **无约束生成** — 要求 AI "生成 N 个方案"但无筛选条件 | 扫描"生成/generate/produce" + 数量词，检查是否有约束条件 |

**Info（建议改善）**

| # | 规则 | 检测方法 |
|---|------|---------|
| L5 | **缺 few-shot** — 关键产出无 good/bad 示例 | 检查 Workflow 中输出模板是否有 example/few-shot/good/bad |
| L7 | **未定义术语** — 正文使用但 Definitions 中未定义 | 提取正文中大写/加粗/反引号术语，对比 Definitions 表 |
| L10 | **缺恢复策略** — 有 Workflow 但无 Recovery/Edge Cases/Defaults | 检查对应 section 是否存在 |

### Workflow

```
Step 1: Read    — 读取目标 SKILL.md（+ stage 文件，如有）
Step 2: Scan    — 逐规则扫描 L1-L10
Step 3: Report  — 输出报告（见下方模板）
Step 4: Summary — 整体评估
```

### 报告模板

```
### Lint Report: {filename} ({N} lines)

errors: {N} | warnings: {N} | info: {N}

**Errors**
- L{n}: {规则名} — line {N} — `{原文引用}`
  → fix: {具体修复建议}

**Warnings**
- L{n}: {规则名} — line {N} — `{原文引用}`
  → fix: {具体修复建议}

**Info**
- L{n}: {规则名} — {位置描述}
  → fix: {具体修复建议}

**Summary**
{error >0: "指令缺陷会影响 AI 执行稳定性，建议修复 errors"}
{error =0, warning >0: "指令基本可用，建议优化 warnings"}
{全 0: "指令质量良好"}
```

### Few-shot

**Good report item**:
```
- L2: 缺量化标准 — line 45 — `大量改动时需要谨慎`
  → fix: 改为 ">5 文件改动时，列出改动清单让用户确认"
```

**Bad report item**:
```
- L2: 缺量化标准 — 文档中有模糊表达
  → fix: 建议量化
```
（缺行号、缺原文引用、修复建议不具体）

---

## Subcommand: simulate

给定 skill + 模拟输入，dry-run AI 执行行为链。纯推理，不执行任何实际操作。

### Workflow

```
Step 1: Load    — 读取目标 SKILL.md + 所有 stage 文件
Step 2: Input   — 解析模拟输入（用户提供或 [STOP:respond] 询问）
Step 3: Trace   — 按 skill 规则 dry-run，输出行为链（见下方模板）
Step 4: Review  — [STOP:respond] 作者审查行为链
```

### 行为链模板

```
### Simulate: {input}

**Entry**
→ 匹配: {入口规则} → {路由结果}

**Stage: {name}**
→ Step {n} ({name}): Gate={type}
  条件: {gate 条件} → {met/not met} → {进入/跳过}
→ Step {n+1} ...
→ Handoff: {关键产出}

**Decision Points**
- {规则 A}: 条件 {met/not met} → 行为 {X}
- {规则 B}: 条件 {met/not met} → 行为 {Y}

**[AMBIGUOUS]** (如有)
- {规则 A} says {X}, {规则 B} says {Y}
  → AI 可能选择 {X 或 Y}，取决于 {上下文}

**[MISSING]** (如有)
- Stage file {path} not found → 跳过该 stage trace
```

### 关键约束

- simulate 不执行任何实际操作（不创建文件、不调用脚本）
- 输出的是 AI **会怎么做**，不是 AI **应该怎么做**
- 发现规则冲突/歧义时标记 `[AMBIGUOUS]`，展示两种可能的行为分支
- stage 文件不存在时标记 `[MISSING]`，跳过该 stage 的 trace

### Few-shot

**Good trace**:
```
### Simulate: /diagnose "按钮颜色不对"

**Entry**
→ 匹配: /diagnose {description} → Quick diagnosis

**Stage: quick**
→ Step 1 (Symptom extraction): Gate=always → 进入
  提取: "按钮颜色不对" → 视觉偏差
→ Step 2 (Signal-based layer mapping): Gate=always → 进入
  Signal: CSS/UI → Layer: Code
→ Step 3 (Directional correction): Gate=always → 进入
  Direction: "检查按钮样式定义，对比设计稿色值"
→ Handoff: layer=Code, direction=检查样式

**Decision Points**
- Hard Rule "Quick does not read code": met → 未读取代码文件
- Escalation trigger "correction didn't work": not met → 保持 Quick
```

**Bad trace**:
```
### Simulate: /diagnose "按钮颜色不对"
AI 会先分析问题，然后找到原因，最后给出修复建议。
```
（无结构、无 gate 判定、无规则触发记录）

---

## Defaults

| Situation | Action |
|-----------|--------|
| 无路径参数 | [STOP:choose] 让用户提供路径 |
| 路径不存在 | 报错: "文件不存在: {path}" |
| 非 .md 文件 | 报错: "仅支持 Markdown 文件" |
| lint 0 条发现 | 输出 "指令质量良好"，不强制找问题 |
| simulate 无 stage 文件 | 只 trace SKILL.md 层面的行为，标记 [MISSING] |

## Recovery

| 场景 | 行为 |
|------|------|
| lint 扫描中 stage 文件读取失败 | 跳过该文件，报告中标注 "[SKIP] {path}: 读取失败" |
| simulate 推理遇到未定义的 stage 结构 | 标记 [MISSING]，继续 trace 已知部分 |
| 用户中断后恢复 | 无状态工具，重新执行即可 |
