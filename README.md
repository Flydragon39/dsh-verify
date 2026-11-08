# dsh-verify · DSH plugin & mode integrity verifier

An independent, external implementation of the *DSH Plugin and Mode Integrity
Verification Plan (final)*. Zero dependencies (Node.js standard library only),
read-only, and it runs entirely outside the DSH process.

It answers one question: **after a DSH update, which of my plugins and modes
silently stopped working?**

> **Just want to run it?** See [QUICKSTART.md](QUICKSTART.md): double-click
> `run.cmd` for a menu. Nothing to install, the profile is auto-detected, and
> Node is taken from the runtime DSH already ships when PATH has none.

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

With no `--profile`, the profile is auto-detected from `<DSH_HOME>/profiles`:
`web` when it exists, otherwise whichever profile the installation actually has.

A Chinese edition is at [`README.zh.md`](README.zh.md).

---

## Project

- **Author**: FlyDragon39
- **Upstream**: this project is built against **DSH (official edition)
  `0.2.0-rc.2`**, and every behaviour and compatibility claim here was tested
  against that version. Compatibility with later DSH versions has **not** been
  tested and is not guaranteed — a DSH update may change composition layers,
  preset discovery or artifact shapes, so re-validate this tool against it.
- **How it was made**: a personal project completed with AI assistance, and the
  author's first attempt at building and submitting code. Shortcomings are
  likely — issues, corrections and suggestions are very welcome.
- **License**: MIT, fully open source. Commercial use and modification are not
  restricted and it may be used freely; the only requirement is that the
  original author and this project are credited.

---

## 1. Why it must be independent of DSH

The plan's first hard constraint is **independence**: the verifier must not run
as a DSH plugin, or the same update that breaks your plugins breaks the thing
checking them.

Building it surfaced a fact that makes the constraint stricter than the plan
assumed:

**`dsh --profile <name> --dump-config` is not read-only.** Before printing the
composed tree, DSH rewrites the profile's `cordis.yml`
(`profile-boot-*.js:209`). Under a confined sandbox it does not merely mutate —
it fails outright:

```
Error: EPERM: operation not permitted, open '...\.dsh\profiles\web\cordis.yml'
    at prepareProfile (dsh/lib/profile-boot-Dk-7KqJc.js:209:2)
```

So this tool **never calls `--dump-config`**. It reads the configuration files
directly and performs the same layer fold in memory. The only dynamic call it
ever makes is `dsh --version`, and only when `--version-probe` is passed.

`dsh-verify` writes exactly two kinds of file, both at paths you name:
reports (`--output`) and baseline snapshots (`baseline save`). **It never
modifies DSH configuration.**

---

## 2. The core mechanism

### 2.1 Seven artifact classes (plan §3)

Plugin implementations are unbounded; what they *register* is not. Each artifact
is reported with `file:line` evidence.

| Artifact | Detected from | Judged by |
|---|---|---|
| tool | `defineTool({ name })` + `tools.register()` | unique name, description, parameter schema |
| command | `commands.register()` / `ctx.command()` | name is a usable slash token |
| hook | `ctx.on('event')` | event name is namespaced |
| service | `ctx.inject([...])`, `provide()`, `register*Provider()` | a publisher exists and can be consumed |
| prompt section | `systemPrompt.section({ name })` | named, with text |
| UI slot | `package.json` → `dsh.client` | declared client entry exists |
| **pure side effect** | none of the above | **mount success is the whole contract — stop** |

> A module's own `export const name` is identity metadata, not a product.
> Counting it would stop a plugin that registers nothing from ever reaching the
> plan's L2 terminal.

### 2.2 Capability binding — the highest-value check

The plan's headline failure mode is "the feature is gone and nothing errors".
Its canonical form:

```yaml
# profiles/web/cordis.patch.yml
- id: web
  config:
    searchProvider: ddg        # a string
    fetchProvider: safe-http   # a string
```

Both values name provider ids that plugins register **at mount time**. If
`acme-search` stops loading, those strings still parse, the tree still
composes, and **nothing reports an error** — search simply falls back to a
different engine.

Only cross-checking the configured value against the capability ids plugins
actually register makes it visible:

```
| web :: searchProvider | binding | PASS | L3 | "ddg" is published by acme-search |
| web :: fetchProvider  | binding | PASS | L3 | "safe-http" is published by acme-fetch |
```

The `register*Provider(...)` argument is frequently a **cross-file class
instance** (`new ExampleFetchProvider(...)` in `index.js`, `id = 'safe-http'` in
`provider.js`), so resolution spans files. When it cannot be resolved the tool
returns `null` rather than guessing: a guessed id would turn a genuinely
dangling reference into a false PASS.

---

## 3. The stepped engine (plan §4)

Each plugin advances L0 → L4. Any failing or terminating layer ends the item
immediately; nothing re-enters an earlier layer.

| Layer | Check | Passes when | Terminates when |
|---|---|---|---|
| **L0** | present in the composed tree | the row was composed | missing → `FAILED`; `disabled: true` → `PASS(mounted)` |
| **L1** | loads without error | manifest resolves, entry exists | unresolvable → `FAILED`; import throws under `--probe` → `FAILED` |
| **L2** | enumerate artifacts | ≥1 artifact class | none → `PASS(mounted)`, **stop** |
| **L3** | artifact contracts | contract satisfied | violated → `DEGRADED`; undecidable → `UNKNOWN` |
| **L4** | minimal behaviour trigger | a normal result | unconstructible → `UNKNOWN` (`deferred`) |

**L2 is the anti-spin gate:** no artifacts means stop, never L3/L4.

### A layer that does not load is a row, not a footnote

The composer reads the profile's bundle layers, the profile's own
`cordis.patch.yml`, the home layer and any `--patch` overlay. When one of them
cannot be read — a parse error, a declared patch file that is missing, a bundle
package that is not installed — every row it would have contributed is simply
*absent* from the tree, and everything below it still composes and passes.

That is the plan's headline silent failure, so each such problem becomes an
ordinary report row (`type: composition`, `FAILED`) rather than a section in the
Markdown report only. By construction it then reaches the verdict, the exit code,
the text report, the JSON and the JUnit export, and no format can disagree with
another about whether the run was complete.

### Modes (agent presets) — a separate ladder

A DSH "mode" is an **agent preset**: `agent.cordis.yml` (composition) plus
`preset.yml` (display metadata) under `<DSH_HOME>/.agent-presets/<id>/`. The plan
asks for four dynamic steps. An external process cannot switch a running
session's preset, so they map onto the same artefacts the switch would consume:

| Plan asks | External equivalent |
|---|---|
| switch to the mode | compose the preset's rows the way the roster would |
| confirm it is not bounced back | **detect an id shadowed by a higher-precedence root** — the actual mechanism of a bounce-back, and one DSH performs *silently* |
| diff the tool list and system prompt | resolve each row's package, diff tool/prompt-section sets against the default mode |
| no change → FAILED | identical row set and identical surface → `FAILED` |

The health pass reproduces DSH's own `discovery` order and wording step for
step. A malformed `preset.yml` is never fatal in DSH, so here it is only
`DEGRADED`.

> **Shadow detection** is the key finding here. DSH's `discoverPresets` runs
> `if (byId.has(preset.id)) continue;` across roots: the first root wins and
> later ones vanish. Your preset directory is present, its file is valid, and
> the mode never appears — DSH says nothing. This reports it as L0 `FAILED`.

---

## 4. Anti-spin (plan §5)

All four rules are implemented: a global ceiling of 20 probe calls and 180
seconds; one attempt per layer; `UNKNOWN` as a first-class state; and no
root-cause analysis.

### Why the budget has two ledgers

The plan's twenty is *tool calls*. A web profile composes 154 rows. If static
file reads counted, the ceiling would be spent before the tenth plugin and the
guarantee would degrade into a useless constant.

- `spend()` is a **probe**: it executes something (a child process, the `dsh`
  CLI, plugin code). Hard-capped at 20.
- `inspect()` is a **static read**: it counts toward the wall clock and the
  report, not the probe ceiling.

The 180-second wall clock applies to everything unconditionally. Both ledgers
are printed: `BUDGET: used 12 probe calls / 2 inspections / 2.39s of 20 / 180s`.

### `deferred`: an honest UNKNOWN

A static run cannot observe mount behaviour, so every plugin gets an L1
`UNKNOWN`. That is honest, but treating it as a finding would pin the exit code
at 1 forever and bury the real ones. `deferred` rows keep their `UNKNOWN` status
— the plan is explicit that UNKNOWN is first-class — while being counted
separately so they neither drive the exit code nor mask findings:

```
VERDICT: 1 UNKNOWN (3 row(s) deferred — rerun with --probe)
```

`deferred` means "a flag fixes this", not "a human must investigate".

---

## 5. Usage

### Mode 0 — the launcher

```powershell
run.cmd                  # double-click: interactive menu
run.cmd doctor           # environment only: Node, DSH install, profile list
run.cmd quick            # static sweep with an auto-detected profile
run.cmd quick --probe    # + the isolated mount smoke test
run.cmd self-test        # prove the tool's own wiring is intact
```

`run.cmd` / `run.ps1` locate Node themselves (`DSH_NODE` → PATH → the runtime DSH
ships), so they work on a machine with no Node on PATH. See
[QUICKSTART.md](QUICKSTART.md).

### Mode A — CLI

```bash
node bin/dsh-verify.mjs                                   # static sweep; profile auto-detected
node bin/dsh-verify.mjs --profile desktop                 # name a profile explicitly
node bin/dsh-verify.mjs --probe                           # + isolated mount smoke test
node bin/dsh-verify.mjs --modes-only --format json --output modes.json
node bin/dsh-verify.mjs --only acme-search,acme-fetch --probe
```

### Mode B — CI

```bash
node bin/dsh-verify.mjs --probe --format junit --output results.xml --quiet
```

Exit codes: `0` no findings · `1` FAILED or DEGRADED present · `2` BLOCKED.

### Mode C — an independent agent session

```bash
node bin/dsh-verify.mjs prompt
```

Prints the self-contained verification prompt (plan §8), realigned with actual
DSH semantics, ready to hand to another DSH session or agent runtime.

### Baseline diff (plan §7)

```bash
node bin/dsh-verify.mjs baseline save --output .dsh-verify/baseline.json --quiet   # before updating
node bin/dsh-verify.mjs --baseline .dsh-verify/baseline.json                       # after
```

The diff covers the plan's five categories: composed plugin rows per profile,
each profile's mode list, the expected tool list, each mode's tool set, and the
DSH version.

### Options

| Option | Meaning |
|---|---|
| `--profile <name>` | profile to compose (default `web`) |
| `--modes-only` / `--plugins-only` | modes only / skip modes |
| `--all` | verify every row, including the platform's own `@deepseek-ai/*` |
| `--only <a,b>` / `--modes <a,b>` | restrict plugins / modes |
| `--patch <file>` | extra patch overlay (repeatable) |
| `--dsh-home` / `--install` / `--dsh-bin` | name the environment explicitly |
| `--version-probe` | spend one budget call on a real `dsh --version` |
| `--probe` | import each plugin in an isolated process (mount smoke test) |
| `--budget-calls` / `--budget-seconds` | ceilings (default 20 / 180) |
| `--baseline <file>` / `--save-baseline <file>` | diff against / write a baseline |
| `--format text\|markdown\|json\|junit` | report format |
| `--output <file>` / `--quiet` | write to a file / suppress the summary |

---

## 6. Scope: the platform is the environment, not the subject

The plan is explicit: *"plugins and modes are the protagonists; the platform is
only an environment guard."*

The default sweep therefore verifies **the plugins you installed** and merely
counts the platform's rows:

```
- Composed rows: 191
- Scope: the rows you installed; 188 platform row(s) treated as the environment
  and not verified (pass `--all` to include them)
```

`--all` sweeps all 191. Expect many UNKNOWNs there: platform-internal Cordis
service keys are frequently not derivable from a file, and that is noise rather
than signal.

When a deployment composes **only** platform rows, the default scope has nothing
to check — and that is reported as a `NOTE` ("nothing outside the platform was
verified") rather than as a silent `ALL PASS`, so an empty sweep cannot be
mistaken for a clean one. `--all` turns it into a real sweep.

---

## 7. Testing

```bash
node tests/verify.test.mjs     # 44 behavioural tests
node tests/yaml-diff.mjs       # differential test of the YAML reader vs js-yaml
```

`tests/verify.test.mjs` builds a complete miniature harness home
(`tests/fixtures/env`) in which **every fixture is broken in exactly one way**,
then asserts that this break produces exactly the verdict the plan requires:
healthy plugins; a plugin that registers nothing terminating at L2; a duplicate
tool name degrading both sides; a dangling provider reference; an import that
throws (UNKNOWN statically, FAILED under `--probe`, deferred when the missing
peer is one the harness supplies); a missing composition; an unparsable one; an
unresolvable row; an invalid directory name; a **shadowed id**; a mode identical
to the default; a **dropped patch layer promoted to a FAILED row**; budget
exhaustion on three dimensions; and baseline diffs.

`tests/yaml-diff.mjs` is the project's most important validation: the built-in
YAML reader is compared value-for-value against `js-yaml` over **every YAML file
found on this machine** (307 at the time of writing — DSH `cordis.patch.yml`
files, all shipped presets, the user presets and `settings.yaml`), and all 307
agree. Two divergences this test found are fixed: a multi-line plain scalar
(`description: …` continued on indented lines) used to abort the whole layer, and
a flow scalar containing a non-separating colon (`tarball: file:F:/…`) used to
be read as a malformed flow mapping. The comparison treats an integer beyond
2^53 as agreeing between the exact digit string this reader keeps and the rounded
float `js-yaml` produces: same YAML scalar, different JavaScript representation.

---

## 8. Plan traceability

| Plan section | Implementation |
|---|---|
| §1 three hard constraints | zero-dependency ESM; no `--dump-config`; budget with three terminals |
| §2 architecture | `src/verify.mjs`: guard → snapshot → stepped engine → report |
| §3 artifact contracts | `src/artifacts.mjs` (seven classes) + `src/ladder.mjs` (contract checks) |
| §4 stepped engine | `src/ladder.mjs` `verifyPlugin` / `verifyMode` |
| §5 anti-spin | `src/budget.mjs`, single attempt, `UNKNOWN`, no root-cause analysis |
| §6 three run modes | CLI, `--format junit`, the `prompt` subcommand |
| §7 baseline diff | `src/baseline.mjs` |
| §8 verification prompt | `src/prompt.mjs` (`dsh-verify prompt`) |
| §9 orchestration layer | this tool decides; `--probe` is a built-in `dsh-plugin-doctor`-style sandbox smoke test |
| §10 one data structure | one row per item, five fields: `item \| type \| status \| layer \| evidence` |

None of the community tools the plan names (`dsh-fix`, `dsh-plugin-doctor`,
`dsh-testkit`, `dsh-plugin-compare`, `awesome-dsh-presets`,
`dsh-agent-preflight`) are installed on this machine, so the implementation is
self-contained; each remains a candidate deeper backend behind `--probe`.

---

## 9. Known limits

Stated plainly, because claiming more would be the dishonesty the plan forbids:

1. **L4 cannot truly trigger.** An external read-only process cannot construct a
   minimal tool invocation. That a tool exists and is well formed is provable;
   that it *answers* is not. L4 is therefore always `UNKNOWN`.
2. **Cordis service keys are not fully resolvable statically.** A
   `class X extends Service` registers under a key the runtime decides.
   Undecidable means `UNKNOWN`.
3. **`--probe` is a smoke test, not a mount.** The stub context supplies no real
   services, so a plugin failing for a missing service is reported as deferred
   `UNKNOWN` ("not observed"), never `FAILED`. Only an *import-time throw* is a
   `FAILED` — and even that is deferred when the missing module is one DSH itself
   supplies (`@deepseek-ai/dsh-settings` and other peers): the isolated process
   resolves from the plugin's directory and cannot see the installation's module
   roots, so the probe reports "not observed" instead of accusing a healthy
   plugin.
4. **Preset rows resolve packages by walking up from the harness base**, which is
   a different algorithm from the profile's `node_modules` chain. Both are
   implemented, because DSH uses both.
5. **A confined sandbox denies piped stdio to children.** The runner retries
   once with file-descriptor redirection, and every subprocess probe (the mount
   smoke test, the launcher's Node and `dsh` discovery, and `self-test`) uses the
   same fallback, so the tool works inside and outside one.
6. **A config-derived provider id is unresolvable, and deliberately not guessed.**
   Some plugins register an id that comes from runtime configuration rather than
   a code constant, in this shape:
   ```js
   // @scope/example-speech
   providerId: z.string().min(1).default("example-voice")   // config default
   ctx.speechToText.register({ info: { id: config.providerId, … } })
   ```
   Static analysis cannot trace `config.providerId`, so such a reference is
   reported `DEGRADED` — a **false positive**. Reporting it is the deliberate
   choice: a guessed id would turn a genuinely dangling reference into a false
   `PASS`, which is strictly worse. Fixing it properly means resolving
   `schemastery` / `zod` `.default()` values and tracing `config.<field>` to the
   registration site. **Known, not fixed.**
7. **Mode verification has a gap on DSH 0.2.0.** Since 0.2.0 a "mode" is no
   longer a `<DSH_HOME>/.agent-presets/<id>/` directory but a `dsh-agent-preset`
   row inside a composition layer (see below). This build scans directories only,
   so it discovers **0 modes** on 0.2.0 and says so with a `NOTE` row rather than
   silently appearing to pass.

---

## 10. What DSH 0.2.0 changed

This tool was first written against DSH 0.1.5. Running it against a real
0.2.0-rc.2 installation exposed three incompatibilities. The first two are fixed;
the third is declared in the report.

### 1. `dsh.bundle.patch` may be an ordered array (fixed)

DSH's own docblock reads "one file, or an ordered list of files", and 0.2.0's
`@deepseek-ai/dsh-web-app` uses the list form:

```json
"bundle": { "patch": [
  "./cordis.patch.yml",
  "./presets/standard.patch.yml",
  "./presets/ptc.patch.yml",
  "./presets/minimal.patch.yml",
  "./presets/cordis.patch.yml"
] }
```

The previous implementation handed the whole array to `path.resolve` and
**crashed outright** (`ERR_INVALID_ARG_TYPE`). Taking only the first entry would
merely have avoided the crash: the `presets/*.patch.yml` layers are exactly what
declares the `standard`, `ptc`, `minimal` and `cordis` modes, so dropping them
composes a tree with **no agent preset at all**. The declared list is now applied
in full and in order, and a missing member is reported as a problem.

### 2. Bundled provider ids need a second declaration shape (fixed)

esbuild-bundled plugins declare providers with a class expression:

```js
var DeepSeekSearchProvider = class { resolveOptions; id = DEEPSEEK_PROVIDER_ID; … };
// …
ctx.web.registerSearchProvider(new DeepSeekSearchProvider(…));
const DEEPSEEK_PROVIDER_ID = "deepseek-official";
```

The resolver understood `const X = {` and `class X {` but **not `var X = class {`**,
so `capabilityId` was always `null` and perfectly valid bindings —
`deepseek-official`, `http` — were reported as dangling references: three false
`DEGRADED` rows. Two strategies were added: recognise the class-expression form,
and, when the `id` field names an identifier, resolve that string constant within
the package — accepting it only when the constant has a single value package-wide,
so an ambiguous name is never guessed.

### 3. Modes are no longer directories (not fixed; declared in the report)

A 0.2.0 mode is a composition row:

```yaml
- insert:
    - id: preset-standard
      name: '@deepseek-ai/dsh-agent-preset'
      config: { id: standard, order: 1, plugins: [ … ] }
```

Mode discovery here still scans `.agent-presets/<id>/` directories and therefore
finds nothing on 0.2.0. `verifyModes` now appends a `discoveryGap` row when it
discovers zero presets; the text report prints it as a `NOTE` that names both
possible causes (no preset yet, or the 0.2.0 model) instead of choosing for the
reader. The row carries `deferred`, so it stays **out of the exit code** — "this
run could not look" is not a finding.
