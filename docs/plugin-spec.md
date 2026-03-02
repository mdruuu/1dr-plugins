# 1dr Plugin Authoring Spec

Copy-paste this into an AI agent prompt or `CLAUDE.md` when building a plugin.

---

## What is a 1dr plugin?

A standalone CLI tool. **1dr does not invoke it** — 1dr only reads its `--summary` output to include it in the skill file so AI agents know it exists and how to call it directly.

---

## Required flags

### `--summary`

Machine-readable metadata. Must print exactly two lines to stdout and exit 0:

```
DESCRIPTION: <one-liner — data source, format, what it does>
COMMANDS: cmd1, cmd2, cmd3
```

- `DESCRIPTION`: one sentence. Mention the data source and what it returns.
- `COMMANDS`: comma-separated list of all subcommands.

Example:

```
DESCRIPTION: Fetches TIKR transcripts and earnings data for a given ticker or CID.
COMMANDS: transcript, earnings, search
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

## Release requirements

Every plugin must ship pre-compiled binaries via a versioned GitHub release.

### Platforms

At minimum: `darwin-arm64`, `darwin-x64`, `linux-x64`, `linux-arm64`

### Release checklist

- [ ] Tagged release (`vX.Y.Z`) — use `npm version patch|minor|major`
- [ ] Pre-compiled binaries for all platforms (`bun build --compile`)
- [ ] `checksums.txt` (SHA256) in every release
- [ ] Installer script (`scripts/install.sh`) that verifies checksums before installing
- [ ] Installer detects OS/arch and picks the right binary
- [ ] Release script bumps the `version` field in the registry JSON after publishing

### Registry version bump (add to end of `scripts/release.sh`)

```bash
REGISTRY_REPO="mdruuu/1dr-plugins"       # or 1dr-plugins-x for private
REGISTRY_FILE="registry.json"
SHA=$(gh api "repos/$REGISTRY_REPO/contents/$REGISTRY_FILE" --jq '.sha')
CURRENT=$(gh api "repos/$REGISTRY_REPO/contents/$REGISTRY_FILE" --jq '.content' | base64 --decode)
UPDATED=$(echo "$CURRENT" | jq --arg v "$VERSION" '(.plugins[] | select(.name == "your-plugin")).version = $v')
gh api "repos/$REGISTRY_REPO/contents/$REGISTRY_FILE" \
  --method PUT \
  -f message="bump your-plugin to v$VERSION" \
  -f content="$(echo "$UPDATED" | base64 | tr -d '\n')" \
  -f sha="$SHA"
```

---

## Registry entry

Add an entry to `mdruuu/1dr-plugins/registry.json` (public) or `mdruuu/1dr-plugins-x/registry.json` (private).

```json
{
  "name": "your-plugin",
  "description": "One-line description matching --summary DESCRIPTION",
  "install": "curl -fsSL https://raw.githubusercontent.com/mdruuu/your-releases/main/install.sh | bash",
  "homepage": "https://github.com/mdruuu/your-releases",
  "feedback": "hello@actuallyprettyuseful.com",
  "version": "1.0.0"
}
```

The `version` field is compared against `installedVersion` stored at install time. **Bump it on every release** — this is what `1dr plugin update` uses to detect new versions.

**Installing from registry:**
```bash
1dr plugin install your-plugin          # from public registry
1dr plugin install <private-registry-url>   # from private registry (URL itself is the access gate)
```

---

## Auth

Each plugin handles its own auth independently. Common patterns:

- **Browser cookie extraction**: read from `~/Library/Application Support/...` or use `get-cookies-txt`
- **Keychain**: use the OS keychain via `security find-generic-password`
- **Config file**: store in `~/.1dr/plugins/<name>/config.json`

1dr has no auth integration — don't depend on it.

---

## Config and cache storage

Store all plugin data under `~/.1dr/plugins/<name>/`. Do not write to `~/.1dr/` root.

```typescript
import { homedir } from 'os';
import { join } from 'path';

const PLUGIN_DIR = join(homedir(), '.1dr', 'plugins', NAME);
const CONFIG = join(PLUGIN_DIR, 'config.json');
const CACHE_DB = join(PLUGIN_DIR, 'cache.db');
```

---

## Standard error messages

Use consistent phrasing so AI agents can recognize and handle errors:

| Condition | Message pattern |
|-----------|----------------|
| Auth expired | `Auth expired. Run: <plugin> login` |
| Rate limit hit | `Rate limited. Try again in <N> seconds.` |
| Not found | `Not found: <the-input-that-failed>` |

---

## Testing before release

```bash
# Test --summary and --help locally
bun run cli.ts --summary
bun run cli.ts --help
bun run cli.ts <command>

# Link the binary for manual testing
bun link   # makes 'your-plugin' available in PATH

# Release
./scripts/release.sh patch
# This: bumps version, compiles all platforms, creates GitHub release, bumps registry version
```

---

## Checklist

Before publishing a plugin:

- [ ] `--summary` prints exactly `DESCRIPTION:` and `COMMANDS:` lines, exits 0
- [ ] `--help` / `-h` lists every command with descriptions
- [ ] stdout = data only, stderr = errors only
- [ ] Exit 0 on success, 1+ on error
- [ ] Tagged releases with per-platform binaries and `checksums.txt`
- [ ] Installer verifies checksums before installing
- [ ] Registry entry has `name`, `description`, `install`, `version`
- [ ] Release script bumps `version` in registry on each release
- [ ] Auth failure prints actionable message to stderr
- [ ] Config/cache stored in `~/.1dr/plugins/<name>/`
