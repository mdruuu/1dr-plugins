# 1dr Plugin Authoring Spec

Copy-paste this into an AI agent prompt or `CLAUDE.md` when building a plugin.

---

## What is a 1dr plugin?

A standalone CLI tool. **1dr does not invoke it** — 1dr only reads its `--summary` output to include it in the skill file so AI agents know it exists and how to call it directly.

---

## Required flags

### `--summary`

Machine-readable metadata. Must print exactly three lines to stdout and exit 0:

```
DESCRIPTION: <one-liner — data source, format, what it does>
COMMANDS: cmd1, cmd2, cmd3
REPLACES: <filings|price|earnings|screen|doc>
```

- `DESCRIPTION`: one sentence. Mention the data source and what it returns.
- `COMMANDS`: comma-separated list of all subcommands.
- `REPLACES`: only include this line if the plugin replaces a 1dr built-in command. Omit the line entirely otherwise.

Example:

```
DESCRIPTION: Fetches TIKR transcripts and earnings data for a given ticker or CID.
COMMANDS: transcript, earnings, search
REPLACES: earnings
```

### `--help` / `-h`

Standard usage help. List every command with a one-line description. Users and AI agents rely on this to know what to call.

---

## stdio contract

| Stream | Purpose |
|--------|---------|
| stdout | Data output (markdown preferred) |
| stderr | Errors and warnings only |

Never mix errors into stdout. AI agents parse stdout.

---

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1+ | Error |

---

## Output format

- **Markdown preferred** for prose and reports
- **Tables** for structured data (aligned, pipe-delimited)
- **`--json` flag** optional but encouraged for machine-readable output

---

## Install patterns

The `install` field in `registry.json` accepts any shell command. Two common patterns:

**Script (npm/bun package):**
```json
{
  "name": "my-plugin",
  "description": "Does something useful",
  "install": "bun install -g my-plugin-package"
}
```

**Binary (curl install — mirrors how 1dr itself installs):**
```json
{
  "name": "tikr",
  "description": "TIKR transcript and earnings data",
  "install": "curl -fsSL https://releases.example.com/tikr/install.sh | bash"
}
```

Both work today. `1dr plugin install <name>` splits and spawns whatever is in the `install` field with inherited stdio.

---

## Registration

**From registry** (preferred):
```bash
1dr plugin install <name>
```

**Manual add** (local/private plugins):
```bash
1dr plugin add <binary-name> --description "Does something useful"
```

After install, run `1dr skill-sync` to regenerate the Claude Code skill file.

---

## Auth

Each plugin handles its own auth independently. Common patterns:

- **Browser cookie extraction**: read from `~/Library/Application Support/...` or use `get-cookies-txt`
- **Keychain**: use the OS keychain via `security find-generic-password`
- **Config file**: store in `~/.config/<plugin-name>/config.json`

1dr has no auth integration — don't depend on it.

---

## Standard error messages

Use consistent phrasing so AI agents can recognize and handle errors:

| Condition | Message pattern |
|-----------|----------------|
| Auth expired | `Auth expired. Run: <plugin> login` |
| Rate limit hit | `Rate limited. Try again in <N> seconds.` |
| Not found | `Not found: <the-input-that-failed>` |

---

## Checklist

Before publishing a plugin:

- [ ] `--summary` prints exactly `DESCRIPTION:`, `COMMANDS:`, and (if replacing a built-in) `REPLACES:` lines
- [ ] `--help` / `-h` lists every command with descriptions
- [ ] stdout = data only, stderr = errors only
- [ ] Exit 0 on success, 1+ on error
- [ ] `install` command in registry.json is copy-pasteable and idempotent
- [ ] Auth failure prints actionable message to stderr
