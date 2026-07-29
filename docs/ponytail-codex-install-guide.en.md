# Install Ponytail in Codex App and CLI

Language: [한국어](ponytail-codex-install-guide.ko.md) · **English** · [简体中文](ponytail-codex-install-guide.zh-CN.md)

This runbook lets another Codex agent install [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) reproducibly in Codex App and CLI on Windows.

Verified baseline:

- Codex CLI `0.145.0`
- Ponytail `4.8.4`
- pinned commit `16f29800fd2681bdf24f3eb4ccffe38be3baec6b`
- Windows PowerShell with Node.js and Git on `PATH`

This changes user-level Codex configuration. Before editing, an agent must show the exact target path and current contents and obtain the user's approval.

## Installation rules

1. Do not modify upstream files.
2. Pin the full reviewed commit SHA, not a branch or tag.
3. Preserve existing marketplace entries and add only Ponytail.
4. Inspect hooks after installation and approve them through Codex's trust UI.
5. Do not install benchmarks, telemetry, automatic updates, or a separate router.

> Important: the upstream `.agents/plugins/marketplace.json` at this commit points its plugin source to `ref: main`. Therefore, merely running `codex plugin marketplace add DietrichGebert/ponytail --ref <SHA>` does not guarantee that the plugin source itself stays pinned. The procedure below pins the plugin URL directly in a local marketplace manifest.

## 1. Preflight

```powershell
Get-Command codex, git, node
codex --version
codex plugin marketplace list --json

$codexRoot = Join-Path $env:USERPROFILE '.codex'
$marketRoot = Join-Path $codexRoot 'local-marketplaces\personal'
$manifest = Join-Path $marketRoot '.agents\plugins\marketplace.json'
$pin = '16f29800fd2681bdf24f3eb4ccffe38be3baec6b'

Test-Path -LiteralPath $manifest
if (Test-Path -LiteralPath $manifest) {
  Test-Json (Get-Content -Raw -LiteralPath $manifest)
  Get-Content -Raw -LiteralPath $manifest
}
```

Stop if:

- `codex`, `git`, or `node` is missing;
- the existing manifest is not valid JSON;
- a `ponytail` entry already exists with a different URL or `ref`; or
- the edit would overlap unreviewed user changes.

## 2. Prepare the personal marketplace

### No manifest exists

Create its parent directory:

```powershell
New-Item -ItemType Directory -Force -Path (Split-Path $manifest -Parent)
```

Use Codex `apply_patch` to create `$manifest` with:

```json
{
  "name": "personal",
  "interface": {
    "displayName": "Personal"
  },
  "plugins": [
    {
      "name": "ponytail",
      "source": {
        "source": "url",
        "url": "https://github.com/DietrichGebert/ponytail.git",
        "ref": "16f29800fd2681bdf24f3eb4ccffe38be3baec6b"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
```

### A manifest already exists

Create a recoverable, non-overwriting backup first:

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$backup = "$manifest.backup-before-ponytail-$stamp"
Copy-Item -LiteralPath $manifest -Destination $backup
Get-FileHash -Algorithm SHA256 -LiteralPath $manifest, $backup
```

After confirming equal hashes, use `apply_patch` to append exactly this object to the existing `plugins` array. Do not reorder or reserialize existing entries.

```json
{
  "name": "ponytail",
  "source": {
    "source": "url",
    "url": "https://github.com/DietrichGebert/ponytail.git",
    "ref": "16f29800fd2681bdf24f3eb4ccffe38be3baec6b"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Productivity"
}
```

Validate the result:

```powershell
Test-Json (Get-Content -Raw -LiteralPath $manifest)

if ($backup) {
  git diff --no-index -- $backup $manifest
}

$market = Get-Content -Raw -LiteralPath $manifest | ConvertFrom-Json
$pony = @($market.plugins | Where-Object name -eq 'ponytail')
if ($pony.Count -ne 1) { throw 'Ponytail entry must exist exactly once' }
if ($pony[0].source.url -ne 'https://github.com/DietrichGebert/ponytail.git') {
  throw 'Unexpected Ponytail source URL'
}
if ($pony[0].source.ref -ne $pin) { throw 'Unexpected Ponytail ref' }
```

`git diff --no-index` returns exit code `1` when it finds a difference; that is expected here.

## 3. Register the marketplace and install

Register `personal` only if it is absent:

```powershell
$marketplaces = (codex plugin marketplace list --json | ConvertFrom-Json).marketplaces
$personal = @($marketplaces | Where-Object name -eq 'personal')

if ($personal.Count -eq 0) {
  codex plugin marketplace add $marketRoot --json
} elseif ($personal.Count -ne 1 -or $personal[0].root -ne $marketRoot) {
  throw 'A different personal marketplace is already registered'
}
```

Confirm discovery, install, and check status:

```powershell
codex plugin list | Select-String -Pattern 'Marketplace `personal`|ponytail@personal' -Context 0,1
codex plugin add ponytail@personal --json
codex plugin list | Select-String -Pattern 'ponytail@personal' -Context 0,1
```

Expected: `ponytail@personal`, `installed, enabled`, version `4.8.4`.

## 4. Verify the installed source and hooks

Do not rely only on the install response. Inspect the actual Codex cache:

```powershell
$pluginBase = Join-Path $codexRoot 'plugins\cache\personal\ponytail'
$installedManifest = Get-ChildItem -LiteralPath $pluginBase -Recurse -Filter plugin.json |
  Where-Object {
    $_.Directory.Name -eq '.codex-plugin' -and
    (Get-Content -Raw -LiteralPath $_.FullName | ConvertFrom-Json).name -eq 'ponytail'
  } |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

if (-not $installedManifest) { throw 'Installed Ponytail manifest not found' }

$pluginRoot = Split-Path (Split-Path $installedManifest.FullName -Parent) -Parent
$plugin = Get-Content -Raw -LiteralPath $installedManifest.FullName | ConvertFrom-Json
$actualPin = (git -C $pluginRoot rev-parse HEAD).Trim()

if ($plugin.version -ne '4.8.4') { throw "Unexpected version: $($plugin.version)" }
if ($actualPin -ne $pin) { throw "Unexpected commit: $actualPin" }

$hookFile = Join-Path $pluginRoot 'hooks\claude-codex-hooks.json'
$hookConfig = Get-Content -Raw -LiteralPath $hookFile | ConvertFrom-Json
$actualHooks = @($hookConfig.hooks.PSObject.Properties.Name | Sort-Object)
$expectedHooks = @('SessionStart', 'SubagentStart', 'UserPromptSubmit') | Sort-Object
if (Compare-Object $expectedHooks $actualHooks) { throw 'Unexpected hook set' }

Get-Content -Raw -LiteralPath $installedManifest.FullName
Get-Content -Raw -LiteralPath $hookFile
Get-Content -Raw -LiteralPath (Join-Path $pluginRoot 'hooks\ponytail-activate.js')
Get-Content -Raw -LiteralPath (Join-Path $pluginRoot 'hooks\ponytail-subagent.js')
Get-Content -Raw -LiteralPath (Join-Path $pluginRoot 'hooks\ponytail-mode-tracker.js')
```

Confirm that:

- `node` is the only executable;
- only the three scripts under the installed plugin root run;
- each timeout is five seconds;
- the only lifecycle groups are `SessionStart`, `SubagentStart`, and `UserPromptSubmit`; and
- no MCP server, connector, telemetry, or benchmark execution is declared.

## 5. Trust hooks and activate App/CLI

Do not automate or bypass this step.

1. Start an interactive Codex CLI session.
2. Open `/hooks` and recheck the three commands and paths.
3. Trust the three hooks.
4. Exit and start a fresh CLI session.
5. Fully restart the Codex desktop app.

In a fresh session, invoke `@ponytail`. Expect an equivalent response:

```text
PONYTAIL MODE ACTIVE — level: full
```

Also confirm that `@ponytail-review` and `@ponytail-help` are discoverable. Do not run `@ponytail-gain` or a benchmark as part of installation verification.

`SubagentStart` applies the same rules when a real task uses a subagent. Do not create a needless subagent only to test installation.

## 6. Coexistence with Superpowers

If Superpowers is already installed, do not change its configuration:

```powershell
codex plugin list | Select-String -Pattern 'superpowers@openai-curated|ponytail@personal' -Context 0,1
```

Recommended policy:

- keep Ponytail automatically active in its default `full` mode;
- invoke Superpowers explicitly only for complex or high-risk work;
- when both apply, Superpowers owns planning, TDD, and verification while Ponytail minimizes scope and implementation; and
- never minimize away security, trust-boundary validation, data-loss prevention, accessibility, or tests the user requested.

## 7. Completion checklist

- [ ] The marketplace JSON is valid.
- [ ] Existing marketplace entries remain unchanged.
- [ ] Exactly one Ponytail entry exists.
- [ ] The URL and full commit SHA match.
- [ ] `ponytail@personal` is `installed, enabled`.
- [ ] Version is `4.8.4` and cached Git HEAD equals the pin.
- [ ] Exactly three reviewed hooks were trusted.
- [ ] Fresh CLI and App sessions activate `full` automatically.
- [ ] Superpowers and other plugins retain their prior state.
- [ ] No benchmark, telemetry, router, or automatic update was added.

## 8. Disable or remove

For the current session only, enter either:

```text
@ponytail off
```

```text
stop ponytail
```

Remove the installed plugin with:

```powershell
codex plugin remove ponytail@personal --json
```

For complete cleanup, back up the current manifest and remove only the Ponytail object with a minimal reverse patch. Do not restore an old backup wholesale because that could erase later user changes. Do not remove the `personal` marketplace while it still contains another plugin.

## 9. Update policy

This installation uses a fixed pointer and does not follow new upstream versions automatically. Handle an update as a separate reviewed task:

1. Review the new commit's plugin manifest, three hooks, and diff.
2. Back up the marketplace manifest.
3. Change only `source.ref` to the approved full SHA.
4. Remove and reinstall the plugin.
5. Recheck cached Git HEAD and the hook set.
6. Re-review trust if any hook changed.

Never switch to `main` or enable automatic updates without review.
