# 简体中文

[English](#dsh-verify) · **[简体中文](#简体中文)**

[![Node](https://img.shields.io/badge/node-%3E%3D18-339933?logo=node.js&logoColor=white)](#quick-start)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)](#how-it-works)
[![Tests](https://img.shields.io/badge/tests-44%20passing-brightgreen)](#testing)
[![DSH](https://img.shields.io/badge/DSH-0.2.0--rc.2-4B32C3)](#compatibility-author-and-licence)

面向 **DeepSeek Harness（DSH）** 插件与 agent preset 模式的**外部只读完整性校验器**。它只回答一个问题：

> **DSH 更新之后，我装的插件和模式里，哪些静默失效了？**

零依赖、无需安装，完全在 DSH 进程之外运行——不加载、不导入、不修改 DSH。

```bash
node bin/dsh-verify.mjs            # 静态扫描，profile 自动探测
node bin/dsh-verify.mjs --probe    # 额外做隔离挂载冒烟
```

Windows 用户可直接双击 `run.cmd` 打开菜单，或 `run.cmd doctor` 做环境自检。

---

## 为什么需要它

DSH 用多层 `cordis.patch.yml` 组合出一个 profile。以下三种情况**不会报任何错**：

1. **某一层没加载。** 解析失败或补丁文件缺失，它声明的行就整段消失；其余配置照样组合、照样通过。
2. **插件不再注册。** 配置里引用能力的值（`searchProvider: ddg`）只是字符串；发布它的插件静默停止加载后，
   字符串照样解析，功能悄悄回退到别处。
3. **preset 被遮蔽。** DSH 只保留先找到的同名 preset id、静默丢弃后者，于是目录存在、文件合法，模式却永远不出现。

本工具同时检查这三类问题及其背后的产出物契约；判不出来的情况一律报 `UNKNOWN`，而不是猜。

## 快速开始

**要求**：Node.js ≥ 18，或使用 DSH 自带的运行时（`PATH` 里没有 `node` 时自动使用）。无需 `npm install`、
无需全局安装、无需注册成 DSH 插件。

```bash
git clone <本仓库> && cd dsh-verify
node bin/dsh-verify.mjs                 # 校验自动探测到的 profile
node bin/dsh-verify.mjs --profile web   # 显式指定 profile
node bin/dsh-verify.mjs doctor          # 只体检，不校验
```

启动器（`run.cmd` / `run.ps1`）会从自身位置推导工具根目录，所以放在哪里都算"已安装"。
`node bin/dsh-verify-launcher.mjs install-path --dir <dir>` 可把 `dsh-verify` 垫片装到已有的 `PATH` 目录。

## 检查什么

每个插件逐层推进，失败即终止，不回头重试。

| 层级 | 检查 | 终止结果 |
|---|---|---|
| **L0** | 是否在组合后的配置树中 | 行缺失 → `FAILED`；`disabled: true` → `PASS (mounted)` |
| **L1** | 可加载：manifest 可解析、入口存在；可选隔离挂载 | 不可解析或导入抛错 → `FAILED` |
| **L2** | 枚举产出物（7 类：工具、命令、钩子、服务、prompt 段、UI 槽位、纯副作用） | 无产出物 → `PASS (mounted)`，**终止** |
| **L3** | 产出物契约、重名、能力绑定 | 违反 → `DEGRADED`；无法判定 → `UNKNOWN` |
| **L4** | 最小行为触发 | 无法构造 → `UNKNOWN`（deferred） |

状态：`PASS`、`PASS (mounted)`、`DEGRADED`、`FAILED`、`UNKNOWN`。模式（agent preset）有独立阶梯，
包含遮蔽检测，以及与默认模式的工具/prompt 会话差异比对。

## 使用

```text
dsh-verify [verify] [options]        校验插件与模式（默认）
dsh-verify quick [options]           用自动探测到的 profile 校验
dsh-verify doctor                    只检查环境，不做校验
dsh-verify menu                      交互式选择
dsh-verify self-test                 检查启动器与 CLI 是否正常
dsh-verify baseline save [options]   保存当前状态为基线
dsh-verify baseline diff [options]   与基线比较
dsh-verify prompt                    打印可交给 Agent 会话的独立校验提示词
```

| 参数 | 说明 |
|---|---|
| `--profile <name>` | 要组合的 profile（默认从 `<DSH_HOME>/profiles` 自动探测） |
| `--all` | 连平台自身的 `@deepseek-ai/*` 行一起校验 |
| `--only <a,b>` | 只校验指定行 id 或包名 |
| `--modes <a,b>` / `--modes-only` / `--plugins-only` | 限定范围 |
| `--probe` | 在隔离进程中导入每个插件（挂载冒烟） |
| `--budget-calls <n>` / `--budget-seconds <s>` | 上限，默认 20 次调用 / 180 秒 |
| `--format text\|markdown\|json\|junit` | 报告格式（默认 `text`） |
| `--output <file>` | 报告写入文件而非 stdout |
| `--dsh-home <path>` / `--install <path>` / `--dsh-bin <path>` | 显式指定环境路径 |
| `--patch <file>` | 追加补丁层（可重复） |
| `--baseline <file>` / `--save-baseline <file>` | 与基线比较 / 写入基线 |
| `--version-probe` | 消耗一次预算调用执行 `dsh --version` |
| `--quiet` | 不在 stderr 输出人类可读摘要 |

示例输出（示意）：

```text
dsh-verify — C:\Users\<you>\.dsh · DSH 0.2.0-rc.2 · static

  DEGRADED  L3    web :: fetchProvider   "acme-fetch" matches no capability id published by any plugin
  UNKNOWN   L3    acme-tools             exampleService (inject): no composed row resolves to it

  NOTE  plugins: every composed row is a platform package, so nothing outside the platform was verified

VERDICT: 1 DEGRADED | 1 UNKNOWN (1 row(s) deferred — rerun with --probe)
BUDGET: used 0 probe calls / 2 inspections / 1.4s of 20 calls / 180s
```

## 报告与退出码

| 退出码 | 含义 |
|---|---|
| `0` | 无发现（deferred 的 `UNKNOWN` 不影响退出码） |
| `1` | 存在 `FAILED` 或 `DEGRADED` |
| `2` | `BLOCKED`——环境守卫未通过，什么都没校验 |

`deferred` 的 `UNKNOWN` 表示"换个参数就能定论"（例如加 `--probe`），因此只报告、不计数，也不会淹没真正的发现。
若一次运行什么都没校验，报告会用 `NOTE` 明说，而不是只打印一句 `ALL PASS`。

## 工作原理

- **构造上独立。** `dsh --profile <name> --dump-config` 并非只读：DSH 打印前会重写 profile 的 `cordis.yml`。
  本工具从不调用它，而是直接读配置文件、在内存里做同样的层折叠；唯一的动态调用是 `--version-probe` 下的 `dsh --version`。
- **零依赖**，只用 Node.js 标准库（YAML 读取器也是自研）。
- **只读。** 只在你指定的路径写两类文件：报告（`--output`）与基线快照；绝不修改 DSH 配置。
- **防空转。** 每项只尝试一次、双账本预算（探针调用与墙钟），不做根因分析；查不到就是 `UNKNOWN`，不是 `FAILED`。
- **单一数据结构。** 每个结果都是 `item | type | status | layer | evidence` 一行，四种报告格式渲染同一批行，不会互相矛盾。

## 测试

```bash
node tests/sanity.mjs          # 模块可导入、导出可解析、编码干净、版本一致
node tests/verify.test.mjs     # 44 项行为测试，跑在迷你 harness home 上
node tests/yaml-diff.mjs       # 自研 YAML 读取器 vs js-yaml，本机全部 YAML 文件 307/307 一致
```

`tests/yaml-diff.mjs` 是最关键的验证：`js-yaml` 仅作**开发期 oracle**（自动发现，永不是运行时依赖）。
行为测试会生成 `tests/fixtures/env`，其中每个 fixture 只以一种方式损坏，测试断言它恰好产生方案要求的判定。

## 已知边界

诚实标注，不做能力外承诺：

- **L4 无法真正触发。** 外部只读进程无法构造最小工具调用，L4 恒为 `UNKNOWN`。
- **Cordis 服务键无法完全静态解析**，判不出来就是 `UNKNOWN`。
- **`--probe` 是冒烟测试，不是挂载。** 因缺服务而失败记为"未观测"（`UNKNOWN`）；缺的是 DSH 自己提供的同级依赖时同样如此。
- **配置来源的 provider id 判不出来，且刻意不猜。** 这类引用可能报 `DEGRADED`（已知误报）；猜错比误报更糟。
- **0.2.0 的模式缺口。** 0.2.0 起 preset 是配置层里的 `dsh-agent-preset` 行，而非 `.agent-presets/<id>/` 目录；
  本版本只扫目录，因此会用 `NOTE` 明说这一点。

## 兼容性、作者与许可

- **上游**：以 **DSH（官方版）`0.2.0-rc.2`** 为基准构建，全部行为与兼容性结论基于该版本实测；
  **后续版本兼容性尚未测试**，也不作保证，DSH 升级后请重新校验。
- **作者**：**FlyDragon39**。本项目是在 AI 协助下完成的个人项目，也是作者首次尝试代码构建与提交，
  难免有不足，欢迎提出 issue、指正与建议。
- **许可**：MIT，完全开源；不限制商用与修改，可自由使用，唯一要求是标注原作者与项目名称。详见 [LICENSE](LICENSE)。

# dsh-verify

**[English](#dsh-verify)** · [简体中文](#简体中文)

An external, read-only completeness verifier for **DeepSeek Harness (DSH)** plugins and agent-preset modes.
It answers one question:

> **After a DSH update, which of my plugins and modes silently stopped working?**

Zero dependencies, no install step, and it runs entirely outside the DSH process — it never loads,
imports or modifies DSH.

```bash
node bin/dsh-verify.mjs            # static sweep, profile auto-detected
node bin/dsh-verify.mjs --probe    # + isolated mount smoke test
```

Windows users can double-click `run.cmd` for a menu, or `run.cmd doctor` for an environment check.

---

## Why

DSH composes a profile from layered `cordis.patch.yml` files. Three things routinely fail *without an error*:

1. **A layer does not load.** A parse error or a missing patch file means every row it declared is
   simply absent — the rest of the tree still composes and still passes.
2. **A plugin stops registering.** Config values that name capabilities (`searchProvider: ddg`) are
   plain strings; if the plugin that publishes them stops loading, the string still parses and the
   feature quietly falls back to something else.
3. **A preset is shadowed.** DSH keeps the first preset id it finds and silently drops later ones, so a
   valid mode directory can exist and never appear.

`dsh-verify` checks all three, plus the artifact contracts behind them, and reports what it *cannot*
know as `UNKNOWN` instead of guessing.

## Quick start

**Requirements** — Node.js ≥ 18, or the runtime DSH already ships (used automatically when `node` is
not on `PATH`). Nothing else: no `npm install`, no global install, no DSH plugin registration.

```bash
git clone <this repo> && cd dsh-verify
node bin/dsh-verify.mjs                 # verify the auto-detected profile
node bin/dsh-verify.mjs --profile web   # name a profile explicitly
node bin/dsh-verify.mjs doctor          # environment only, verifies nothing
```

The launcher (`run.cmd` / `run.ps1`) finds its own tool root, so the checkout is already "installed"
wherever it sits. `node bin/dsh-verify-launcher.mjs install-path --dir <dir>` optionally adds a
`dsh-verify` shim to a directory you already have on `PATH`.

## What it checks

Each plugin advances a ladder; a failing layer ends the item, and nothing re-enters an earlier layer.

| Layer | Check | Ends the item with |
|---|---|---|
| **L0** | present in the composed tree | row missing → `FAILED`; `disabled: true` → `PASS (mounted)` |
| **L1** | loads: manifest resolves, entry exists; optional isolated mount | unresolvable or import throws → `FAILED` |
| **L2** | enumerates artifacts (7 classes: tool, command, hook, service, prompt section, UI slot, side effect) | none → `PASS (mounted)`, **stop** |
| **L3** | artifact contracts, duplicate names, capability bindings | violated → `DEGRADED`; undecidable → `UNKNOWN` |
| **L4** | minimal behaviour trigger | unconstructible → `UNKNOWN` (deferred) |

Statuses: `PASS`, `PASS (mounted)`, `DEGRADED`, `FAILED`, `UNKNOWN`. Modes (agent presets) have their
own ladder, including shadow detection and a tool/prompt-session diff against the default mode.

## Usage

```text
dsh-verify [verify] [options]        verify plugins and modes (default)
dsh-verify quick [options]           verify with an auto-detected profile
dsh-verify doctor                    show environment readiness; verify nothing
dsh-verify menu                      interactive picker
dsh-verify self-test                 check the launcher and CLI are wired up
dsh-verify baseline save [options]   snapshot the current state as a baseline
dsh-verify baseline diff [options]   compare the current state against a baseline
dsh-verify prompt                    print the standalone verification prompt
```

| Option | Meaning |
|---|---|
| `--profile <name>` | profile to compose (otherwise auto-detected from `<DSH_HOME>/profiles`) |
| `--all` | also verify the platform's own `@deepseek-ai/*` rows |
| `--only <a,b>` | restrict to these row ids or package names |
| `--modes <a,b>` / `--modes-only` / `--plugins-only` | scope the run |
| `--probe` | import each plugin in an isolated process (mount smoke test) |
| `--budget-calls <n>` / `--budget-seconds <s>` | ceilings, default 20 calls / 180 s |
| `--format text\|markdown\|json\|junit` | report format (default `text`) |
| `--output <file>` | write the report to a file instead of stdout |
| `--dsh-home <path>` / `--install <path>` / `--dsh-bin <path>` | explicit paths for the environment |
| `--patch <file>` | extra patch overlay (repeatable) |
| `--baseline <file>` / `--save-baseline <file>` | diff against / write a baseline |
| `--version-probe` | spend one budget call on `dsh --version` |
| `--quiet` | suppress the human summary on stderr |

Sample output (illustrative):

```text
dsh-verify — C:\Users\<you>\.dsh · DSH 0.2.0-rc.2 · static

  DEGRADED  L3    web :: fetchProvider   "acme-fetch" matches no capability id published by any plugin
  UNKNOWN   L3    acme-tools             exampleService (inject): no composed row resolves to it

  NOTE  plugins: every composed row is a platform package, so nothing outside the platform was verified

VERDICT: 1 DEGRADED | 1 UNKNOWN (1 row(s) deferred — rerun with --probe)
BUDGET: used 0 probe calls / 2 inspections / 1.4s of 20 calls / 180s
```

## Reports and exit codes

| Exit | Meaning |
|---|---|
| `0` | no findings (deferred `UNKNOWN` rows do not affect it) |
| `1` | at least one `FAILED` or `DEGRADED` |
| `2` | `BLOCKED` — the environment guard failed, nothing was inspected |

A `deferred` `UNKNOWN` means "a flag would settle this" (for example `--probe`), so it is reported but
never counted, and it cannot drown out real findings. A run that verified nothing says so with a
`NOTE` rather than printing a bare `ALL PASS`.

## CI

```yaml
- run: node bin/dsh-verify.mjs --probe --format junit --output results.xml --quiet
- run: node bin/dsh-verify.mjs --baseline .dsh-verify/baseline.json --format json --output report.json
```

## How it works

- **Independent by construction.** `dsh --profile <name> --dump-config` is *not* read-only: DSH rewrites
  the profile's `cordis.yml` before printing. So this tool never calls it — it reads the configuration
  files and performs the same layer fold in memory. The only dynamic call it makes is `dsh --version`,
  and only with `--version-probe`.
- **Zero dependencies**, Node.js standard library only, including the YAML reader.
- **Read-only.** It writes exactly two kinds of file, both at paths you name: reports (`--output`) and
  baseline snapshots. It never modifies DSH configuration.
- **Anti-spin.** One attempt per item, a two-ledger budget (probe calls and wall clock), and no
  root-cause analysis: unreachable checks become `UNKNOWN`, not `FAILED`.
- **One data structure.** Every result is one row of `item | type | status | layer | evidence`, and all
  four report formats are renderings of the same rows, so they cannot disagree.

## Testing

```bash
node tests/sanity.mjs          # modules import, exports resolve, encoding clean, version agrees
node tests/verify.test.mjs     # 44 behavioural tests against a miniature harness home
node tests/yaml-diff.mjs       # YAML reader vs js-yaml over every YAML file on this machine (307/307)
```

`tests/yaml-diff.mjs` is the most important validation: the built-in reader is compared value-for-value
against `js-yaml`, which is a **dev-only oracle** discovered automatically (it is never a runtime
dependency). The behavioural suite builds `tests/fixtures/env`, where every fixture is broken in exactly
one way and the test asserts it produces exactly the verdict the plan requires.

## Limitations

Stated plainly, because claiming more would be dishonest:

- **L4 cannot truly trigger.** An external read-only process cannot construct a minimal tool invocation;
  L4 is always `UNKNOWN`.
- **Cordis service keys are not fully resolvable statically.** Undecidable means `UNKNOWN`.
- **`--probe` is a smoke test, not a mount.** A plugin that fails for a missing service is "not
  observed" (`UNKNOWN`), and so is a missing peer that DSH itself supplies.
- **A config-derived provider id is unresolvable and deliberately not guessed.** Such a reference can be
  reported `DEGRADED` — a known false positive; guessing would be strictly worse.
- **Modes on DSH 0.2.0.** 0.2.0 declares presets as `dsh-agent-preset` rows rather than
  `.agent-presets/<id>/` directories; this build scans directories only and says so with a `NOTE`.

## Compatibility, author and licence

- **Upstream**: built against **DSH (official edition) `0.2.0-rc.2`**; every behaviour and compatibility
  claim was tested against that version. Compatibility with later DSH versions is **not tested and not
  guaranteed** — re-validate after a DSH update.
- **Author**: **FlyDragon39**. This is a personal project completed with AI assistance and the author's
  first attempt at building and submitting code; issues, corrections and suggestions are very welcome.
- **Licence**: MIT — fully open source. Commercial use and modification are not restricted; the only
  requirement is that the original author and this project are credited. See [LICENSE](LICENSE).

---


