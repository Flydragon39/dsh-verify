# 快速上手 · dsh-verify

> 目标：**双击就能跑**，不需要装任何东西，不需要记路径，不需要配 PATH。

---

## 一、最快的一条路（Windows）

在资源管理器里双击：

```
dsh-verify\run.cmd
```

出现的不是一堆报错，而是一个菜单：

```
dsh-verify 0.2.0 — ~\.dsh · DSH 0.2.0-rc.2

  1) 快速校验 (static)        组合配置树，检查契约与绑定；不启动子进程
  2) 完整校验 (--probe)       额外在隔离进程里导入每个插件
  3) 仅模式校验 (--modes-only)
  4) 保存基线 (baseline save)
  5) 与基线对比 (baseline diff)
  6) 环境自检 (doctor)
  7) 生成 Markdown 报告

  profile: desktop
选择 [1-7, q]:
```

敲 `1` 再回车即可。

**关掉窗口前会停住**，所以不会一闪而过看不到结果。

---

## 二、如果你更喜欢命令行

三个等价入口，随便挑一个：

```powershell
# 1) 批处理 shim（和双击同一个东西，可传参）
.\run.cmd doctor
.\run.cmd quick --probe

# 2) PowerShell shim
.\run.ps1 doctor

# 3) 直接调 Node（任何平台）
node bin\dsh-verify.mjs doctor
node bin\dsh-verify.mjs quick
node bin\dsh-verify.mjs quick --probe
```

`npm run quick` / `npm run doctor` / `npm run menu` 也可以，但**不需要 `npm install`**——
本工具零依赖（只用 Node 标准库），`node_modules` 是空的。

---

## 三、四条最常用的命令

| 命令 | 干什么 | 什么时候用 |
|---|---|---|
| `doctor` | 只体检，不校验：Node、DSH 安装、Harness home、有哪些 profile | 第一次用、换机器、报错时 |
| `quick` | 自动挑 profile 跑静态校验（约 1.5 秒） | 日常 |
| `quick --probe` | 加上隔离进程挂载冒烟测试 | DSH 更新之后 |
| `baseline save` | 存快照 | **升级 DSH 之前** |

升级前后的标准动作：

```powershell
# 升级前
.\run.cmd baseline save --output .dsh-verify\baseline.json

# 升级 DSH ...

# 升级后：先看差异，再看全量
.\run.cmd --baseline .dsh-verify\baseline.json
.\run.cmd quick --probe
```

---

## 四、没有 Node 怎么办？

**不用管。** 启动器按顺序自己找：

1. 环境变量 `DSH_NODE` 指向的 `node.exe`
2. PATH 里的 `node`
3. **DSH 自带的 Node 运行时**：
   `%USERPROFILE%\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe`

第 3 条是关键：装了 DSH 的机器上几乎一定有它，所以本工具在**没装 Node 的机器上也能跑**。
本机实测就是走第 3 条（`v24.21.0`）。

三个都找不到时，脚本会明确告诉你查过哪些位置，而不是抛一句看不懂的错。

---

## 五、profile 不用再手填

老版本默认 `--profile web`。但在真实机器上 `web` 往往**不存在**——
`dsh-desktop`、`dsh-pocket` 会建自己的 profile——于是默认运行会报一个
"profile directory not found" 的 FAILED，说的其实是默认值，不是你的安装。

现在不带 `--profile` 时会**自动探测** `<DSH_HOME>/profiles`：

- 有 `web` 就用 `web`（沿用 DSH 自带模板）
- 否则用实际存在的那个

显式传 `--profile` 依然优先。想看有哪些：

```powershell
.\run.cmd doctor
```

---

## 六、想让裸命令 `dsh-verify` 也能用（可选）

**不推荐也不必要**，因为 `run.cmd` 已经能在任何地方用。但如果你确实想要：

```powershell
node bin\dsh-verify-launcher.mjs install-path --dir <一个已经在 PATH 里的目录>
```

它会先把**确切要写的路径打印出来**再写。回收：

```powershell
node bin\dsh-verify-launcher.mjs install-path --dir <同一个目录> --remove
```

> 注意：这里写的是**磁盘上的 shim 文件**，与环境变量 PATH 无关，也**不碰 DSH 的任何配置**。

---

## 七、验证安装是否完好

```powershell
.\run.cmd self-test
```

```
dsh-verify self-test

  ok    tool root resolves             ...\dsh-verify\bin\dsh-verify.mjs
  ok    node >= 18                     v24.21.0 (bundled runtime)
  ok    CLI --version                  dsh-verify 0.2.0
  ok    CLI --help                     usage printed
  ok    prompt subcommand              4644 bytes
  ok    harness home readable          ~\.dsh
  ok    package.json engines honoured  bin declared
  ok    no DSH plugin surface          no dsh/cordis manifest keys, no composition files

  all checks passed
```

最后一条不是装饰：它**主动断言本工具没有变成 DSH 插件**（见下节）。

---

## 八、重要：这不是 DSH 插件，也不是模式

这是刻意的设计，不是遗漏。

校验器的唯一职责是回答"DSH 更新后哪些插件和模式静默失效了"。
如果它自己是个 DSH 插件，那么**它会被同一次更新一起弄坏**——
一个和故障同归于尽的报警器没有价值。所以：

- 本工具**不注册**任何 Cordis 组件，不导出 `apply` / `name` / `inject`
- `package.json` 里**没有** `dsh`、`cordis` 键
- 没有 `cordis.patch.yml`、`agent.cordis.yml`、`preset.yml`
- 不会出现在 `dsh plugin list` 里，不会被任何 profile 的 bundles 引用
- **永不修改 DSH 配置**。它只写两种文件：你显式指定路径的报告（`--output`）和基线（`baseline save`）

依赖方向是单向的：**本工具观察 DSH，DSH 不知道本工具存在。**

前七节讲的启动器、shim、菜单，全都是**工具外部的便利层**，
不改变这条边界——它们只是让同一份校验代码更容易被叫起来。

---

## 九、退出码

| 码 | 含义 | 该怎么办 |
|---|---|---|
| `0` | 没有发现 | 什么都不用做 |
| `1` | 有 `FAILED` 或 `DEGRADED` | 看报告里那一行 |
| `2` | `BLOCKED` 或环境没准备好 | 先跑 `doctor` |

`run.cmd` / `run.ps1` 会**原样透传**这个退出码，所以可以直接接进 CI。

---

## 十、结果怎么读

输出每行五个字段，和方案 §10 一致：

```
item | type | status | layer | evidence
```

- **状态**：`PASS` / `PASS(mounted)` / `DEGRADED` / `FAILED` / `UNKNOWN`
- **`UNKNOWN` 是一等状态**，不是错误——文件判不了就诚实说判不了，不猜
- 标了 `deferred` 的 `UNKNOWN` 意思是"换个参数就能解决"（比如加 `--probe`），
  它**不参与退出码**，免得淹没真正的发现

---

## 十一、一种已知的误报形态

用 `--probe` 或静态模式跑，可能出现这样一行：

```
example-speech :: defaultProvider   "example-voice" matches no capability id
                                    published by any composed plugin
```

**这是误报，原因已定位**：有些插件注册时用的 id 来自**运行时配置**，而不是代码里的字符串常量：

```js
// @scope/example-speech
providerId: z.string().min(1).default("example-voice")   // 配置默认值
// ...
ctx.speechToText.register({ info: { id: config.providerId, ... } })
```

静态分析无法把 `config.providerId` 追到这个默认值上，所以工具**拒绝猜**
（猜错会把真正悬空的引用变成假 PASS——那比误报更糟）。

**已知并接受**，不是待修列表里排第一的东西。真要修，方向是解析
`schemastery` / `zod` 的 `.default()` 并追踪 `config.<field>` 到注册点——
那是另一个量级的静态分析工作。

---

## 十二、下一步

- 校验语义、产物契约、阶梯引擎、预算：见 [README.zh.md](README.zh.md)
- 模式校验（agent preset）在 **DSH 0.2.0** 上有已知缺口：0.2.0 起模式不再是
  `.agent-presets/<id>/` 目录，而是配置层里的 `dsh-agent-preset` 行。
  本工具的报告会**明说自己找不到模式**，而不是假装通过。
