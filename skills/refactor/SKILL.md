---
name: refactor
description: 重构工具集。单入口路由分发。Use when user says "refactor", "重构", "压缩skill", "优化文档", or invokes /refactor.
---

# Refactor

`/refactor {subcommand} [target]` → route → execute.

## Subcommands

| Subcommand | Alias | Description |
|------------|-------|-------------|
| `skill` | `s` | 重构并压缩 Skill Markdown 文档 |

未匹配任何子命令时，显示此表并提示用户选择。

## Routing

```
/refactor {input}
1. 第一个词匹配子命令名或alias？ → 进入对应流程
2. 输入为 .md 文件路径？         → 默认 skill 子命令
3. 否则                         → 显示帮助
```

---

## Subcommand: skill

重构并压缩 Skill Markdown 文档。消除冗余、统一术语、最大化信息密度。

### 输入

- `/refactor skill {path}` — 指定文件路径
- `/refactor skill` — 无路径时，`[STOP:choose]` 让用户提供路径或从当前目录选择 .md 文件

### Rules

1. **语义守恒** — 不丢失任何原始语义，不引入新假设。
2. **信息密度最大化** — 用更短的表达替换冗长描述，不降低精度。
3. **单一术语** — 同义词只保留一个主称呼，其余统一替换。
4. **冲突解决** — 发现矛盾约定时，选择更合理的一版并统一全文。
5. **隐式显式化** — 多处体现但未定义的规则，提取为显式条目。
6. **原文不保留** — 轻量文档完全重写；标准文档保留结构重写内容；成熟文档仅重写问题 section（见复杂度分层）。

### 复杂度分层

Step 1 (Read) 后自动评估，结果在 Step 3 (Plan) 中展示。用户可覆盖。

| 类型 | 条件 | 分析范围 | 重写策略 | Rule 6 行为 |
|------|------|---------|---------|------------|
| 轻量 | ≤200 行 或 缺 ≥3 个必需 section | 全文 | 按目标结构完全重写，建新术语表 | 原文不保留 |
| 标准 | 201-500 行，结构基本完整 | 全文 | 保留原 section 划分和顺序，逐 section 压缩内容，基于原术语表统一 | 结构保留，内容重写 |
| 成熟 | >500 行，≥5 个 section 存在 | 仅问题 section | 只重写标记了问题的 section，其余原样保留 | 仅重写问题 section |

必需 section: Core Principles, Definitions, Rules, Workflow（来自目标结构）。

### Workflow

```
Step 1: Read      — 读取目标文件全文
Step 2: Classify  — 复杂度分层（轻量/标准/成熟）
Step 3: Analyze   — 识别问题（范围受复杂度分层控制）：
                    - 重复/语义重叠段落
                    - 术语不一致（列出 → 统一映射表）
                    - 冲突约定
                    - 隐式规则
                    - 冗长表达
                    成熟模式: 只分析有问题的 section
Step 4: Plan      — 生成重构计划摘要（不超过 10 行），含复杂度分类结果
                    [STOP:confirm] 等待用户确认（用户可覆盖分类）
Step 5: Rewrite   — 按分层策略重写
Step 6: Verify    — 语义守恒检查（见下方）
Step 7: Output    — 将重构结果写回原文件（或用户指定的新路径）
```

### 语义守恒验证 (Step 6)

重写完成后、写入文件前执行：

1. 从原文提取语义清单（每个 section 的核心指令，格式: "动词+对象"，1行1条）
2. 从重写版提取同样格式的语义清单
3. 逐条对比：✓ 保留 | ⚠ 合并（多条→一条，语义未丢） | ✗ 丢失
4. 输出对比报告：
   ```
   ### 语义守恒检查
   - 原文语义: {N} 条
   - 保留: {N} | 合并: {N} | 丢失: {N}
   
   **丢失项** (如有):
   - {原始语义} — 来自 {section}
   ```
5. 丢失 >0 → 修复丢失项，重新执行 Step 6
6. 丢失 =0 → 继续 Step 7

成熟模式: 只对变更 section 执行语义守恒检查。

### 目标结构

重构后的文档必须遵循以下结构（按顺序）：

```markdown
---
(保留原始 frontmatter，按需更新 description)
---

# {Skill Name}

`/command {args}` → brief flow description.

## Core Principles
(3-7 条核心原则)

## Definitions
(关键术语表，| Term | Meaning | 格式)

## Rules
(硬性规则，编号列表)

## Workflow
(执行流程，用代码块或编号步骤)

## Examples
(最少但有代表性的示例)

## Edge Cases
(可选，仅当有必要时保留)
```

### 执行要点

- **删除**：重复段落只保留信息更完整的一处。
- **合并**：相似规则抽象为通用原则。
- **压缩**：每条规则/原则不超过 2 行。
- **术语表**：在 Definitions 中定义所有关键术语，正文中统一使用。
- 不输出修改说明，不解释过程，直接输出最终文档。
- 标准模式: 变更 section 标记 `[重写]`，未变更 section 不标记。
- 成熟模式: 变更 section 标记 `[重写]`，未变更 section 标记 `[原样保留]`。
