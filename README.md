# My Agents (`rafa.agents`)

Fork of Omarchy's **Agents** bar widget (`omarchy.agents`). One bar icon and one panel for every AI coding subscription on the machine: plan limits, tokens by day and model, prepaid balance (Fireworks), and optional cross-device sync.

This is a **cloned plugin with improvements**. It does not replace the package in `/usr/share/omarchy/`. Install under `~/.config/omarchy/plugins/rafa.agents/` and use the id `rafa.agents` in your bar layout.

## Differences from stock `omarchy.agents`

| Area | Stock Omarchy | This fork (`rafa.agents`) |
|---|---|---|
| **Providers** | Claude Code, Codex, Fireworks | + **Cursor** and **Kiro CLI** (monthly plan limits) |
| **Bar** | Fixed glyph (`󱚣`) via `BarIconButton` | Provider icon + text: `Codex 5h - 1%`, `Cursor 1m - 12%`, etc. |
| **Vertical bar** | Same glyph only | Fallback icon; horizontal layout shows icon + label |
| **Codex CLI** | Collector calls `codex` with `-a untrusted` | `bin/codex` shim maps `untrusted` → `never` (Codex CLI ≥ 0.151.0) |
| **Usage refresh** | `omarchy-agent-usage-update` directly | `bin/usage-update`: Codex shim on `PATH` + plugin collectors |
| **Collectors** | Omarchy package only | Stock Omarchy collectors + plugin `bin/cursor` and `bin/kiro` |
| **IPC / shell** | `omarchy-shell omarchy.agents …` | `omarchy-shell rafa.agents …` |

### Cursor (new)

Stock Omarchy **does not** include Cursor. This fork adds:

- Collector `bin/cursor`: session token from `~/.config/Cursor/.../state.vscdb` (IDE) or `~/.config/cursor/auth.json` (`cursor-agent` / CLI)
- `GET https://cursor.com/api/usage-summary` (same undocumented endpoint the dashboard uses)
- Panel limits: **Cursor Models** and **Other Models**, with billing-cycle reset
- Bar label: `Cursor 1m - N%` (`1m` = monthly; `N` = highest-used pool)
- Icon `assets/cursor.svg`

### Kiro CLI (new)

Stock Omarchy **does not** include Kiro CLI. This fork adds:

- Collector `bin/kiro`: reuses the authenticated Kiro CLI session from `~/.local/share/kiro-cli/data.sqlite3`
- The same `GetUsageLimits` operation used by Kiro CLI's `/usage` panel
- Monthly credit usage and billing-cycle reset in the panel
- Bar label: `Kiro 1m - N%`
- Official Kiro mark under `assets/kiro.svg`

### Inline bar usage (new)

The stock widget shows only the bar glyph. On the horizontal bar, this fork shows:

- Active provider logo (SVG under `assets/`)
- Short window label and percent:
  - **Claude / Codex**: short 5-hour rolling window (`5h - N%`)
  - **Cursor / Kiro**: monthly billing cycle (`1m - N%`)
- Tooltip with percent, window, and time until reset
- Alarm styling (`active`) when the headline limit is ≥ 90%

### Codex fix (new)

The packaged Omarchy 4.0.1 collector still invokes Codex with `-a untrusted`. Recent Codex CLI versions renamed that approval value to `never` and reject the old one. This fork prepends `bin/codex` to `PATH` before the updater runs: it only translates `untrusted` → `never`; every other argument passes through unchanged.

### `usage-update` wrapper (new)

`Main.qml` calls `~/.config/omarchy/plugins/rafa.agents/bin/usage-update` instead of `omarchy-agent-usage-update`:

1. Puts the plugin `bin/` on `PATH` (Codex shim)
2. Runs the stock Omarchy updater (Claude, Codex, Fireworks)
3. Runs plugin-only collectors (`cursor`, `kiro`)

You keep official collectors updated by `omarchy update` without editing `/usr/share/omarchy/`, while adding providers only in the fork.

---

## Panel

- **Hero**: mark, tool name, and plan ("Max 20x", "Pro", "Ultra").
- **Subscription switch**: one chip per enabled agent (`h`/`l` or click); shown only when more than one is active.
- **Limits**: percent used, meter, and time until reset (session, weekly, or monthly).
- **Balance**: prepaid agents (Fireworks): remaining credit and spent vs. funded.
- **Tokens by day**: last 7 days; today highlighted at the bottom.
- **Tokens by model**: breakdown with bars scaled to the heaviest model.

A provider appears only when enabled in settings and has data (locally or via sync). With nothing to show, the module leaves the bar entirely. To drop the stock widget and use only this fork:

```bash
omarchy plugin disable omarchy.agents
```

## Data

Each agent is one JSON file in `~/.local/state/omarchy/agents/usage/`, written by the updater. The panel only reads those records.

| Collector | Limits | Local stats |
|---|---|---|
| `claude` | Anthropic OAuth (5h session + 7-day weekly) | `~/.claude/projects`, opencode, fallbacks |
| `codex` | Codex app-server RPC | native sessions + pi/omp/opencode |
| `fireworks` | Estimated prepaid balance | billing API (last 30 days) |
| **`cursor`** | **Monthly pools (Cursor Models + Other Models)** | **None** (plan limits only) |
| **`kiro`** | **Monthly Kiro credits** | **None** (plan limits only) |

Claude: `CLAUDE_CONFIG_DIR`. Codex: `CODEX_HOME`. Fireworks: `FIREWORKS_API_KEY`, `firectl`, opencode. Cursor: IDE or `cursor-agent`; overrides `CURSOR_DB_PATH` / `CURSOR_AGENT_AUTH_PATH`. Kiro: authenticated `kiro-cli`; override `KIRO_DB_PATH` when needed.

### Fireworks balance

Same as stock: tries `:getBalance`; on failure, estimates from `~/.config/omarchy/agents/fireworks.json` (`fundedAmount`, `fundedAt`, etc.).

## Interactions

- Bar: click = panel; right-click = agent launcher; middle-click = next provider.
- Panel: `h`/`l` switch provider, `j`/`k` scroll, `r` or Enter refresh, Tab = neighbor panel, Esc close.
- IPC: `omarchy-shell rafa.agents <open|close|toggle|refresh|next>`.

## Settings

Configured in `~/.config/omarchy/shell.json` under id `rafa.agents`:

```bash
omarchy bar set rafa.agents refreshIntervalSec 300 --json
omarchy bar set rafa.agents syncDir '~/Sync/agent-usage'
```

Per-provider enablement (includes `cursor` and `kiro`):

```bash
omarchy bar set rafa.agents providers '{
  "claude": { "enabled": true },
  "codex": { "enabled": true },
  "fireworks": { "enabled": true },
  "cursor": { "enabled": true },
  "kiro": { "enabled": true }
}' --json
```

| Key | Default | What it does |
|---|---|---|
| `refreshIntervalSec` | `900` | How often usage records regenerate |
| `syncMode` | `"Off"` | `"On"` merges snapshots from other machines |
| `syncDir` | `""` | Sync folder (Syncthing, Dropbox, …) |
| `syncFileName` | `<hostname>.json` | This machine's snapshot file |
| `syncDeviceId` | hostname | Stable device id inside aggregates |

With `syncMode` on, day and model totals are unioned by date; **rate limits are never merged** (they stay per-account).

**All-time caveat:** Codex and Fireworks cover roughly the last 30 days from their APIs; Claude uses every transcript still on disk.

## Install

```bash
git clone https://github.com/RafaelGoulartB/omarchy-ai-usage.git \
  ~/.config/omarchy/plugins/rafa.agents
```

In `shell.json`, replace `omarchy.agents` with `rafa.agents` in the bar layout (or add the id). Restart the shell:

```bash
omarchy restart shell
```

Refresh usage manually:

```bash
bash ~/.config/omarchy/plugins/rafa.agents/bin/usage-update
```

## License

MIT. Fork of Omarchy's Agents widget.
