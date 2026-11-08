# Quick start · dsh-verify

> Goal: **double-click and it runs.** No install, no path to remember, no PATH entry.

---

## 1. The fastest path (Windows)

Double-click, from Explorer:

```
dsh-verify\run.cmd
```

You get a menu, not a wall of errors:

```
dsh-verify 0.2.0 — ~\.dsh · DSH 0.2.0-rc.2

  1) 快速校验 (static)        compose the tree, check contracts and bindings
  2) 完整校验 (--probe)       also import every plugin in an isolated process
  3) 仅模式校验 (--modes-only)
  4) 保存基线 (baseline save)
  5) 与基线对比 (baseline diff)
  6) 环境自检 (doctor)
  7) 生成 Markdown 报告

  profile: desktop
选择 [1-7, q]:
```

Type `1` and press Enter.

The window **stays open** afterwards, so a result never flashes past unseen.

---

## 2. If you prefer a shell

Three equivalent entry points:

```powershell
# 1) batch shim (same thing the double-click runs, but takes arguments)
.\run.cmd doctor
.\run.cmd quick --probe

# 2) PowerShell shim
.\run.ps1 doctor

# 3) call Node directly (any platform)
node bin\dsh-verify.mjs doctor
node bin\dsh-verify.mjs quick
node bin\dsh-verify.mjs quick --probe
```

`npm run quick` / `npm run doctor` / `npm run menu` work too, but **no `npm install`
is needed** — the tool has zero dependencies and uses only the Node standard library.

---

## 3. The four commands that matter

| Command | What it does | When |
|---|---|---|
| `doctor` | Checks the environment only: Node, DSH install, harness home, profiles | First run, new machine, anything looks wrong |
| `quick` | Static verification with an auto-detected profile (~1.5 s) | Everyday |
| `quick --probe` | Adds the isolated-process mount smoke test | After a DSH update |
| `baseline save` | Snapshots the current state | **Before** upgrading DSH |

The upgrade routine:

```powershell
# before upgrading
.\run.cmd baseline save --output .dsh-verify\baseline.json

# ... upgrade DSH ...

# after: diff first, then a full check
.\run.cmd --baseline .dsh-verify\baseline.json
.\run.cmd quick --probe
```

---

## 4. What if Node is not installed?

**You do not need to care.** The launcher looks, in order:

1. `DSH_NODE`
2. `node` on PATH
3. **the Node runtime DSH ships**:
   `%USERPROFILE%\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe`

Step 3 is what makes this work on a machine with no Node at all — and it is the
step that actually fires on this machine (`v24.21.0`).

If all three fail, the script prints exactly which locations it checked instead
of an opaque error.

---

## 5. The profile is auto-detected

Older builds defaulted to `--profile web`. On a real machine `web` often **does
not exist** — `dsh-desktop` and `dsh-pocket` create their own — so the default
run reported a "profile directory not found" FAILED that described the default
rather than your installation.

Now, with no `--profile`, the tool **auto-detects** from `<DSH_HOME>/profiles`:

- `web` if it exists (DSH's own template), otherwise
- the profile that actually does

An explicit `--profile` still wins. To see what is available:

```powershell
.\run.cmd doctor
```

---

## 6. Making the bare name `dsh-verify` work (optional)

**Not needed**, since `run.cmd` already works from anywhere. But if you want it:

```powershell
node bin\dsh-verify-launcher.mjs install-path --dir <a directory already on PATH>
```

It prints the **exact path it will write** before writing it. To undo:

```powershell
node bin\dsh-verify-launcher.mjs install-path --dir <same directory> --remove
```

> This writes a shim **file**; it is unrelated to the PATH environment variable
> and **does not touch any DSH configuration**.

---

## 7. Checking the install is intact

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

That last check is not decoration: it **asserts that this tool has not become a
DSH plugin** (see below).

---

## 8. Important: this is not a DSH plugin, and not a mode

That is deliberate, not an omission.

The verifier's only job is to answer "after the DSH update, which of my plugins
and modes silently stopped working?" If it were itself a DSH plugin, **the same
update would break it too** — and an alarm that dies with the fault it watches
for is worthless. Therefore:

- It registers **no** Cordis component and exports no `apply` / `name` / `inject`
- `package.json` has **no** `dsh` or `cordis` key
- There is no `cordis.patch.yml`, no `agent.cordis.yml`, no `preset.yml`
- It never appears in `dsh plugin list` and is referenced by no profile bundle
- It **never modifies your DSH configuration**. It writes exactly two kinds of
  file: a report at a path you name (`--output`) and a baseline (`baseline save`)

The dependency arrow points one way: **this tool observes DSH; DSH does not know
this tool exists.**

The launcher, shims and menu described above are all an **external convenience
layer**. They do not move that boundary — they only make the same verification
code easier to invoke.

---

## 9. Exit codes

| Code | Meaning | What to do |
|---|---|---|
| `0` | No findings | Nothing |
| `1` | A `FAILED` or `DEGRADED` exists | Read that row |
| `2` | `BLOCKED`, or the environment is not ready | Run `doctor` first |

`run.cmd` / `run.ps1` pass this code through unchanged, so they drop straight
into CI.

---

## 10. Reading the output

Every row carries the plan's five fields:

```
item | type | status | layer | evidence
```

- **Status**: `PASS` / `PASS(mounted)` / `DEGRADED` / `FAILED` / `UNKNOWN`
- **`UNKNOWN` is a first-class state**, not an error — when files cannot decide,
  the tool says so rather than guessing
- A `deferred` `UNKNOWN` means "a different flag would settle this" (e.g. add
  `--probe`). It **does not affect the exit code**, so it cannot drown out real
  findings

---

## 11. One shape that is known to be a false positive

A static or `--probe` run can report a row like this:

```
example-speech :: defaultProvider   "example-voice" matches no capability id
                                    published by any composed plugin
```

**It is a false positive, and the cause is identified.** Some plugins register an
id that comes from **runtime configuration**, not from a string constant in the
code:

```js
// @scope/example-speech
providerId: z.string().min(1).default("example-voice")   // config default
// ...
ctx.speechToText.register({ info: { id: config.providerId, ... } })
```

Static analysis cannot trace `config.providerId` back to that default, so the
tool **refuses to guess** — guessing wrong turns a genuinely dangling reference
into a false PASS, which is strictly worse than a false alarm.

**Known and accepted.** Fixing it properly means resolving `schemastery` / `zod`
`.default()` values and tracing `config.<field>` to the registration site — a
different order of static-analysis work.

---

## 12. Where next

- Verification semantics, artifact contracts, the ladder engine and the budget:
  see [README.md](README.md)
- Mode (agent-preset) verification has a **known gap on DSH 0.2.0**: since 0.2.0
  modes are no longer `.agent-presets/<id>/` directories but `dsh-agent-preset`
  rows inside composition layers. The report **says it found no modes** rather
  than pretending to have passed.
