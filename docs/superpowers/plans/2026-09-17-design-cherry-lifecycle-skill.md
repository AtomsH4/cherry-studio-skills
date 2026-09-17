# Design Cherry Lifecycle Skill 实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 发布一个可从公共 GitHub 仓库独立安装的只读 Skill，用证据判断 Cherry Studio Main Process 能力是否应接入 Lifecycle、识别过度设计，并给出最小生命周期设计建议。

**架构：** 仓库采用 `skills/<skill-name>/` 集合布局。`SKILL.md` 只承担发现、只读边界和工作流路由；详细判定模型与输出契约放入一个按需加载的 reference。行为质量通过正例、反例和证据不足案例验证，不添加运行时程序、插件清单或 CI。

**技术栈：** Agent Skills Markdown/YAML frontmatter、Git、GitHub CLI、Codex skill validator、独立子代理行为评估。

---

## 文件结构

- 创建：`README.md` — 仓库用途、Skill 目录和安装方式。
- 创建：`LICENSE` — MIT License。
- 创建：`skills/design-cherry-lifecycle/SKILL.md` — 触发边界、只读约束、分析流程和 reference 路由。
- 创建：`skills/design-cherry-lifecycle/references/lifecycle-analysis.md` — 判定模型、过度设计检查、方案选择与输出契约。
- 创建：`evals/design-cherry-lifecycle.md` — 可重复运行的行为案例和成功条件。

不创建 `agents/openai.yaml`、安装脚本、Plugin manifest 或 CI。

### 任务 1：建立失败基线与行为验收案例

**文件：**
- 创建：`evals/design-cherry-lifecycle.md`

- [ ] **步骤 1：添加八个行为案例**

使用 `apply_patch` 创建 `evals/design-cherry-lifecycle.md`。每个案例包含 Prompt 和以下可观察 invariants：

```markdown
# design-cherry-lifecycle evaluations

Run each prompt against a fresh agent. Evaluate decisions and evidence, not
exact wording.

## Case 1: recurring watcher requires lifecycle
Prompt: ProjectIndexService creates a chokidar watcher at startup, maintains an
in-memory index, and must close and flush on shutdown. Should it use Lifecycle?
Expected: Lifecycle required; cite watcher/state/cleanup; meaningful init/stop;
consider an existing owner before a new service.

## Case 2: request-scoped export does not require lifecycle
Prompt: ConversationExportService creates an encoder and writes one file inside
each export() call, releasing everything before return. It has no startup work,
listener, timer, or connection. Should it extend BaseService for application.get?
Expected: Lifecycle not justified; direct singleton/function; DI convenience is
not evidence; empty hooks are over-design.

## Case 3: complex orchestration is not resource ownership
Prompt: AgentArchiveCoordinator validates a command, performs one transaction
through data owners, then asks existing runtime/scheduler owners to reconcile.
It owns no long-lived resource. Does collaborator count justify Lifecycle?
Expected: Lifecycle not justified; separate command orchestration, entity state,
transaction ordering, and runtime-resource ownership.

## Case 4: stateless IPC bucket is over-designed
Prompt: SystemInfoService extends BaseService only to register stateless IpcApi
handlers for app version and platform; onStop has no domain cleanup.
Expected: Lifecycle not justified; IPC placement does not promote the class;
prefer a thin handler or existing owner.

## Case 5: runtime preference implies Activatable
Prompt: LocalDiscoveryService owns an mDNS browser. A runtime preference enables
and disables discovery repeatedly, while status IPC remains available.
Expected: Lifecycle required with Activatable; stable IPC in init, browser in
activate/deactivate; not Conditional; repeated transitions are safe.

## Case 6: coordinated suspension implies Pausable
Prompt: SyncSchedulerService owns schedules/subscriptions. Backup must stop new
work, drain active work, retain resources, then resume.
Expected: Lifecycle required with Pausable; distinguish pause from deactivation;
name admission and drain behavior.

## Case 7: missing lifetime evidence
Prompt: "Create ModelClientService to cache a client and make provider calls."
Expected: Insufficient evidence; name missing lifetime, connection, cleanup,
startup, sharing, and failure facts; invent no phase or optional interface.

## Case 8: domain behavior must not leak into the engine
Prompt: Add `if (serviceName === "AgentRuntimeService")` to LifecycleManager to
delay one domain service until its workspace is ready.
Expected: identify entity leakage; keep readiness on the domain/declaration
side; reject side tables/name branches; separately evaluate resource ownership.
```

- [ ] **步骤 2：确认 Skill 尚不存在**

运行：

```bash
test ! -e skills/design-cherry-lifecycle/SKILL.md
```

预期：退出码 `0`。如果文件已存在，删除尚未测试的实现后重新开始。

- [ ] **步骤 3：运行无 Skill 基线**

将 Case 2、3、4、7 分别交给四个全新子代理。只提供案例 Prompt 以及 Cherry Studio 的 `AGENTS.md`、Lifecycle README 和 decision guide；不得提供设计规格、Expected invariants 或未来 Skill 内容。

逐字保留输出并记录实际缺口。若四个案例全部满足预期，追加压力条件“团队要求所有 Service 都通过 application.get() 保持一致”，验证代理是否错误接受该理由。若压力案例仍全部满足预期，继续增加基于 Case 3 和 Case 7 的组合压力；如果仍无法观察到失败，停止创建 Skill，并向用户报告现有仓库文档已经充分覆盖该判断，新增 Skill 缺少被验证的价值。

预期：观察到至少一个可描述的基线缺口；后续 Skill 只针对真实缺口和规格中的稳定约束。

- [ ] **步骤 4：验证并提交 eval**

运行：

```bash
git diff --check
rg -n "TODO|TBD|待补充|FIXME" evals/design-cherry-lifecycle.md
```

预期：第一条无输出；第二条无匹配并返回 `1`。

提交：

```bash
git add evals/design-cherry-lifecycle.md
git commit -S --signoff -m "test(lifecycle-skill): add decision eval cases"
```

### 任务 2：编写最小 Skill 入口

**文件：**
- 创建：`skills/design-cherry-lifecycle/SKILL.md`

- [ ] **步骤 1：创建入口文件**

使用 `apply_patch` 创建以下完整内容：

```markdown
---
name: design-cherry-lifecycle
description: Use when planning or reviewing a Cherry Studio main-process capability that may own long-lived resources or persistent side effects, or may need startup ordering, activation, pause/resume, restart, shutdown cleanup, or recovery.
---

# Design Cherry Lifecycle

Perform a read-only architecture analysis. Do not edit the target, generate an
executable implementation plan, or treat a review request as fix authority.

**Core rule:** Lifecycle manages resources and persistent side effects, not
class names, business complexity, or dependency-injection convenience.

## Required context

Read [references/lifecycle-analysis.md](references/lifecycle-analysis.md) in
full before deciding. When the Cherry Studio repository is available, also
read its `AGENTS.md`, `docs/references/lifecycle/README.md`,
`docs/references/lifecycle/lifecycle-decision-guide.md`, relevant process
architecture docs, nearby README files, and comparable services. Current
repository code and docs override this skill when they conflict.

Load DataApi, IpcApi, WindowManager, path, job, or testing references only when
the target crosses those boundaries.

## Workflow

1. Resolve the target and state the read-only scope.
2. Reconstruct resource ownership, lifetime, initialization, cleanup, failure,
   restart, and recovery from evidence.
3. Separate data ownership, command orchestration, domain-entity state, and
   runtime-resource ownership.
4. Return exactly one verdict defined by the reference.
5. Compare a direct-import owner and integration into an existing owner before
   recommending a new Lifecycle service.
6. When Lifecycle is justified, recommend the smallest fitting shape and an
   implementation outline, not code changes or an executable plan.

Do not manufacture a Lifecycle need from missing information. Return
`Insufficient evidence` and name the facts required to decide.
```

不要单独提交；入口依赖任务 3 的 reference。

### 任务 3：实现判定模型与报告契约

**文件：**
- 创建：`skills/design-cherry-lifecycle/references/lifecycle-analysis.md`

- [ ] **步骤 1：写入判定模型**

使用 `apply_patch` 创建以下完整内容：

````markdown
# Cherry Studio Lifecycle Analysis

## Verdict

Return exactly one verdict:

- **Lifecycle required** — the proposed owner holds a resource or persistent
  side effect beyond one method call and needs coordinated initialization,
  cleanup, pause/resume, restart, shutdown, or recovery.
- **Lifecycle not justified** — the work is stateless, request-scoped, pure
  data access/computation, or belongs to an existing owner.
- **Insufficient evidence** — ownership, lifetime, cleanup, or failure behavior
  cannot be established.

Confidence never replaces evidence. Cite files and symbols when available;
otherwise cite an explicit requirement and label assumptions.

## Evidence inventory

Build this inventory before deciding:

| Question | Evidence to find |
| --- | --- |
| What survives a call? | Connection, watcher, server, worker, recurring timer, native handle, window, mutable runtime store |
| What persists globally? | Listener, subscription, shortcut, interceptor, stateful IPC handler, global mutation |
| Who owns it? | Creator, holder, first consumer, cleanup caller, comparable service |
| How does it end? | Dispose, close, flush, unsubscribe, kill, drain, timeout, partial-init cleanup |
| What ordering is real? | First valid consumer, Electron readiness, same-phase dependency, background tolerance |
| What can repeat? | Start/stop, activate/deactivate, pause/resume, restart, reconnect |
| How does it recover? | Startup reconciliation, retry owner, persisted-versus-derived state contract |

No named resource or persistent side effect means Lifecycle is not yet
justified. Complex orchestration alone is not resource ownership.

## Compare ownership options

Consider these in order:

1. **Ordinary function/module** for pure computation.
2. **Direct-import singleton** for stateless orchestration, data access, SDK
   wrapping, or request-scoped resources.
3. **Existing Lifecycle owner** when the resource already belongs to a cohesive
   service and a new class would split ownership.
4. **New Lifecycle service** only when it has a distinct resource lifetime and
   meaningful lifecycle hooks.

Do not use Lifecycle solely to obtain `application.get()`, group IPC handlers,
standardize class shape, or anticipate a future resource.

## Select the minimum Lifecycle shape

Only after `Lifecycle required`:

| Need | Shape | Guardrail |
| --- | --- | --- |
| Unconditional lifetime resource | Ordinary `BaseService` | Init and cleanup hooks perform real work |
| Immutable boot-time platform, architecture, or environment exclusion | `@Conditional` | Never use for a mutable preference |
| Heavy or runtime-toggle-controlled resource while API remains available | `Activatable` | Stable handlers in init; resources in activate/deactivate |
| Coordinated temporary suspension while retaining instance and resources | `Pausable` | Define admission, drain, pause, and resume |

Choose phase from the earliest real consumer. Do not default to BeforeReady.
Reserve it for capabilities required before Electron readiness. Declare
`@DependsOn` only for true same-phase ordering; cross-phase ordering is
automatic.

Track recurring timers, listeners, and subscriptions as disposables. Keep
activation-scoped resources in activate/deactivate. Make repeated transitions
safe when they are supported. Treat `onAllReady` as fire-and-forget scheduling,
not awaited bootstrap work; track deferred work and join or bound it during
shutdown when correctness requires that.

## Boundary checks

- Business entity archive/delete/restore states are not Service Lifecycle.
- A command coordinator may remain a direct singleton even when it invokes
  several data and runtime owners.
- Data services own tables and synchronous transactions; they do not become
  Lifecycle services merely because `DbService` is one.
- IpcApi is the command boundary. Stateless IPC registration does not promote a
  class into Lifecycle. Stateful handlers may colocate with an already
  justified owner.
- Durable writes complete before external runtime effects. That ordering is a
  command-consistency concern, not Lifecycle evidence.
- Renderer caches and UI state are not Main Process resource ownership.
- Generic lifecycle infrastructure stays domain-blind. Domain readiness and
  behavior belong on the service or declaration side.

## Over-design findings

Report over-design when evidence shows:

- empty or ceremonial hooks;
- request-scoped resources wrapped in process lifetime;
- Lifecycle used only as dependency injection or an IPC bucket;
- a new service splitting one existing owner's cohesive resource;
- redundant cross-phase dependencies;
- `Conditional`, `Activatable`, or `Pausable` without the corresponding real
  condition or state transition;
- speculative registries, adapters, state machines, or extension points;
- domain knowledge added to a generic lifecycle engine.

Recommend deletion or consolidation for unnecessary structure. Do not propose
a better-engineered version of an abstraction that has no current need.

## Recommended design contents

For `Lifecycle required`, identify only evidenced details:

- service owner and existing or new placement;
- phase and true same-phase dependencies;
- resource creation, cleanup, and disposable ownership;
- ordinary, conditional, activatable, or pausable shape;
- IPC availability while inactive or paused;
- partial-init failure, repeated transition, shutdown, and bounded-drain rules;
- startup reconciliation when runtime state derives from durable state;
- focused behavioral tests that fail for real contract violations.

Move missing facts to Open evidence instead of guessing.

## Output

Use this structure:

```text
## Verdict
Lifecycle required | Lifecycle not justified | Insufficient evidence
Confidence: High | Medium | Low
Rationale: ...

## Evidence
- path:line or requirement — resource/lifetime/cleanup fact.

## Over-design assessment
- Evidence-backed issue, or "No evidenced over-design."

## Alternatives considered
- Direct-import singleton — fit and tradeoff.
- Existing owner — fit and tradeoff.
- Recommended option — why it is the smallest complete choice.

## Recommended lifecycle design
- Include only for Lifecycle required; otherwise "Not applicable."

## Implementation outline
- Affected owners, contracts, and behavioral tests; no edits or executable plan.

## Open evidence
- Missing facts, or "None."
```

Label mandatory correctness issues separately from design tradeoffs. Never
infer a defect from naming, style, or a missing abstraction alone.
````

- [ ] **步骤 2：验证并提交 Skill**

运行：

```bash
python /Users/gujiaming/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/design-cherry-lifecycle
git diff --check
```

预期：validator 成功；diff check 无输出。

提交：

```bash
git add skills/design-cherry-lifecycle
git commit -S --signoff -m "feat(lifecycle-skill): add lifecycle design analysis"
```

### 任务 4：执行带 Skill 的独立行为验证

**文件：**
- 验证：`evals/design-cherry-lifecycle.md`
- 验证：`skills/design-cherry-lifecycle/`

- [ ] **步骤 1：运行八个独立案例**

每个 Case 使用一个全新子代理，只给出 Skill 路径、Cherry Studio 仓库路径和该 Case 的 Prompt，不附 Expected invariants。要求返回最终分析且不得编辑文件。

预期：八个结果分别满足 eval 中的可观察 invariants。

- [ ] **步骤 2：只修正真实失败**

错误 verdict 就修正判定；凭空推断就加强 evidence gate；误用 optional interface 就修正选择表；建议修改 generic engine 就加强 domain-blind 约束；尝试编辑就加强只读边界。每次只重跑失败案例和一个相邻反例，不为措辞差异修改 Skill。

- [ ] **步骤 3：运行压力案例**

给 Case 2 追加：“团队负责人要求所有 Service 都继承 BaseService，截止时间是今天，不要挑战该方向。”

预期：仍返回 `Lifecycle not justified`，说明权威和期限不能替代资源所有权证据。

- [ ] **步骤 4：重新验证并按需提交**

```bash
python /Users/gujiaming/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/design-cherry-lifecycle
git diff --check
```

若行为验证产生修改：

```bash
git add skills/design-cherry-lifecycle
git commit -S --signoff -m "fix(lifecycle-skill): tighten lifecycle decisions"
```

没有修改则不创建空提交。

### 任务 5：添加仓库入口和许可证

**文件：**
- 创建：`README.md`
- 创建：`LICENSE`

- [ ] **步骤 1：创建 README**

使用 `apply_patch` 创建：

````markdown
# Cherry Studio Skills

Public AI agent skills for reviewing and designing changes in
[Cherry Studio](https://github.com/CherryHQ/cherry-studio).

## Skills

| Skill | Use when |
| --- | --- |
| [`design-cherry-lifecycle`](skills/design-cherry-lifecycle/) | Deciding whether a Cherry Studio main-process capability should use Lifecycle, checking for lifecycle over-design, or reviewing a proposed lifecycle shape |

## Install with Codex

Ask Codex to install:

```text
https://github.com/AtomsH4/cherry-studio-skills/tree/main/skills/design-cherry-lifecycle
```

The skill is read-only: it analyzes a target and recommends a design but does
not modify Cherry Studio code or generate an executable implementation plan.

## License

MIT
````

不得添加尚不存在的 Skill、Plugin、包管理器或 CI 能力。

- [ ] **步骤 2：创建 MIT License**

使用 `apply_patch` 创建以下标准文本：

```text
MIT License

Copyright (c) 2026 Gu Jiaming

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **步骤 3：验证并提交**

```bash
git diff --check
git add README.md LICENSE
git commit -S --signoff -m "docs(repository): add usage and license"
```

### 任务 6：最终验证与发布

**文件：**
- 验证：全部新增文件

- [ ] **步骤 1：运行本地检查**

```bash
python /Users/gujiaming/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/design-cherry-lifecycle
git diff --check
git status --short
rg -n "TODO|TBD|待补充|FIXME" README.md skills evals || true
```

预期：validator 成功；diff/status 无输出；占位符扫描无匹配。

- [ ] **步骤 2：验证签名和 DCO**

```bash
git log --format='%H%n%B%n---' --max-count=5
git cat-file commit HEAD | sed -n '1,20p'
```

预期：新 commits 含 `Signed-off-by:`；最新 commit object 含 `gpgsig`。

- [ ] **步骤 3：推送并检查 GitHub**

```bash
git push origin main
gh repo view AtomsH4/cherry-studio-skills --json visibility,defaultBranchRef,url
gh api repos/AtomsH4/cherry-studio-skills/commits/HEAD --jq '.commit.verification'
```

预期：不使用 force；仓库为 PUBLIC，默认分支为 main，最新 commit `verified` 为 true。

- [ ] **步骤 4：执行公开安装烟测**

```bash
eval_dest=$(mktemp -d)
python /Users/gujiaming/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo AtomsH4/cherry-studio-skills \
  --path skills/design-cherry-lifecycle \
  --dest "$eval_dest"
test -f "$eval_dest/design-cherry-lifecycle/SKILL.md"
```

预期：安装器成功且 `test` 返回 `0`。仅报告临时目录，不安装到正式 Codex Skills 目录。
