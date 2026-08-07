---
name: pr-merge-review
description: >
  Use this skill when reviewing a Pull Request (PR) before merging a development branch
  into a target branch (e.g. main / develop). It performs comprehensive, detailed merge
  risk analysis: PR context understanding, code review, dev feature completeness analysis,
  textual + semantic + structural conflict detection, conflict resolution proposals, and
  a structured verification & testing plan. The final deliverable is a written Merge Report
  (merge_report.md) aimed at giving the PR submitter clear, prioritized, actionable feedback
  on merge risks. Trigger phrases: "审查这个PR", "review this PR", "合并风险评估",
  "可以合并吗", "merge branch", "合并到主分支", "代码评审", "PR review", "合并方案",
  "check this PR before merge". This skill ONLY produces a written report and does NOT
  execute the actual git merge, commit or push. It MUST use subagents for independent
  review / search / resolution tasks whenever subagent capability is available. Requires
  two inputs from the user: the target branch to merge into, and the single development
  branch (or PR) to be merged. If either is missing, ask for it before proceeding.
---

# PR 合并风险审查与报告生成器

帮助用户在一个开发分支（通常是他人提交的 PR）合并入目标分支之前，对合并风险进行**全面、详细的分析**，最终产出一份**结构化合并报告（`merge_report.md`）**，反馈给 PR 提交者。

**核心目标**：在合并前识别风险、量化风险、给出可操作的修复建议，让 PR 提交者明确知道：
- 这个 PR 能不能合并？
- 有哪些必须先修复的阻塞项？
- 有哪些值得改进但不阻塞的建议？
- 哪些功能/行为变更可能在合并后丢失或行为改变？

**本 skill 只生成报告，不执行实际的 `git merge` / `git commit` / `git push`。**
允许执行的 git 操作仅用于分析：`diff`、`log`、`status`、`rev-parse`、`merge-base`、`worktree` 临时合并探测等。临时合并只能发生在独立 worktree 中，完成后必须清理。

注意：生成 `merge_report.md` 本身会写入一个报告文件。除此之外，不应修改代码文件、提交历史、当前分支或用户当前工作区。

## 审查心态与原则

### 1. 审查心态

| 审查目标 ✅ | 非审查目标 ❌ |
|---|---|
| 捕获合并后会破坏功能的 Bug | 炫耀知识 |
| 识别合并冲突和语义冲突 | 挑剔代码格式（交给 linter） |
| 评估 dev 分支功能能否完整合入 | 不必要地阻塞进度 |
| 识别公共接口 / schema / migration 风险 | 按个人偏好重写代码 |
| 评估测试覆盖和回归风险 | |
| 给 PR 提交者可操作的修复建议 | |
| 建设性的代码审查文化 | |

### 2. 反馈原则

反馈应面向 **PR 提交者**，语气专业、尊重、协作，避免攻击性表达。每条建议都应包含：

- **位置**：文件 + 行号或代码片段
- **问题**：是什么
- **为什么重要**：风险/影响说明
- **如何修复**：具体的修改建议（如非显而易见）

对严重问题保持明确态度，避免模棱两可。对低置信度结论必须标注 `⚠️ AI 不确定，需人工确认`，不武断下结论。

### 3. 及早审查、频繁审查

核心机制：派发 subagent 在问题级联之前捕获它们。审查者获取的是**精心构造的上下文**，而非主 agent 的会话历史，以保护上下文窗口并聚焦工作产出本身。

## 必需输入

开始前确认拿到以下两项，缺一不可，缺失则向用户询问：

1. **目标分支（target branch）**：要合并入哪个分支，例如 `main` / `develop`
2. **开发分支（dev branch）/ PR**：需要被合并的分支或 PR 标识，例如 `feature/xxx`、`pull/123/head`

可选输入：

- PR 描述或关联 issue 链接（强烈建议提供，用于理解变更意图）
- 测试命令，例如 `npm test`、`pytest`
- 是否允许执行 `git fetch`
- 是否有 CI 配置文件可参考，例如 `.github/workflows/`、`.gitlab-ci.yml`
- `merge_report.md` 的保存位置（默认为仓库根目录）
- 团队编码规范文件路径（如有，用于对照检查）

## 整体流程

```text
Step 0: 前置校验与上下文收集
Step 1: 收集分支信息与改动清单
Step 2: AI 代码审查（subagent 并行）
Step 3: dev 功能完整性分析（subagent 并行）
Step 3.5: target 分支功能完整性保护分析（核心风险评估，subagent 复核）
Step 4: 冲突检测与分类（文本 / 语义 / 结构 / 特殊文件）
Step 5: 风险评估与严重程度分级
Step 6: 冲突解决方案设计（subagent 复核）
Step 7: 验证与测试方案
Step 8: 生成结构化合并报告（merge_report.md）
```

不要省略最终报告中的**dev 功能合入完整性矩阵**、**target 功能保护影响矩阵**、**风险评估清单**和**验证与测试方案**，这是核心产出。

本 skill 要求**必须使用 subagent** 辅助执行独立审查任务。若当前环境没有 subagent 能力，必须先向用户说明无法完整按本 skill 执行，并询问是否允许降级为主 agent 单独执行。

---

## Step 0: 前置校验与上下文收集

### 0.1 工作区状态检查

```bash
git status --porcelain
```

若输出非空，说明当前工作区有未提交改动。此时：

- 不允许在当前工作区执行 checkout、merge、rebase、reset 等会影响当前状态的操作
- 可以继续做只读分析
- 可以使用独立 `git worktree --detach` 做临时合并探测，因为它不会修改当前工作区
- 最终报告中必须注明：当前工作区存在未提交改动，并列出受影响文件

### 0.2 PR 上下文理解（关键步骤）

**理解变更目的与影响范围是审查的第一步**，不能跳过直接看 diff。

收集以下信息（若有）：

1. **PR 描述 / 标题**：变更目的是什么？解决什么问题？
2. **关联 issue**：业务需求是什么？验收标准？
3. **commit message**：开发分支的提交颗粒度和意图
4. **PR 大小评估**：改动行数、文件数。若 > 400 行 diff，应在报告中提示"PR 过大，建议拆分"以便后续审查
5. **CI/CD 状态**：若可获取，记录 CI 是否通过

```bash
git log "<target-ref>..<dev-ref>" --oneline
git log "<target-ref>..<dev-ref>" --format="%h %s%n%b" -20
```

若 PR 描述或 issue 信息缺失，应在报告中明确标注"缺乏业务上下文"，并在审查中**先提出疑问、说明风险、请求补充背景信息**，不武断下结论。

### 0.3 分支解析与有效性校验

优先基于本地已有 refs 分析。不要默认执行 `git fetch --all`。

如用户要求使用最新远端状态，或本地 refs 明显过旧，应先说明：

> `git fetch` 会更新本地远端跟踪引用，但不会修改当前工作区文件。

获得用户允许后再执行：

```bash
git fetch --all --prune
```

校验分支时，避免直接拼接模糊 ref。优先检查：

```bash
git rev-parse --verify --quiet "refs/heads/<branch>"
git rev-parse --verify --quiet "refs/remotes/<remote>/<branch>"
git merge-base "<target-ref>" "<dev-ref>"
```

分支解析策略：

- 若本地分支存在，记录其 commit
- 若远端跟踪分支存在，记录其 commit
- 若本地与远端不一致，标注 ahead / behind / diverged，并建议以远端最新状态为准重新分析
- 不默认远端一定叫 `origin`，必要时通过 `git for-each-ref refs/remotes` 查找候选
- 若目标分支或开发分支无法解析，停止并向用户询问
- 若 `git merge-base` 失败，停止后续分析，说明可能是分支名错误、仓库历史不相关或历史被改写

---

## Step 1: 收集分支信息与改动清单

以下命令中的 `<target-ref>` / `<dev-ref>` 使用 Step 0 确定的实际分析对象。

```bash
git log --all --oneline --graph -20
git merge-base "<target-ref>" "<dev-ref>"

git diff --name-status "<target-ref>...<dev-ref>"
git diff --stat "<target-ref>...<dev-ref>"
git diff "<target-ref>...<dev-ref>"

git log "<target-ref>..<dev-ref>" --oneline
git log "<dev-ref>..<target-ref>" --oneline
```

建立改动文件清单时，不使用 `--stat` 作为唯一来源。优先使用：

```bash
git diff --name-status -z "<target-ref>...<dev-ref>"
```

记录每个文件的状态：

- `A`: 新增
- `M`: 修改
- `D`: 删除
- `R`: 重命名
- `C`: 复制
- lockfile / generated file / binary file / migration / schema 等特殊文件单独标记

这份清单是 Step 2 的覆盖基准，后续必须确认每个文件都被审查过。

同时查找测试和 CI 信息：

```bash
find . -maxdepth 3 -name package.json -o -name pyproject.toml -o -name pytest.ini -o -name tox.ini
find .github/workflows .gitlab-ci.yml -type f 2>/dev/null
```

根据项目类型读取相关配置，例如：

- `package.json` scripts
- `pyproject.toml`
- `Makefile`
- CI workflow
- test 配置文件

---

## Step 2: AI 代码审查

分析开发分支相对目标分支的 diff，产出结构化审查结果。**审查优先级原则**：

```text
正确性 > 可维护性 > 性能优化 > 风格一致性
```

审查结论不能只由脚本或正则生成。可以使用 `git`、`rg`、`find` 等工具搜索引用、定位调用方、查配置项，但最终必须阅读相关上下文并做语义判断。

### 2.1 审查输出字段

- `summary`: 功能描述，1-2 句
- `files_changed`: 改动文件列表，含新增、修改、删除、重命名
- `modules_touched`: 涉及模块或组件
- `risk_level`: 高 / 中 / 低
- `public_api_changes`: 函数签名、导出接口、API 路由、事件、schema、数据结构、配置项变化
- `dependencies`: 该分支依赖或影响的模块
- `special_files`: lockfile、migration、schema、generated file、binary file 等
- `test_coverage`: 是否新增或修改测试，现有测试是否覆盖改动点
- `concerns`: 潜在 bug、边界情况、风格不一致、兼容性风险
- `strengths`: 本次变更做得好的地方（**审查前先肯定优点**，建立信任）

### 2.2 审查清单

对照以下维度逐项检查（**不是所有项都需要在报告中复述，但都应被审视过**）：

**逻辑与正确性**
- [ ] 边界情况是否处理（空值、空数组、off-by-one、超大输入）
- [ ] 异常路径是否正确处理
- [ ] 并发/竞态条件风险
- [ ] 事务边界是否正确

**安全性**
- [ ] 输入验证（SQL 注入、XSS、命令注入、SSRF、IDOR）
- [ ] 敏感信息是否硬编码（API_KEY、SECRET、TOKEN、PASSWORD）
- [ ] 权限校验是否完整（auth vs authorization）

**性能**
- [ ] N+1 查询
- [ ] 不必要的循环 / 内存泄漏
- [ ] 算法复杂度

**可维护性**
- [ ] 命名清晰、单一职责
- [ ] 函数过长（> 80 行）、嵌套过深（> 4 层）
- [ ] 是否 DRY 但避免过早抽象

**架构与集成**
- [ ] 设计决策是否合理，与周围代码集成是否干净
- [ ] 公共接口兼容性（破坏性变更？）
- [ ] 是否有安全隐患
- [ ] 与目标分支新增改动的交叉影响

**测试**
- [ ] 测试验证的是真实行为而非 Mock
- [ ] 边界情况是否覆盖
- [ ] 重要处是否有集成测试
- [ ] 改动点是否被现有测试覆盖

**生产就绪性**
- [ ] Schema 变更是否有迁移策略和回滚方案
- [ ] 向后兼容性
- [ ] 文档是否完整
- [ ] 是否有明显的 Bug

### 2.3 必须使用 subagent 执行代码审查

主 agent 先根据 Step 1 的改动文件清单拆分审查任务，再将至少一个独立审查任务分配给 subagent。拆分策略：

- diff 跨越多个模块时，按模块拆分，例如前端、后端、数据库、配置、CI
- diff 集中在单一模块时，也应至少分配一个 subagent 做独立复核，重点检查遗漏文件、公共接口变化、测试覆盖和潜在回归
- 特殊文件较多时，可单独分配 subagent 审查 lockfile、migration、schema、generated file、binary file

派发给 subagent 时，提供**精心构造的上下文**（PR 描述、改动文件清单、审查清单、输出格式要求），**不要传主 agent 的完整会话历史**。

主 agent 必须最终汇总 subagent 结果，并核对：

```text
已审查文件集合 == Step 1 改动文件清单
```

任何未覆盖文件都必须补充审查，即使只是格式、注释、文档或生成文件。

---

## Step 3: dev 功能完整性分析

必须从开发分支中提炼"功能/行为变更清单"，不能只停留在文件 diff 层面。目标是回答：**dev 分支中每一项可推断的功能、行为变化、接口变化或配置变化，是否能在合入 target 后保留下来**。

信息来源包括：

- commit message 和 commit 颗粒度
- `git diff <target-ref>...<dev-ref>` 中的代码改动
- 新增/修改的测试
- UI 入口、路由、菜单、页面、组件
- API route、CLI 命令、job、event、queue、RPC 方法
- schema、migration、配置项、环境变量、feature flag
- 文档、示例、fixtures、生成文件

必须使用 subagent 参与功能完整性分析。拆分策略：

- 按功能入口拆分，例如 UI / API / CLI / 后台任务
- 按模块拆分，例如前端 / 后端 / 数据库 / 配置
- 若功能数量很少，至少安排一个 subagent 独立复核功能清单是否完整

每个功能项至少记录：

- `feature_id`: 稳定编号，例如 `F1`、`F2`
- `name`: 功能名称或行为变化
- `intent`: 从 commit、diff、测试中推断出的功能意图
- `entry_points`: UI / API / CLI / job / event / config 等入口
- `files`: 涉及文件
- `dependencies`: schema、migration、env、feature flag、外部服务、生成代码等依赖
- `tests`: 已有或新增测试
- `target_interactions`: 与 target 分支新增改动的交叉影响
- `conflicts`: 关联的文本/语义/结构/特殊文件冲突编号
- `merge_completeness`: `可完整合入` / `需调整后合入` / `不确定` / `建议人工确认`
- `verification`: 合入后如何验证该功能仍然成立

功能清单覆盖度核对：

```text
已归类到功能项的改动文件 / Step 1 改动文件总数
已归类到功能项的 public_api_changes / Step 2 public_api_changes 总数
已归类到功能项的测试改动 / 测试改动总数
```

无法归类到功能项的文件不能静默忽略，必须放入 `unmapped_changes`，并说明原因，例如纯格式化、文档补充、生成文件、无法判断业务意图。

---

## Step 3.5: target 分支功能完整性保护分析（核心风险评估）

> **这是合并审查中最重要的一步。** dev 分支功能能否完整合入固然重要，但更关键的是：**合并后是否会破坏 target 分支（如 main / master）已有的功能、行为、接口和契约**。PR 提交者的改动是为 target 做贡献，不能以牺牲 target 现有功能为代价。

目标：回答 **"如果合并这个 PR，target 分支现有的哪些功能会受损、行为会改变、契约会被破坏"**。

### 3.5.1 建立 target 现有功能/契约清单

从 target 分支（相对 merge-base 的新增/修改部分，以及被 dev 改动触及的既有代码）中提炼：

```bash
base="$(git merge-base "<target-ref>" "<dev-ref>")"

# target 相对 base 的改动（理解 target 新做了什么）
git diff --name-status "$base..<target-ref>"

# dev 改动了 target 的哪些既有文件（这些是可能破坏 target 的热点）
git diff --name-status "$base..<dev-ref>" | grep -F "$(git diff --name-status "$base..<target-ref>" --name-only | head -100)"
```

提炼 target 功能/契约清单，每项至少记录：

- `target_feature_id`: 稳定编号，例如 `T1`、`T2`
- `name`: target 现有功能/行为/接口名称
- `entry_points`: UI / API / CLI / job / event / config / schema 等入口
- `files`: 涉及的 target 文件
- `contracts`: 对外契约（函数签名、API 路由、事件格式、schema、配置项、迁移约定）
- `existing_tests`: target 中验证该功能的现有测试

### 3.5.2 逐项评估 dev 改动对 target 功能的影响

对每个 `target_feature_id`，判断 dev 分支的改动是否：

| 影响类型 | 说明 | 严重程度倾向 |
|---|---|---|
| 🔴 **破坏行为** | dev 改变了 target 功能的运行时行为，导致原功能不再按预期工作 | blocking |
| 🔴 **破坏契约** | dev 修改/删除了 target 的公共接口、API 路由、事件格式、schema 字段 | blocking/critical |
| 🟠 **删除/重命名副作用** | dev 删除或重命名了 target 仍依赖的文件、函数、变量、路由 | critical |
| 🟠 **配置/环境不兼容** | dev 修改了配置项、环境变量、feature flag，使 target 功能行为改变 | critical |
| 🟠 **数据/migration 冲突** | dev 的 migration 与 target 的 migration 顺序、字段含义、表结构冲突 | critical |
| 🟡 **间接依赖** | dev 改动间接影响 target 功能所依赖的公共模块/工具函数 | important |
| 🟢 **无影响** | dev 改动与该 target 功能无交叉 | nit/无 |

每个 `target_feature_id` 至少记录：

- `impact_level`: `🔴 破坏` / `🟠 高风险` / `🟡 间接影响` / `🟢 无影响`
- `impact_description`: 具体如何被影响
- `evidence`: 引用 dev 侧 `file:line` 和 target 侧 `file:line` 的对照
- `affected_by`: 关联的 dev `feature_id` 和 `risk_id`
- `verification`: 合并后如何验证该 target 功能仍然正常
- `mitigation`: 保护/修复建议

### 3.5.3 重点关注的高风险破坏模式

必须主动排查以下"静默破坏"模式（这些通常不会被 git 文本冲突检测到）：

1. **公共接口破坏性变更**
   - dev 修改了 target 中被广泛调用的函数签名、返回值结构
   - dev 删除/重命名了 target 公共导出的函数、类、常量
   - 对每个 `public_api_changes`，在 target 全局搜索引用，确认无遗漏调用方

2. **运行时行为改变**
   - dev 修改了 target 的默认值、阈值、重试逻辑、超时、限流策略
   - dev 改变了 target 的事件触发顺序、副作用顺序、中间件链
   - dev 关闭/绕过了 target 的校验、鉴权、日志、监控

3. **数据/schema 破坏**
   - dev 的 migration 删除/重命名了 target 使用的表或字段
   - dev 的 schema 变更与 target 的 schema 读取方不兼容
   - dev 修改了 target 共享的数据模型，导致 target 既有读写逻辑失效

4. **配置/环境破坏**
   - dev 移除/重命名了 target 依赖的配置项或环境变量
   - dev 改变了 feature flag 默认值，影响 target 既有行为

5. **测试/CI 破坏**
   - dev 删除/修改了 target 的既有测试，可能掩盖回归
   - dev 修改了 CI 配置，可能跳过 target 的关键检查
   - dev 改动了共享 fixtures、mock、工厂方法

6. **隐式依赖破坏**
   - dev 修改了 target 依赖的公共工具函数、中间件、基类
   - dev 改动了 target 功能所读取的共享状态（全局变量、缓存、单例）

### 3.5.4 必须使用 subagent 独立复核 target 保护分析

target 功能保护分析是高风险环节，必须至少安排一个 subagent 独立复核：

- 复核 `target_feature_id` 清单是否完整覆盖了被 dev 改动触及的 target 功能
- 复核每项 `impact_level` 判断是否准确，是否有遗漏的破坏点
- 对每个 `public_api_changes` 在 target 全局做引用搜索，确认调用方清单完整

主 agent 汇总 subagent 结论后，必须核对覆盖度：

```text
已评估的 target 功能项数 / dev 改动触及的 target 既有功能总数
target 公共接口引用搜索覆盖率：已搜索 / 总数
```

任何被 dev 改动触及但未评估的 target 功能，都必须补评估或明确标注"无法评估，原因：..."。

---

## Step 4: 冲突检测与分类

### 4.1 文本冲突

使用独立 worktree 做合并探测，不切换、不修改用户当前工作区。

推荐流程：

```bash
probe_dir="$(mktemp -d /tmp/merge-probe.XXXXXX)"
git worktree add --detach "$probe_dir" "<target-ref>"

cd "$probe_dir"
git merge --no-commit --no-ff "<dev-ref>"

git diff --name-only --diff-filter=U
```

清理要求：

```bash
git merge --abort 2>/dev/null || true
cd -
git worktree remove --force "$probe_dir"
```

实现时应保证异常路径也清理 worktree。若 shell 环境允许，使用 `trap` 管理清理逻辑。

记录：

- 是否发生文本冲突
- 冲突文件总数
- 每个冲突文件中的冲突块编号，例如 `C1a`、`C1b`
- 每个冲突块双方意图
- 每个冲突块影响的 dev 功能项 `feature_id`
- 是否存在 rename/delete、modify/delete、add/add 等特殊冲突

上下文读取原则：

- 至少读取冲突块前后 50 行
- 若冲突位于函数、类、路由、配置块、migration、schema 中，应读取完整结构化单元
- 不允许只看冲突标记内的几行就给解决方案

### 4.2 双向语义冲突

即使 git 没报告文本冲突，也必须检查语义冲突。

先确定共同祖先：

```bash
base="$(git merge-base "<target-ref>" "<dev-ref>")"
```

分别分析两边新增变化：

```bash
git diff "$base..<target-ref>"
git diff "$base..<dev-ref>"
```

建立双向矩阵（**优先看 dev 对 target 的破坏**，再看 target 对 dev 的影响）：

1. **dev 破坏 target 现有功能**（优先级更高，这是合并的核心风险）
   - dev 修改/删除/重命名的接口，target 现有代码（base→target 的改动及既有代码）是否仍按旧方式调用
   - dev 修改 schema / config / 数据模型，target 现有功能是否不兼容或行为改变
   - dev 是否改变了 target 功能的运行时行为、默认值、副作用顺序
   - 每项发现必须关联到 Step 3.5 的 `target_feature_id`

2. **target 影响 dev 新代码**（次要，评估 dev 功能能否保留）
   - target 修改/删除/重命名的接口，dev 新增代码是否仍按旧方式调用
   - target 修改 schema / config / 数据模型，dev 新代码是否不兼容
   - 每项发现必须关联到 Step 3 的 `feature_id`

重点检查：

- 函数签名和返回值变化
- API route / event / queue topic / RPC 方法变化
- 类型、schema、protobuf、OpenAPI、GraphQL 变化
- 数据库 migration 顺序和字段含义
- 配置项、环境变量、feature flag
- 文件删除或重命名后的旧路径引用
- 同名变量、路由、字段、migration 编号但定义不同

对 Step 2 中每一项 `public_api_changes` 都必须完成引用搜索，并给出结论：

- `无引用`
- `有引用，需要同步处理`
- `不确定，需要人工确认`

每一项语义冲突都必须关联到 Step 3 的一个或多个 `feature_id`。若无法关联，必须说明该冲突属于全局架构/配置风险，不能遗漏。

### 4.3 结构冲突与特殊文件冲突

单独检查以下文件类型：

- lockfile：`package-lock.json`、`pnpm-lock.yaml`、`yarn.lock`、`uv.lock`、`poetry.lock`
- migration：数据库迁移文件、版本号、执行顺序
- schema：OpenAPI、GraphQL、protobuf、数据库 schema
- generated files：自动生成代码是否应重新生成
- binary files：图片、模型、PDF、Office 文件等
- build / CI 配置：依赖版本、构建矩阵、部署脚本

这些文件即使没有文本冲突，也要标注风险和建议处理方式。

每一项结构冲突或特殊文件风险都必须说明其影响的：
- **dev 功能项**（Step 3 的 `feature_id`）：dev 功能能否完整合入
- **target 功能项**（Step 3.5 的 `target_feature_id`）：是否破坏 target 现有功能/契约

尤其是 migration、schema、lockfile、generated file 对 **target 既有数据、既有接口契约、既有运行时行为** 的影响。

---

## Step 5: 风险评估与严重程度分级

对 Step 2-4 发现的所有问题、冲突、风险项进行**统一分级**。这是面向 PR 提交者反馈的核心。

### 5.1 严重程度标签体系

| 标签 | 含义 | 是否阻塞合并 | 修复时机 |
|---|---|---|---|
| 🔴 `[blocking]` | 合并前必须修复：会导致功能损坏、数据丢失、安全漏洞、合并后无法运行 | 阻塞 | 合并前 |
| 🟠 `[critical]` | 严重风险：正确性 Bug、破坏性接口变更未处理、migration 顺序错误、测试缺失关键路径 | 阻塞 | 合并前 |
| 🟡 `[important]` | 重要风险：架构问题、错误处理不当、性能隐患、测试覆盖不足 | 不阻塞但应处理 | 本 PR 或下个 PR |
| 🟢 `[nit]` | 小问题：命名、注释、轻微重构 | 不阻塞 | 可选 |
| 💡 `[suggestion]` | 改进建议：考虑替代方案、未来优化方向 | 不阻塞 | 后续迭代 |
| 📚 `[learning]` | 教育性评论：分享最佳实践，无需立即操作 | 不阻塞 | 仅参考 |
| 🎉 `[praise]` | 做得好的地方，鼓励继续保持 | 不阻塞 | — |

> **严重性层级**：🔴/🟠 是阻塞项，合并前必须处理；🟡 应处理；🟢 可选。其余标签为非阻塞注解。
> **校准原则**：按**实际严重程度**分类，并非一切都是 Critical。不要把代码风格挑剔标记为阻塞项。

### 5.2 风险量化

对每项风险记录：

- `risk_id`: 稳定编号，例如 `R1`、`R2`
- `severity`: 上表的标签
- `category`: 类型，例如 `correctness` / `security` / `conflict-text` / `conflict-semantic` / `conflict-structural` / `migration` / `test-coverage` / `breaking-change` / `target-regression` / `performance` / `maintainability`
- `impact_direction`: `dev→target（破坏 target）` / `target→dev（影响 dev）` / `双向` / `无方向`
- `location`: 文件:行号
- `description`: 问题描述
- `impact`: 为什么重要，对合并后行为的影响
- `affected_dev_features`: 关联的 dev `feature_id` 列表（Step 3）
- `affected_target_features`: 关联的 target `target_feature_id` 列表（Step 3.5）
- `fix_suggestion`: 具体修复建议
- `confidence`: `✅ AI 已给出确定方案` 或 `⚠️ AI 不确定，需人工确认`
- `manual_review_required`: 是否需要人工确认，及原因

### 5.3 整体风险评估

综合所有风险项，给出：

- **阻塞项数量**（🔴 + 🟠）
- **重要项数量**（🟡）
- **小问题数量**（🟢 + 💡 + 📚）
- **破坏 target 功能的数量**（Step 3.5 中 `impact_level` 为 🔴/🟠 的项数，这是合并风险的核心指标）
- **合并风险等级**：`高` / `中` / `低`
  - 高：存在阻塞项，或存在 🔴 破坏 target 功能的项 → **不建议合并**
  - 中：存在重要项或 🟠 高风险破坏项，但无 🔴 破坏 → **修复后可合并**
  - 低：仅有小问题/建议，无 target 功能破坏 → **可合并，后续跟进**

---

## Step 6: 冲突解决方案设计

对每个文本冲突块和语义冲突项给出解决建议。

每个条目至少包含：

- 冲突编号
- 关联的 `feature_id` 和 `risk_id`
- 文件路径
- 冲突类型：文本 / 语义 / 结构 / 特殊文件
- 严重程度：使用 Step 5 的标签体系
- 双方意图说明
- 建议解决方案
- **置信度**: `✅ AI 已给出确定方案` 或 `⚠️ AI 不确定，需人工确认`
- 需要人工确认的点
- 对应验证方式
- 对 dev 功能完整性的影响：完整保留 / 需补充修改 / 不确定 / 可能改变行为

若给出合并后代码片段，应保留双方意图，并明确标注置信度（用直观中文，便于 PR 提交者理解）：

```text
✅ AI 已给出确定方案
```

或：

```text
⚠️ AI 不确定，需人工确认，原因：...
```

必须使用 subagent 辅助冲突解决方案设计。拆分策略：

- 冲突数量较多且互不依赖时，按冲突点分配给多个 subagent 并行分析
- 冲突数量较少时，至少分配一个 subagent 做独立复核，检查主 agent 的解决方案是否保留双方意图、是否遗漏语义影响、是否需要标注 `⚠️ AI 不确定，需人工确认`
- 语义冲突和特殊文件冲突可以单独分配给 subagent 做引用搜索或风险复核

主 agent 必须做最终一致性检查，确认多个解决方案之间不会产生新的冲突或行为不一致。

覆盖度核对：

```text
文本冲突块覆盖率：已处理 / 总数
语义冲突项覆盖率：已处理 / 总数
public_api_changes 引用搜索覆盖率：已处理 / 总数
改动文件审查覆盖率：已处理 / 总数
dev 功能项冲突映射覆盖率：已映射 / 总功能项
```

任一项不是 100% 时，不能静默生成最终报告，必须补齐或明确标注无法完成的原因。

---

## Step 7: 验证与测试方案

为每个 dev 功能项和每个冲突解决点设计**具体到"跑什么、检查什么"**的验证方案。

### 7.1 应运行的测试命令

根据 Step 1 收集的项目配置，列出：

- 单元测试命令，例如 `npm test`、`pytest tests/`
- 集成测试命令
- lint 检查命令
- 类型检查命令，例如 `tsc --noEmit`、`mypy`
- build 命令

### 7.2 针对每个 dev 功能项的验收检查

每个 `feature_id` 对应：

- 验证该功能在合并后仍然成立的检查方式
- 应执行的具体测试用例或手动操作步骤
- 期望结果

### 7.3 针对每个冲突解决点的测试建议

每个冲突解决方案对应：

- 解决后应验证什么
- 回归检查点
- 是否需要补充测试用例

### 7.4 接口变更回归检查

对每项 `public_api_changes`：

- 受影响的调用方清单
- 应执行的回归测试
- 是否需要破坏性变更通知

### 7.5 CI 检查建议

- 应配置或检查的 CI 项
- 是否需要更新 CI workflow

### 7.6 人工复核清单

列出需要人工确认的点（标注 `⚠️ AI 不确定，需人工确认` 的项），说明：

- 为什么需要人工确认
- 应由谁确认（PR 提交者 / 仓库维护者 / 业务负责人）
- 确认时应关注什么

### 7.7 灰度和回滚策略（如适用）

- 是否建议分批合并
- 回滚步骤
- feature flag 使用建议

---

## Step 8: 生成结构化合并报告

最终产物为 `merge_report.md`。

若存在 `references/merge_report_template.md`，优先使用该模板。若模板不存在，使用以下默认结构生成。

### 8.1 报告必须包含的章节

1. **报告概要**
   - PR/分支信息（目标分支、开发分支、实际分析 refs、merge base）
   - 是否执行 fetch、本地与远端是否一致、当前工作区是否干净
   - PR 意图摘要（来自 PR 描述或 commit message）
   - **合并决策建议**：`Approve` / `Approve with fixes` / `Request Changes` / `Block`
   - **合并风险等级**：`高` / `中` / `低`
   - 阻塞项数量、重要项数量、小问题数量
   - 一段话总体评价

2. **变更概览**
   - 功能摘要
   - commit 列表
   - 改动文件清单（含状态标记）
   - 模块影响范围
   - 风险等级
   - PR 大小评估（是否过大、是否建议拆分）

3. **优点**
   - 本次变更做得好的 2-5 点（**先肯定优点，建立信任**）
   - 引用具体文件:行号

4. **代码审查结果**
   - public API changes
   - dependencies
   - special files
   - test coverage
   - concerns
   - 审查覆盖度统计

5. **dev 功能合入完整性矩阵**
   - 功能项编号
   - 功能名称或行为变化
   - 入口点
   - 涉及文件
   - 依赖项
   - 关联冲突或风险
   - 合并完整性判断
   - 验证方式

   推荐表格：

   ```markdown
   | 功能项 | 入口点 | 涉及文件 | 冲突/风险 | 解决建议 | 验证方式 | 结论 |
   |---|---|---|---|---|---|---|
   ```

6. **target 功能保护影响矩阵**（核心章节，优先展示）
   - target 功能项编号
   - target 功能/契约名称
   - 入口点
   - 涉及的 target 文件
   - dev 改动对该功能的影响等级（🔴 破坏 / 🟠 高风险 / 🟡 间接 / 🟢 无影响）
   - 影响说明
   - 关联的 dev 改动和风险编号
   - 保护/修复建议
   - 合并后验证方式

   推荐表格：

   ```markdown
   | target 功能 | 入口点 | 涉及 target 文件 | 影响等级 | 影响说明 | 关联 dev 改动 | 保护建议 | 验证方式 |
   |---|---|---|---|---|---|---|---|
   ```

   > 若存在任何 🔴 破坏项，必须在报告概要中高亮标注，并列为合并前必须处理的阻塞项。

7. **未映射改动**
   - 无法归类到具体 dev 功能项的文件
   - 原因说明
   - 是否影响合并完整性

8. **冲突清单**
   - 文本冲突
   - 语义冲突
   - 结构冲突
   - 特殊文件冲突

9. **风险评估清单**（核心章节，面向 PR 提交者）

   按严重程度分组，**阻塞项优先展示**，**target 功能破坏项次优先**：

   ```markdown
   ### 🔴 阻塞项（合并前必须修复）

   #### R1: [问题描述]
   - **严重程度**: 🔴 blocking
   - **位置**: src/auth.ts:42-58
   - **关联功能**: F2
   - **问题**: [具体说明]
   - **为什么重要**: [风险/影响]
   - **修复建议**: [具体建议]
   - **置信度**: `✅ AI 已给出确定方案` / `⚠️ AI 不确定，需人工确认`

   ### 🟠 严重项

   ...

   ### 🟡 重要项

   ...

   ### 🟢 小问题 / 💡 建议 / 📚 学习 / 🎉 亮点
   ...
   ```

10. **冲突解决方案**
   - 每个冲突点的建议处理方式
   - 关联的 dev 功能项和 risk_id
   - 合并后代码片段，如适用
   - **置信度**: `✅ AI 已给出确定方案` / `⚠️ AI 不确定，需人工确认`
   - 人工确认点

11. **验证与测试方案**
    - 应运行的测试命令
    - 针对每个 dev 功能项的验收检查
    - 针对每个冲突解决点的测试建议
    - 接口变更后的回归检查点
    - CI 检查建议
    - 人工复核清单
    - 灰度和回滚策略，如适用

12. **覆盖度统计**
    - 改动文件审查覆盖率
    - dev 功能项覆盖率
    - **target 功能项保护评估覆盖率**
    - 未映射改动数量
    - 文本冲突覆盖率
    - 语义冲突覆盖率
    - 冲突到功能项映射覆盖率
    - public API 引用搜索覆盖率

13. **合并决策总结**
    - dev 分支功能是否可完整合入 target
    - **合并后是否会破坏 target 现有功能**（核心判断）
    - 哪些功能可完整合入
    - 哪些功能需调整后合入
    - 哪些功能不确定或需人工确认
    - 哪些 target 现有功能会被破坏（必须列出）
    - 合并前必须处理的阻塞项清单
    - **最终建议**：`Approve` / `Approve with fixes` / `Request Changes` / `Block`
    - **理由**：1-2 句技术评估说明

14. **限制与假设**
    - 是否未 fetch
    - 是否未执行实际合并探测
    - 是否存在脏工作区
    - 是否有无法确定的业务语义
    - 是否缺乏 PR 描述/业务上下文

### 8.2 报告语气与格式要求

- 使用简洁专业的中文，适当穿插技术术语（幂等性、事务边界、死锁风险、破坏性变更等）
- 对严重问题保持明确态度，避免模棱两可
- 保持尊重与合作语气，避免攻击性表达
- 所有建议都配上"为什么"与"怎么改"两部分说明
- 报告长度适中，让 PR 提交者能在几分钟内理解重点
- 严重问题优先展示
- 报告应可直接复制到 PR 评论中

### 8.3 交付

交付时向用户说明 `merge_report.md` 的保存位置，并简要总结：

- 合并决策建议
- 最高风险项（前 3 项）
- PR 提交者下一步应做什么

---

## 注意事项

- 不执行真实合并，不提交，不推送
- 必须使用 subagent；若环境不支持 subagent，需先向用户说明并请求是否允许降级
- 不在当前工作区执行 merge / rebase / checkout / reset
- 临时合并探测必须使用独立 `worktree --detach`
- 临时 worktree 必须使用唯一目录，完成后清理
- 不默认远端名为 `origin`
- 不默认本地分支就是最新远端状态
- 不使用 `--stat` 作为唯一文件清单来源
- 必须提炼 dev 分支功能/行为变更清单，并建立功能合入完整性矩阵
- **必须评估 dev 改动对 target 现有功能/契约的破坏风险，这是合并审查的核心**（不能只看 dev 功能能否合入）
- 每个冲突和高风险项都必须尽量映射到受影响的 dev 功能项 **和** target 功能项
- 不只看文本冲突，必须做双向语义冲突分析，**优先看 dev 对 target 的破坏**
- 对 lockfile、migration、schema、generated file、binary file 单独标注其对 target 既有契约的影响
- 低置信度结论必须标注 `⚠️ AI 不确定，需人工确认`
- 验证方案必须具体到"跑什么、检查什么"
- 如果无法完成某项检查，必须在最终报告中明确说明原因和残余风险
- 报告面向 PR 提交者，语气专业、尊重、协作
- 严重程度按实际分级，不要把代码风格挑剔标记为阻塞项
- 先肯定优点，再指出问题，建立信任
- 缺乏业务上下文时先提出疑问，不武断下结论

## Subagent 使用原则

本 skill **必须使用 subagent**。subagent 不是可选优化，而是合并审查的强制交叉检查机制。**核心设计**：派发 subagent 时提供精心构造的上下文（PR 描述、改动文件清单、审查清单、输出格式），不传主 agent 的完整会话历史——这样 subagent 聚焦于工作产出本身，且 diff 和评估都在 subagent 上下文中进行，只有发现结果返回给主 agent，保护主 agent 的上下文窗口。

必须至少安排 subagent 参与以下任务中的两类：

- 代码审查复核
- public API change 引用搜索
- 语义冲突分析
- 冲突解决方案复核
- 特殊文件风险检查
- **target 功能保护分析复核**（高优先级）

当任务很小、没有文本冲突、改动文件很少时，仍必须至少安排一个 subagent 执行独立复核，并在 `merge_report.md` 中记录 subagent 的覆盖范围和结论。

subagent 适用于互不依赖、可独立完成的子任务：

- 按模块审查 diff
- 按 public API change 搜索引用
- 按冲突点分析解决方案
- 按特殊文件类型检查风险

subagent 不适用于：

- Step 0 前置校验
- Step 1 基础 git 信息收集
- Step 4.1 临时 worktree 合并探测
- Step 8 最终报告汇总

主 agent 始终负责：

- 分支解析
- 安全边界判断
- 覆盖度核对
- 跨模块语义冲突判断
- 风险严重程度最终分级
- 最终 `merge_report.md` 成文

## 红旗清单

**绝不：**

- ❌ 因为"很简单"就跳过审查
- ❌ 忽略 🔴 阻塞项
- 🟠 严重项未处理就建议合并
- ❌ 在 🟡 重要项未修复的情况下建议直接 Approve（除非明确说明后续跟进）
- ❌ 争论有效的技术反馈
- ❌ 因为缺乏业务上下文就武断下结论
- ❌ 把代码风格挑剔标记为 🔴 阻塞项
- ❌ 对未实际阅读的代码给出反馈
- ❌ 含糊其辞（如"改进错误处理"）而不给出具体位置和修复方式
- ❌ 回避给出明确合并决策
