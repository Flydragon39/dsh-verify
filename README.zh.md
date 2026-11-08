# dsh-verify · DSH 插件与模式完整性校验器

`《DSH 插件与模式完整性校验方案（最终版）》` 的独立外部实现。零依赖（仅 Node.js 标准库）、只读、进程外运行。

它回答一个问题：**DSH 更新之后，我装的插件和模式里，哪些静默失效了？**

> **想直接开跑？** 看 [QUICKSTART.zh.md](QUICKSTART.zh.md)：双击 `run.cmd` 就有菜单，
> 不需要装任何东西，`profile` 自动探测，没有 Node 时自动用 DSH 自带的运行时。

```bash
node bin/dsh-verify.mjs
```

```
dsh-verify — C:\Users\<you>\.dsh · DSH 0.2.0-rc.2 · static

  UNKNOWN   L3    acme-tools   5 artifact(s): 1 UNKNOWN, 4 PASS — exampleService (inject): no composed row resolves to the service "exampleService"; …
  DEGRADED  L3    example-plugin :: defaultProvider   "example-voice" matches no capability id …

  NOTE  modes: no agent preset was discovered in any root, so mode verification examined nothing …

VERDICT: 1 DEGRADED | 1 UNKNOWN (2 row(s) deferred — rerun with --probe)
BUDGET: used 0 probe calls / 1 inspections / 1.44s of 20 calls / 180s
```

`--profile` 省略时会从 `<DSH_HOME>/profiles` 自动探测（有 `web` 用 `web`，否则用实际存在的那个）。

---

## 项目说明

- **作者**：FlyDragon39
- **上游**：本项目以上游 **DSH（官方版）`0.2.0-rc.2`** 为基准构建，文中全部行为与兼容性
  结论都基于该版本实测。**后续版本的兼容性尚未测试**，也不作保证；DSH 升级可能改变配置层、
  preset 发现方式或产出物形态，届时需要重新校验本工具。
- **关于本项目**：这是作者在 AI 协助下完成的个人项目，也是作者首次尝试代码构建与提交，
  难免有不足之处，欢迎提出 issue、指正与建议。
- **开源声明**：本项目采用 MIT 协议、完全开源，不限制商用与修改，可自由使用；唯一要求是
  标注原作者与项目名称。

---

## 一、为什么必须独立于 DSH

方案的第一条硬约束是**独立性**：校验器不能作为 DSH 插件运行，否则自身会被同一次更新损坏。

实现过程中发现的事实让这条约束比方案写得更硬：

**`dsh --profile <name> --dump-config` 并不是只读的。** DSH 在打印配置树之前会先 `writeFileSync` 重写 profile 的 `cordis.yml`（`profile-boot-*.js:209`）。`dsh-fix` 式的外部工具可以用它，但一个"只读校验器"不能——在受限沙箱下它甚至直接 `EPERM` 失败：

```
Error: EPERM: operation not permitted, open '...\.dsh\profiles\web\cordis.yml'
    at prepareProfile (dsh/lib/profile-boot-Dk-7KqJc.js:209:2)
    at runDumpConfig (dsh/lib/dump-config-lFgMwK8i.js:25:17)
```

因此本工具**不调用 `--dump-config`**。它按 DSH 自己的分层算法直接读取配置文件，在内存里做同样的 fold。唯一的动态调用是 `dsh --version`，而且只在显式传 `--version-probe` 时才发生。

`dsh-verify` 只写两种文件，且都必须由使用者显式指定路径：报告文件（`--output`）和基线快照（`baseline save`）。**它永不修改 DSH 配置。**

---

## 二、核心机制：产出物契约 + 能力绑定

### 2.1 七类产出物（方案 §3）

插件实现千奇百怪，但运行时注册的东西收敛为有限几类。本工具静态识别这七类，每条产出物都带 `文件:行号` 证据：

| 产出物 | 识别方式 | 判定 |
|---|---|---|
| 工具 | `defineTool({ name })` + `tools.register()` | 名称唯一 + 有描述 + 有参数 schema |
| 命令 | `commands.register()` / `ctx.command()` | 名称是可用的 slash token |
| 钩子 | `ctx.on('event')` | 事件名带命名空间 |
| 服务 | `ctx.inject([...])` / `provide()` / `register*Provider()` | 有提供方、可被消费 |
| 提示词段 | `systemPrompt.section({ name })` | 有名称 + 有正文 |
| UI 插槽 | `package.json` 的 `dsh.client` | 声明的 client 入口文件存在 |
| **纯副作用** | 以上全无 | **挂载成功即达标，立即终止** |

> 模块自身的 `export const name` 是身份信息，不算产出物。否则"什么都没注册"的插件永远到不了 L2 终点。

### 2.2 能力绑定检查——这是最值钱的一项

方案要抓的头号失效模式是"功能消失但不报错"。它最典型的样子是：

```yaml
# profiles/web/cordis.patch.yml
- id: web
  config:
    searchProvider: ddg        # 一个字符串
    fetchProvider: safe-http   # 一个字符串
```

这两个值指向的是**插件在挂载时才注册的 provider id**。如果 `acme-search` 停止加载，这个字符串照样解析、配置树照样组合、**没有任何东西会报错**——搜索只是静默回退到别的引擎。

只有把"配置里的值"和"插件实际注册的 capability id"交叉验证，才能看见它。本工具做到了：

```
| web :: searchProvider | binding | PASS     | L3 | "ddg" is published by acme-search |
| web :: fetchProvider  | binding | PASS     | L3 | "safe-http" is published by acme-fetch |
```

实现要点：`register*Provider(...)` 的参数常常是**跨文件的类实例**（`new ExampleFetchProvider(...)` 在 `index.js`，`id = 'safe-http'` 在 `provider.js`）。工具会做一次跨文件解析；**解析不出来时返回 `null` 而不是猜**——猜错会把真正悬空的引用变成假 PASS。

---

## 三、阶梯引擎（方案 §4）

每个插件按 L0→L4 推进，**任一层失败或终止即停，不进入下一层**：

| 层 | 检查 | 通过条件 | 终止条件 |
|---|---|---|---|
| **L0** | 配置树中存在 | 行被组合出来 | 缺失 → `FAILED`；`disabled: true` → `PASS(mounted)` |
| **L1** | 可加载 | manifest 可解析、入口存在 | 不可解析 → `FAILED`；`--probe` 时导入抛错 → `FAILED` |
| **L2** | 枚举产出物 | ≥1 类产出物 | 无产出物 → `PASS(mounted)`，**终止** |
| **L3** | 产出物契约 | 契约满足 | 违反 → `DEGRADED`；无法判定 → `UNKNOWN` |
| **L4** | 最小行为触发 | 返回正常结果 | 无法构造 → `UNKNOWN`（`deferred`） |

**L2 是防空转第一道闸门**：枚举不到任何产出物就终止，不进 L3/L4。

### 组合层读不进来，也要出一行，而不是脚注

组合器会读 profile 的 bundle 层、profile 自己的 `cordis.patch.yml`、home 层，以及任何 `--patch` 覆盖文件。其中任一层读不进来——解析失败、声明的补丁文件缺失、bundle 包未安装——它本该贡献的那些行就**整段消失**，而它下面的所有行仍能正常组合并通过。

这正是方案点名的静默失败，因此每个这样的问题都变成一条普通报告行（`type: composition`、`FAILED`），而不是只写在 Markdown 报告里的一节。这样一来，它天然会进入裁决、退出码、文本报告、JSON 与 JUnit，任何格式都无法对"这次运行是否完整"给出互相矛盾的说法。

### 模式校验（独立阶梯）

DSH 的"模式"是 **agent preset**：`<DSH_HOME>/.agent-presets/<id>/` 下的 `agent.cordis.yml`（组合）+ `preset.yml`（显示元数据）。方案要求"切换模式→确认没被弹回→diff 工具列表和 system prompt→无变化即 FAILED"。外部进程无法切换运行中会话的 preset，因此这四个动态步骤被映射到同源的静态等价物：

| 方案要求 | 外部等价实现 |
|---|---|
| 切换到目标模式 | 按 roster 的方式组合该 preset 的行 |
| 确认未被弹回默认 | **检测 id 被更高优先级 root 遮蔽**——这正是"弹回"的真实成因，也是 DSH 会**静默丢弃**的情形 |
| diff 工具列表与 system prompt | 解析每行对应的包，取工具/提示词段集合，与默认模式求差 |
| 无变化 → FAILED | 行集合与产出面完全一致 → `FAILED` |

健康检查逐步复刻 DSH 自己的 `discovery` 顺序与措辞（文件存在 → 可读 → YAML 可解析 → 行结构合法 → 模块可解析）。`preset.yml` 损坏在 DSH 里**从不致命**，因此这里也只记 `DEGRADED`。

> **遮蔽检测**是这里最关键的发现。DSH 的 `discoverPresets` 对跨 root 的重名 id 执行 `if (byId.has(preset.id)) continue;` —— 第一个 root 胜出，后来者**静默消失**。用户的 preset 目录在、文件合法、模式却永远不出现，DSH 不会说一个字。本工具把它报成 L0 `FAILED`。

---

## 四、防空转（方案 §5）

四条硬规则全部落地：

1. **全局预算封顶。** 20 次 probe 调用 / 180 秒，任一超限立即停止，未决项标 `UNKNOWN`。
2. **每层单次尝试。** 没有重试、没有换参数再试。
3. **`UNKNOWN` 是一等状态。** 判不了就标 UNKNOWN 并跳过，不进入探索循环。
4. **不追根因。** 只报现象和层级。

### 预算为什么分两本账

方案说的 20 次是**工具调用**。一个 web profile 组合出 154 行——如果静态读文件也计入，20 次会在第 10 个插件前耗尽，预算就从"防空转"退化成"没用的常数"。

所以：

- `spend()` = **probe**：真正执行东西（子进程、`dsh` CLI、插件代码）。硬顶 20。
- `inspect()` = **静态读**：计入挂钟与报告，不占 probe 配额。

挂钟 180 秒对所有操作无条件生效。两本账都在报告里明写：
```
BUDGET: used 12 probe calls / 2 inspections / 2.39s of 20 calls / 180s
```

---

## 五、`deferred`：诚实的 UNKNOWN

静态运行看不见挂载错误。每一个插件都会得到一条 L1 `UNKNOWN`。这是**诚实**的，但把它当"发现"会让退出码永远是 1，真实发现被淹没。

因此 UNKNOWN 行带一个 `deferred` 标记：**状态仍是 UNKNOWN**（方案明确 UNKNOWN 是一等状态），但单独计数——它不进退出码、不淹没真发现，文本报告里用一行说明：

```
VERDICT: 1 UNKNOWN (3 row(s) deferred — rerun with --probe)
```

`deferred` 的语义是"换个参数就能解决"，不是"需要人来调查"。

---

## 六、用法

### 模式 0：启动器（最省事）

```powershell
run.cmd                  # 双击：菜单
run.cmd doctor           # 只体检：Node / DSH 安装 / profile 列表
run.cmd quick            # 自动探测 profile 的静态校验
run.cmd quick --probe    # 加上隔离进程挂载冒烟
run.cmd self-test        # 验证本工具自身接线完好
```

`run.cmd` / `run.ps1` 会自己找 Node（`DSH_NODE` → PATH → DSH 自带运行时），
所以**没装 Node 也能跑**。详见 [QUICKSTART.zh.md](QUICKSTART.zh.md)。

### 模式 A：CLI（推荐）

```bash
# 日常快速校验（静态层）；profile 自动探测
node bin/dsh-verify.mjs

# 显式指定 profile
node bin/dsh-verify.mjs --profile desktop

# 完整校验（含隔离进程挂载冒烟测试）
node bin/dsh-verify.mjs --probe

# 仅模式校验
node bin/dsh-verify.mjs --modes-only --format json --output modes.json

# 只看自己装的插件（默认即是；--all 才连平台一起扫）
node bin/dsh-verify.mjs --only acme-search,acme-fetch --probe
```

### 模式 B：CI Job

```bash
node bin/dsh-verify.mjs --probe --format junit --output results.xml --quiet
```

退出码：`0` 无发现 / `1` 有 FAILED 或 DEGRADED / `2` BLOCKED。

### 模式 C：独立 Agent 会话

```bash
node bin/dsh-verify.mjs prompt
```

打印一份自包含的校验提示词（方案 §8 的那份，已与真实 DSH 语义对齐），可直接投喂给另一个 DSH 会话或别的 Agent 运行时。

### 基线 Diff（方案 §7）

```bash
# 更新前
node bin/dsh-verify.mjs baseline save --output .dsh-verify/baseline.json --quiet

# 更新后：先 diff，再跑阶梯校验
node bin/dsh-verify.mjs --baseline .dsh-verify/baseline.json
```

diff 覆盖方案点名的五类：配置树插件列表、各 profile 的模式列表、预期工具列表、各模式工具集、DSH 版本号。

```
Baseline diff: 3 change(s)
  dsh-version          harness — 0.0.1-ancient → 0.1.5-rc.1
  plugin-added         web:acme-search — acme-search is newly composed
  mode-tools-gained    example-mode — gained ask_user_question, edit, glob, …
```

### 参数

| 参数 | 说明 |
|---|---|
| `--profile <name>` | 要组合的 profile（默认 `web`） |
| `--modes-only` / `--plugins-only` | 只校验模式 / 跳过模式 |
| `--all` | 校验全部行，含平台自身的 `@deepseek-ai/*` |
| `--only <a,b>` / `--modes <a,b>` | 限定插件 / 模式 |
| `--patch <file>` | 附加 patch 覆盖层（可重复） |
| `--dsh-home` / `--install` / `--dsh-bin` | 显式指定环境 |
| `--version-probe` | 花一次预算真正调用 `dsh --version` |
| `--probe` | 在隔离进程中导入每个插件（挂载冒烟测试） |
| `--budget-calls` / `--budget-seconds` | 预算上限（默认 20 / 180） |
| `--baseline <file>` / `--save-baseline <file>` | 基线 diff / 保存 |
| `--format text\|markdown\|json\|junit` | 报告格式 |
| `--output <file>` / `--quiet` | 输出到文件 / 静默 |

---

## 七、范围：平台是环境守卫，不是被检对象

方案明确："**插件和模式是主角，平台只做环境守卫。**"

所以默认只校验**你装的插件**，平台自身的行只计数不逐行校验：

```
- Composed rows: 191
- Scope: the rows you installed; 188 platform row(s) treated as the environment
  and not verified (pass `--all` to include them)
```

`--all` 可以全扫（191 行），但平台内部包存在大量无法从文件判定的 Cordis 服务键，会产出大量 UNKNOWN——那是噪声，不是发现。

若某个部署组合出来的行**全部**是平台包，默认范围就没有任何可校验对象；这时报告会给出一条 `NOTE`（"nothing outside the platform was verified"），而不是静默地 `ALL PASS`，避免把"什么都没查"误读成"一切正常"。用 `--all` 可转为真实全扫。

---

## 八、退出码与状态语义

| 状态 | 含义 |
|---|---|
| `PASS` | 契约满足 |
| `PASS(mounted)` | 什么都没注册，挂载成功即全部契约（L2 终点） |
| `DEGRADED` | 有确凿问题，但仍然能用 |
| `FAILED` | 用不了 |
| `UNKNOWN` | 文件判不了，需要人或更深的后端 |

---

## 九、自测

```bash
node tests/verify.test.mjs     # 44 项行为测试
node tests/yaml-diff.mjs       # YAML 读取器与 js-yaml 的差分测试
```

`tests/verify.test.mjs` 构建一个完整的小型 harness home（`tests/fixtures/env`），里面的每个 fixture **只以一种方式损坏**，测试断言这种损坏恰好产生方案规定的那个判定。覆盖：

- 健康插件走完 L0–L3、L4 记为 deferred
- 什么都没注册的插件在 L2 以 `PASS(mounted)` 终止
- 工具重名 → 双方都 `DEGRADED`
- 悬空的 provider 引用 → `DEGRADED`；有提供方的 → `PASS`
- 导入即抛错的插件：静态运行 `UNKNOWN`，`--probe` 下 `FAILED`；若缺的是 harness 自己提供的同级依赖，则记为 deferred 的 `UNKNOWN`
- **补丁层被丢弃 → 直接产生 `FAILED` 行**，同时改变裁决、退出码与文本报告
- 缺失组合文件 / YAML 非法 / 行不可解析 / 目录名非法 / **id 被遮蔽** / 与默认模式完全相同
- 预算三种维度耗尽后剩余项变 `UNKNOWN` 而非 `FAILED`
- 基线 diff 检出消失的模式、被替换的插件行、版本变化

`tests/yaml-diff.mjs` 是本项目最关键的验证：自研 YAML 读取器与 `js-yaml` 在本机 **全部真实 YAML 文件**（实测 307 个：DSH 各 `cordis.patch.yml`、所有 shipped preset、用户 preset、`settings.yaml` 等）上逐值比对，307/307 一致。它发现并已修复的两处差异是：多行 plain scalar（`description: …` 续写在缩进行上）曾导致整层被丢弃；flow 标量里不构成分隔的冒号（`tarball: file:F:/…`）曾被读成非法 flow mapping。比对时，超过 2^53 的整数按数值视为一致——本读取器保留精确数字串，`js-yaml` 转成 float64 会丢精度，两者是同一个 YAML 标量。

---

## 十、与方案的对应

| 方案章节 | 实现 |
|---|---|
| §1 三条硬约束 | 零依赖 ESM；不调用 `--dump-config`；预算 + 三种终点 |
| §2 系统架构 | `src/verify.mjs` 编排：环境守卫 → 快照采集 → 阶梯引擎 → 报告 |
| §3 产出物契约 | `src/artifacts.mjs`（七类）+ `src/ladder.mjs`（契约判定） |
| §4 阶梯引擎 | `src/ladder.mjs` `verifyPlugin` / `verifyMode` |
| §5 防空转 | `src/budget.mjs` + 单次尝试 + `UNKNOWN` + 无根因分析 |
| §6 三种运行方式 | CLI / `--format junit` / `prompt` 子命令 |
| §7 基线 Diff | `src/baseline.mjs` |
| §8 校验提示词 | `src/prompt.mjs`（`dsh-verify prompt`） |
| §9 编排层定位 | 本工具做判定；`--probe` 是 `dsh-plugin-doctor` 式沙箱冒烟的内建替代 |
| §10 单一数据结构 | 每项一行、五个字段：`item \| type \| status \| layer \| evidence` |

方案 §9 列出的社区工具（`dsh-fix`、`dsh-plugin-doctor`、`dsh-testkit`、`dsh-plugin-compare`、`awesome-dsh-presets`、`dsh-agent-preflight`）在本机均未安装，故本实现是自包含的，不依赖它们；它们各自可作为更深的后端接入 `--probe` 的位置。

---

## 十一、已知边界

诚实标注，不做能力外承诺：

1. **L4 无法真正触发。** 外部只读进程不能构造最小工具调用。工具存在且合法可以证明，工具**能回答**不能。所以 L4 恒为 `UNKNOWN`——这正是方案第三条硬规则要求的诚实。
2. **Cordis 服务键无法完全静态解析。** `class X extends Service` 的注册键由运行时决定。判不出来就是 `UNKNOWN`。
3. **`--probe` 是冒烟测试，不是挂载。** stub ctx 不提供真实服务；插件因缺服务而失败记为 `deferred UNKNOWN`（"未观测"），不是 `FAILED`。只有**导入期抛错**才是 `FAILED`——而如果缺的模块是 DSH 自己提供的同级依赖（`@deepseek-ai/dsh-settings` 等），同样记为"未观测"：隔离进程从插件目录解析模块，看不到安装目录的模块根，probe 现在会明确说明这一点，而不是指控一个能正常工作的插件。
4. **预设行的包解析按 DSH 的方式沿 harness base 向上查找**，与 profile 的 `node_modules` 链不同——两者都实现了，因为 DSH 对两者用不同算法。
5. **受限沙箱下 `spawn` 管道会 EPERM。** 工具会自动改用文件描述符重定向重试一次；挂载冒烟、启动器的 Node / `dsh` 探测与 `self-test` 都走同一回退，因此沙箱内外都能工作。
6. **配置来源的 provider id 判不出来，且刻意不猜。** 有些插件注册的 id 来自运行时配置而非代码常量，形态如下：
   ```js
   // @scope/example-speech
   providerId: z.string().min(1).default("example-voice")   // 配置默认值
   ctx.speechToText.register({ info: { id: config.providerId, … } })
   ```
   静态分析追不到 `config.providerId`，因此这类引用会报 `DEGRADED`（**误报**）。选择误报而非猜：
   猜错会把真正悬空的引用变成假 PASS，那比误报更糟。要真正修好需要解析 `schemastery` / `zod` 的
   `.default()` 并追踪 `config.<field>` 到注册点——**已知，未修**。
7. **模式校验在 DSH 0.2.0 上有缺口。** 0.2.0 起"模式"不再是 `<DSH_HOME>/.agent-presets/<id>/` 目录，
   而是配置层里的 `dsh-agent-preset` 行（见下节）。本工具目前只扫目录，因此在 0.2.0 上**发现 0 个模式**，
   并在报告里用 `NOTE` 明说这件事——而不是静默地当作通过。

---

## 十二、DSH 0.2.0 带来的三处变化

本工具最初针对 DSH 0.1.5 编写。在本机（DSH 0.2.0-rc.2）实测时暴露了三处不兼容，
已定位并修复前两处：

### 1. `dsh.bundle.patch` 可以是**有序数组**（已修）

DSH 自己的文档字符串写着 "one file, or an ordered list of files"。0.2.0 的
`@deepseek-ai/dsh-web-app` 就用了列表形式：

```json
"bundle": { "patch": [
  "./cordis.patch.yml",
  "./presets/standard.patch.yml",
  "./presets/ptc.patch.yml",
  "./presets/minimal.patch.yml",
  "./presets/cordis.patch.yml"
] }
```

原实现把整个数组交给 `path.resolve`，于是**直接崩溃**（`ERR_INVALID_ARG_TYPE`）。
即使只取第一个元素也只是不崩而已——`presets/*.patch.yml` 正是声明
`standard` / `ptc` / `minimal` / `cordis` 四个模式的那几层，漏掉它们会组合出一棵
**没有任何 agent preset** 的树。现在按顺序完整应用，缺失成员单独记为 problem。

### 2. 打包后的 provider id 需要跨形态解析（已修）

esbuild 打包的插件用类表达式声明 provider：

```js
var DeepSeekSearchProvider = class { resolveOptions; id = DEEPSEEK_PROVIDER_ID; … };
// …
ctx.web.registerSearchProvider(new DeepSeekSearchProvider(…));
const DEEPSEEK_PROVIDER_ID = "deepseek-official";
```

原实现只认 `const X = {` 和 `class X {` 两种形态，**认不出 `var X = class {`**，
于是 `capabilityId` 恒为 `null`，`deepseek-official` / `http` 这类**完全合法的绑定**
被报成"悬空引用"——三个假 `DEGRADED`。现在新增两条策略：
识别类表达式形态；`id` 指向标识符时，在包内解析该字符串常量，且**要求该常量在包内取值唯一**
才接受（有歧义就不猜）。

### 3. 模式不再是磁盘目录（**未修，已在报告中声明**）

0.2.0 的模式来自配置层里的行：

```yaml
- insert:
    - id: preset-standard
      name: '@deepseek-ai/dsh-agent-preset'
      config: { id: standard, order: 1, plugins: [ … ] }
```

本工具的模式发现只扫 `.agent-presets/<id>/` 目录，因此在 0.2.0 上找不到任何模式。
`verifyModes` 现在会在零发现时追加一行 `discoveryGap`，在文本报告里以 `NOTE` 呈现，
并明确写出两种可能（确实还没有模式 / 0.2.0 换了模型），不替读者猜。
该行带 `deferred`，**不进退出码**——"这次没看成"不是发现。
