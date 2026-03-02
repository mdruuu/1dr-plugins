# Browser-to-Plugin Prompt

A reusable prompt for building a 1dr plugin from any website you're logged into.

Usage: tell Claude "build me a 1dr plugin for [SITE URL]" — it will follow these phases automatically.

---

```
I want to build a 1dr plugin for this website: [PASTE URL]

I am logged into this site in my browser. Build me a fully working plugin that fetches
data from it programmatically, and set up the full release pipeline so it can be
distributed via the 1dr plugin registry. Do NOT ask me to open DevTools, paste curl
commands, or do anything manually — figure everything out from the URL alone.

Ask me only these two things before starting:
1. What should the binary name be? (e.g. tikr, substack — lowercase, no spaces)
2. Should this be public (anyone can install) or private (invite only)?

Then proceed through all phases automatically. Ask for help only when genuinely stuck.

---

## Phase 1 — Identify the Auth System

The goal of this phase is to name the auth system before touching browser storage.
Knowing the system tells you exactly where credentials live, how long they last,
and what refresh strategy to use. Do not scan browser storage blind.

### 1a — Identify the auth system from the JS bundle

Fetch the page and find the main JS bundle(s):

```bash
curl -s https://the-site.com | grep -oE 'src="[^"]*\.js[^"]*"' | head -20
```

Download the largest bundle(s) and search for auth fingerprints:

```bash
# Auth library imports
grep -oE "(aws-amplify|@aws-amplify|amazon-cognito-identity|firebase[/\"]auth|@auth0|okta-auth|supabase|clerk)" bundle.js | sort -u

# Auth config objects (Cognito pool IDs, Firebase config)
grep -oE '"UserPoolId"\s*:\s*"[^"]+"' bundle.js
grep -oE '"IdentityPoolId"\s*:\s*"[^"]+"' bundle.js
grep -oE '"authDomain"\s*:\s*"[^"]+"' bundle.js   # Firebase
grep -oE '"projectId"\s*:\s*"[^"]+"' bundle.js    # Firebase

# Auth endpoint URLs
grep -oE 'cognito-idp\.[a-z0-9-]+\.amazonaws\.com' bundle.js
grep -oE 'securetoken\.googleapis\.com' bundle.js
grep -oE '/oauth[2]?/token' bundle.js
grep -oE '/auth/login|/api/auth|/v[0-9]/auth' bundle.js
```

**Match findings to auth system:**

| Signal | Auth system | Tokens in | Expiry |
|--------|-------------|-----------|--------|
| `cognito-idp.*.amazonaws.com` or `UserPoolId` | AWS Cognito | localStorage | idToken 1hr, refreshToken 30d |
| `securetoken.googleapis.com` or `authDomain` | Firebase Auth | localStorage | idToken 1hr |
| `@auth0` or `auth0.com` in URLs | Auth0 | localStorage or cookies | varies |
| `/oauth2/token` with `client_id` | OAuth2 / custom | varies | varies |
| No auth library found | Custom session | cookies | varies |

**Stop here and state:** "This site uses [auth system]." Then proceed to the matching
extraction section below. Skip sections that don't apply.

---

### 1b — Extraction: AWS Cognito

Tokens are in localStorage. Read via LevelDB:

```bash
# Browsers and profiles to check
# macOS: ~/Library/Application Support/{browser}/{profile}/Local Storage/leveldb/
# Linux: ~/.config/{browser}/{profile}/Local Storage/leveldb/
# Browsers: Google/Chrome, Arc/User Data, BraveSoftware/Brave-Browser, Microsoft Edge,
#            Chromium, Vivaldi, Opera Software/Opera Stable
# Profiles: Default, Profile 1, Profile 2, Profile 3
```

Copy the leveldb dir to /tmp (browser holds a lock). Use `classic-level` to iterate:

```typescript
import { ClassicLevel } from 'classic-level';
const db = new ClassicLevel('/tmp/leveldb-copy', { valueEncoding: 'buffer' });
for await (const [key, value] of db.iterator()) {
  const k = key.toString();
  if (k.includes('CognitoIdentityServiceProvider')) {
    // strip leading \x01 byte (Chromium type prefix)
    console.log(k, value.slice(1).toString());
  }
}
```

Keys follow the pattern `CognitoIdentityServiceProvider.<ClientId>.<username>.<tokenType>`.
Extract: `idToken`, `accessToken`, `refreshToken`, `LastAuthUser`.
The `ClientId` is the string between the first and second dots — you'll need it for refresh.

**What to use in API calls:** test `idToken` first (Authorization: Bearer), then `accessToken`.

---

### 1c — Extraction: Firebase Auth

Tokens are in localStorage under `firebase:authUser:<apiKey>:<projectId>`.

Same LevelDB read approach as 1b. Filter for keys starting with `firebase:authUser`.
The value is a JSON object — extract `.stsTokenManager.accessToken` and `.stsTokenManager.refreshToken`.

**What to use in API calls:** `Authorization: Bearer <accessToken>`

---

### 1d — Extraction: Session Cookies

Cookies are the credential. No localStorage needed.

**Chromium** stores cookies as SQLite:
```
macOS: ~/Library/Application Support/{browser}/{profile}/Cookies
Linux: ~/.config/{browser}/{profile}/Cookies
Windows: %LOCALAPPDATA%/{browser}/User Data/{profile}/Network/Cookies
```

Copy to /tmp, then query:
```sql
SELECT name, value, encrypted_value, host_key
FROM cookies
WHERE host_key LIKE '%.domain.com' OR host_key = 'domain.com'
```

Chromium encrypts values with AES-128-CBC:
- macOS master key: `security find-generic-password -s "{Browser} Safe Storage" -w`
- Linux master key: usually `"peanuts"` (or from libsecret/kwallet)
- Derive: PBKDF2-SHA1(masterKey, salt=`"saltysalt"`, iterations=1003, keylen=16)
- Decrypt: strip `"v10"` prefix, IV = 16 null bytes

**Firefox** stores cookies as plaintext SQLite at:
```
macOS: ~/Library/Application Support/Firefox/Profiles/*.default*/cookies.sqlite
Linux: ~/.mozilla/firefox/*.default*/cookies.sqlite
```

Ignore analytics cookies: `_ga`, `_gcl_*`, `_gid`, `_fbp`, `_hjid`, `__utm*`.
Target: any name containing `session`, `token`, `auth`, `jwt`, `sid`, `access`, `refresh`,
or the site's own name.

**What to use in API calls:** `Cookie: name=value; name2=value2`

---

### 1e — Extraction: Unknown / Custom

If the bundle search found no known auth library, search for:
- Any JWT-shaped values in localStorage (decode with `atob` — should parse as JSON with `exp` field)
- API keys hardcoded in the bundle: `grep -oE '"apiKey"\s*:\s*"[A-Za-z0-9_-]{20,}"' bundle.js`
- Custom auth headers used in fetch calls: `grep -oE 'X-[A-Za-z-]+-[Tt]oken|X-Api-Key|X-Auth' bundle.js`

Test the candidate against any API endpoint before assuming it works.

---

### 1f — Confirm and report

Make one authenticated request to any API endpoint. Then state:

```
Auth system:   [Cognito / Firebase / session cookies / custom]
Credential:    [what you extracted — token type and first 20 chars]
Transport:     [Authorization: Bearer / Cookie: name=value / POST body field]
Expiry:        [how long the token lasts]
Refresh:       [see Phase 3 section: X]
```

**Do not proceed to Phase 2 until you have a confirmed working credential.**

---

## Phase 2 — Discover API Endpoints

Do NOT ask me to paste network requests. Read the JavaScript bundle directly.

### 2a — Find JS bundles

```bash
curl -s https://the-site.com | grep -o 'src="[^"]*\.js[^"]*"'
```

Also check for common bundle patterns: `/_next/static/`, `/_nuxt/`, `/assets/`, `/static/js/`.
Download the main app bundle(s) — skip tiny files under 10KB, skip vendor/polyfill chunks.

### 2b — Extract endpoints from the bundle

For each bundle file, search for API patterns:
- URL strings containing `/api/`, `/v1/`, `/v2/`, `/graphql`, or the site's API domain
- `fetch(`, `axios.`, `.post(`, `.get(`, `XMLHttpRequest`
- Endpoint path strings like `"/users"`, `"/data"`, `"/search"`

For each match, extract 600 chars of surrounding context to read:
- The full URL construction
- The request body being built (field names are often visible even in minified code)
- Response destructuring (reveals response shape)

Pay close attention to minified field names — they are often shortened:
- `cid` instead of `companyId`
- `tid` instead of `tradingItemId`
- `p` instead of `period`
The actual short names are what the API expects, not the long versions.

### 2c — Check for embedded third-party credentials

Search the bundles for hardcoded credentials to external services:
- Algolia: `X-Algolia-Application-Id`, `apiKey` near an app ID
- Search for: `algolia`, `typesense`, `elasticsearch`, `firebase`, `supabase`
These credentials are usually plaintext in the bundle.

### 2d — Live-test each endpoint

For each discovered endpoint, make a real authenticated request and inspect:
- What returns 200 vs 400/500 (wrong field names → 400, bad auth → 401)
- The actual response shape (don't guess — print the first result)
- Whether cid/tid-style fields need to be integers vs strings (try both if uncertain)
- Pagination — test if `page`, `offset`, `cursor`, `start`, `after` params do anything

If a field name causes a 400, extract more JS context around that endpoint —
the body construction is always within a few hundred chars of the URL string.

---

## Phase 2.5 — Discuss Before Building

Before writing any code, stop and present your findings.

**What I found:**
- List every discovered endpoint with: what it returns, what parameters it takes,
  and any interesting fields in the response
- Note any endpoints that look redundant or overlapping
- Note any data that requires chaining calls (need to call /search first to get an ID,
  then /detail with that ID)
- Note hard limits (e.g., "this endpoint is capped at 100 results with no pagination")

**What I propose to build:**
- Suggest natural CLI commands based on the data available, e.g.:
  `<name> AAPL` → snapshot view
  `<name> AAPL history` → historical data
- Flag anything technically complex vs straightforward
- Suggest a sensible default command (what should happen with no subcommand)

**Then ask the user:**
- "Which of these do you want?"
- "Any preferences on output format — tables, JSON, compact one-liners?"
- "Should I build all of these or start with the most useful subset?"

**Do not write any code until the user confirms what they want built.**

---

## Phase 3 — Handle Token Refresh and Expiry

### If the site uses AWS Cognito:

The idToken expires every hour. Refresh via:
```
POST https://cognito-idp.{region}.amazonaws.com/
X-Amz-Target: AWSCognitoIdentityProvider.InitiateAuth
{
  "AuthFlow": "REFRESH_TOKEN_AUTH",
  "AuthParameters": { "REFRESH_TOKEN": "...", "DEVICE_KEY": "..." },
  "ClientId": "..."
}
```

The ClientId is the string after `CognitoIdentityServiceProvider.` in the localStorage keys.
The region is in the User Pool ID (e.g., `us-east-1_AbCdEfGh` → region `us-east-1`).

When the refresh token itself expires (~30 days), fall back to stored credentials.
**`USER_PASSWORD_AUTH` is often disabled** on Cognito clients — test `USER_SRP_AUTH` first.
If it returns a `PASSWORD_VERIFIER` challenge, implement full SRP in pure TypeScript
using BigInt and Web Crypto — no dependencies.

### If the site uses Firebase Auth:

Refresh via: `POST https://securetoken.googleapis.com/v1/token?key={apiKey}`
with `{ "grant_type": "refresh_token", "refresh_token": "..." }`

### If the site uses session cookies:

Cookies don't usually need refresh logic — valid until the server invalidated them.
Just re-scan the browser on failure.

### Credential storage

Store everything in `~/.config/<name>-cli/` (the tool's own config dir):

```typescript
import { homedir } from 'os';
import { join } from 'path';
const CONFIG_DIR = join(homedir(), '.config', '<name>-cli');
const CONFIG = join(CONFIG_DIR, 'config.json');
```

For passwords/credentials needed for full re-auth:
- macOS: `security add-generic-password -s "<name>-cli" -a "<user>" -w "<password>"`
- Linux fallback: `${CONFIG_DIR}/.credentials` with `chmod 600`

---

## Phase 4 — Build the Plugin

### File structure

Split into multiple files for clarity — they compile to a single binary.

```
<name>-cli/
├── cli.ts        ← entry point: --summary, --help, command dispatch
├── <name>.ts     ← API calls and data fetching
├── auth.ts       ← token extraction and refresh logic
├── cache.ts      ← SQLite cache helpers
├── views.ts      ← output formatting
├── package.json
└── scripts/
    ├── release.sh
    └── install.sh
```

### CLI contract (non-negotiable)

`cli.ts` must handle `--summary` and `--help` before any other logic:

```typescript
#!/usr/bin/env bun

const NAME = '<name>';
const DESCRIPTION = '<specific description: data source, what it returns>';

const args = process.argv.slice(2);

if (args[0] === '--summary') {
  console.log(`DESCRIPTION: ${DESCRIPTION}`);
  console.log(`COMMANDS: cmd1, cmd2, cmd3`);
  process.exit(0);
}

if (args[0] === '--help' || args[0] === '-h') {
  console.log(`${NAME} — ${DESCRIPTION}\n`);
  console.log(`Usage: ${NAME} <command> [options]\n`);
  console.log('Commands:');
  // list each command with a one-line description
  process.exit(0);
}
```

### Auth error messages

Every fetch function must produce actionable errors on 401/403:

```typescript
if (!res.ok) {
  if (res.status === 401 || res.status === 403) {
    throw new Error(`Auth expired. Run: ${NAME} setup`);
  }
  const text = await res.text().catch(() => '');
  throw new Error(`API error ${res.status}: ${text.slice(0, 200)}`);
}
```

### Cache pattern (SQLite via bun:sqlite)

```typescript
import { Database } from 'bun:sqlite';
import { mkdirSync } from 'fs';

mkdirSync(CONFIG_DIR, { recursive: true });
const db = new Database(join(CONFIG_DIR, 'cache.db'));
db.run('CREATE TABLE IF NOT EXISTS cache (key TEXT PRIMARY KEY, value TEXT, expires_at INTEGER)');

function getCached<T>(key: string): T | null {
  const row = db.query('SELECT value, expires_at FROM cache WHERE key = ?').get(key) as any;
  if (!row || row.expires_at < Date.now()) return null;
  return JSON.parse(row.value);
}

function setCached(key: string, value: unknown, ttlSeconds: number): void {
  db.run('INSERT OR REPLACE INTO cache VALUES (?, ?, ?)',
    [key, JSON.stringify(value), Date.now() + ttlSeconds * 1000]);
}
```

### Output rules

- **stdout only** — all data output
- **stderr only** — all errors via `console.error()`
- **Exit codes** — 0 = success, non-zero = error
- **Format** — markdown preferred; pipe-delimited tables for structured data

---

## Phase 5 — Set Up the Release Pipeline

Do this immediately after the CLI works locally. Do not wait until later.

### 5a — One question for the user

"Does the `mdruuu/<name>-releases` GitHub repo already exist?"

If not: `gh repo create mdruuu/<name>-releases --public`

### 5b — package.json scripts block

```json
{
  "name": "<name>-cli",
  "version": "1.0.0",
  "scripts": {
    "dev": "bun run cli.ts",
    "build": "bun build ./cli.ts --outdir ./dist --target bun",
    "compile": "bun build ./cli.ts --compile --outfile <name>",
    "compile:darwin-arm64": "bun build ./cli.ts --compile --target bun-darwin-arm64 --outfile <name>-darwin-arm64",
    "compile:darwin-x64":   "bun build ./cli.ts --compile --target bun-darwin-x64   --outfile <name>-darwin-x64",
    "compile:linux-x64":    "bun build ./cli.ts --compile --target bun-linux-x64    --outfile <name>-linux-x64",
    "compile:linux-arm64":  "bun build ./cli.ts --compile --target bun-linux-arm64  --outfile <name>-linux-arm64",
    "compile:windows-x64":  "bun build ./cli.ts --compile --target bun-windows-x64  --outfile <name>-windows-x64.exe",
    "compile:all": "bun run compile:darwin-arm64 && bun run compile:darwin-x64 && bun run compile:linux-x64 && bun run compile:linux-arm64 && bun run compile:windows-x64",
    "checkpoint": "git add . && git commit -m checkpoint && npm version patch && bun run build && bun link",
    "release":       "./scripts/release.sh patch",
    "release:minor": "./scripts/release.sh minor",
    "release:major": "./scripts/release.sh major"
  }
}
```

### 5c — scripts/release.sh

Use `mdruuu/1dr-plugins` for public, `mdruuu/1dr-plugins-x` for private:

```bash
#!/bin/bash
set -e

BUMP="${1:-patch}"
NAME="<name>"
RELEASES_REPO="mdruuu/<name>-releases"
REGISTRY_REPO="mdruuu/1dr-plugins"   # or 1dr-plugins-x for private

npm version "$BUMP"
VERSION=$(node -p "require('./package.json').version")
TAG="v$VERSION"

echo ""
echo "  Building $TAG..."
echo ""

bun run compile:all

shasum -a 256 \
  ${NAME}-darwin-arm64 \
  ${NAME}-darwin-x64 \
  ${NAME}-linux-x64 \
  ${NAME}-linux-arm64 \
  ${NAME}-windows-x64.exe \
  > checksums.txt

git push && git push --tags

gh release create "$TAG" \
  ${NAME}-darwin-arm64 \
  ${NAME}-darwin-x64 \
  ${NAME}-linux-x64 \
  ${NAME}-linux-arm64 \
  ${NAME}-windows-x64.exe \
  checksums.txt \
  --repo "$RELEASES_REPO" \
  --title "$TAG" \
  --notes "${NAME} $TAG"

rm -f ${NAME}-darwin-arm64 ${NAME}-darwin-x64 ${NAME}-linux-x64 ${NAME}-linux-arm64 ${NAME}-windows-x64.exe checksums.txt

# Bump version in 1dr plugin registry
SHA=$(gh api "repos/$REGISTRY_REPO/contents/registry.json" --jq '.sha')
CURRENT=$(gh api "repos/$REGISTRY_REPO/contents/registry.json" --jq '.content' | base64 --decode)
UPDATED=$(echo "$CURRENT" | jq --arg v "$VERSION" '(.plugins[] | select(.name == "'"$NAME"'")).version = $v')
gh api "repos/$REGISTRY_REPO/contents/registry.json" \
  --method PUT \
  -f message="bump $NAME to v$VERSION" \
  -f content="$(echo "$UPDATED" | base64 | tr -d '\n')" \
  -f sha="$SHA"

echo ""
echo "  Released $TAG"
echo ""
```

`chmod +x scripts/release.sh`

### 5d — scripts/install.sh

```bash
#!/usr/bin/env bash
# <name> installer
# Usage: curl -fsSL https://raw.githubusercontent.com/mdruuu/<name>-releases/main/install.sh | bash

set -euo pipefail

REPO="mdruuu/<name>-releases"
BINARY="<name>"
INSTALL_DIR="$HOME/.local/bin"

info()    { printf "  \033[34m•\033[0m %s\n" "$*"; }
success() { printf "  \033[32m✓\033[0m %s\n" "$*"; }
warn()    { printf "  \033[33m!\033[0m %s\n" "$*"; }
error()   { printf "  \033[31m✗\033[0m %s\n" "$*" >&2; exit 1; }

OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)

case "$OS" in
  darwin) OS="darwin" ;;
  linux)  OS="linux"  ;;
  *)      error "Unsupported OS: $OS" ;;
esac

case "$ARCH" in
  x86_64)        ARCH="x64"   ;;
  arm64|aarch64) ARCH="arm64" ;;
  *)             error "Unsupported architecture: $ARCH" ;;
esac

ASSET="${BINARY}-${OS}-${ARCH}"

info "Fetching latest release..."
RELEASE_JSON=$(curl -fsSL "https://api.github.com/repos/${REPO}/releases/latest")

LATEST=$(echo "$RELEASE_JSON" | grep '"tag_name"' | head -1 | cut -d'"' -f4)
[ -z "$LATEST" ] && error "Could not determine latest release"

URL=$(echo "$RELEASE_JSON" | grep "browser_download_url" | grep "/${ASSET}\"" | sed 's/.*"browser_download_url": *"\([^"]*\)".*/\1/')
[ -z "$URL" ] && error "No asset found for ${ASSET} in ${LATEST}"

CHECKSUM_URL=$(echo "$RELEASE_JSON" | grep "browser_download_url" | grep '/checksums.txt"' | sed 's/.*"browser_download_url": *"\([^"]*\)".*/\1/' || true)

info "Latest: $LATEST"

TMP=$(mktemp)
info "Downloading ${ASSET}..."
curl -fsSL "$URL" -o "$TMP"

if [ -n "$CHECKSUM_URL" ]; then
  CHECKSUMS=$(curl -fsSL "$CHECKSUM_URL")
  EXPECTED=$(echo "$CHECKSUMS" | grep " ${ASSET}$" | awk '{print $1}')
  if [ -n "$EXPECTED" ]; then
    if command -v shasum &>/dev/null; then
      ACTUAL=$(shasum -a 256 "$TMP" | awk '{print $1}')
    else
      ACTUAL=$(sha256sum "$TMP" | awk '{print $1}')
    fi
    if [ "$ACTUAL" != "$EXPECTED" ]; then
      rm -f "$TMP"
      error "Checksum mismatch for ${ASSET}"
    fi
  else
    warn "No checksum entry for ${ASSET}; skipping verification"
  fi
else
  warn "No checksums.txt in release; skipping verification"
fi

mkdir -p "$INSTALL_DIR"
mv "$TMP" "$INSTALL_DIR/$BINARY"
chmod +x "$INSTALL_DIR/$BINARY"

success "Installed $BINARY to $INSTALL_DIR"

if [[ ":$PATH:" != *":$INSTALL_DIR:"* ]]; then
  warn "'$INSTALL_DIR' is not in your PATH."
  warn "Add to your shell profile: export PATH=\"\$HOME/.local/bin:\$PATH\""
fi

printf "\n"
printf "  \033[32m%s installed!\033[0m\n\n" "$BINARY"
printf "  Run: %s --help\n\n" "$BINARY"
```

`chmod +x scripts/install.sh`

Push `install.sh` to the releases repo root so the install URL works:
```bash
gh api repos/mdruuu/<name>-releases/contents/install.sh \
  --method PUT \
  -f message="add installer" \
  -f content="$(cat scripts/install.sh | base64 | tr -d '\n')" \
  --jq '.commit.sha'
```

### 5e — Add the registry entry

```bash
REGISTRY_REPO="mdruuu/1dr-plugins"   # or 1dr-plugins-x for private
SHA=$(gh api "repos/$REGISTRY_REPO/contents/registry.json" --jq '.sha')
CURRENT=$(gh api "repos/$REGISTRY_REPO/contents/registry.json" --jq '.content' | base64 --decode)
NEW_ENTRY=$(jq -n \
  --arg name "<name>" \
  --arg desc "<one-line description matching --summary DESCRIPTION>" \
  --arg install "curl -fsSL https://raw.githubusercontent.com/mdruuu/<name>-releases/main/install.sh | bash" \
  --arg homepage "https://github.com/mdruuu/<name>-releases" \
  '{name: $name, description: $desc, install: $install, homepage: $homepage, feedback: "hello@actuallyprettyuseful.com", version: "1.0.0"}')
UPDATED=$(echo "$CURRENT" | jq --argjson entry "$NEW_ENTRY" '.plugins += [$entry]')
gh api "repos/$REGISTRY_REPO/contents/registry.json" \
  --method PUT \
  -f message="add <name> plugin" \
  -f content="$(echo "$UPDATED" | base64 | tr -d '\n')" \
  -f sha="$SHA"
```

### 5f — First release

```bash
git init && git add . && git commit -m "initial"
gh repo create mdruuu/<name>-cli --private   # or --public
git remote add origin git@github.com:mdruuu/<name>-cli.git
git push -u origin main

./scripts/release.sh patch
```

### 5g — Verify end to end

```bash
1dr plugin install <name>            # public
1dr plugin install <private-url>     # private
<name> --help
<name> --summary
```

---

## If You Get Stuck

- **400 on an endpoint**: extract more JS context around that URL string —
  the body construction is always within a few hundred chars of the URL string.
- **Token not working**: check if there are multiple token types (idToken vs accessToken)
- **Can't find endpoints in main bundle**: site may use code splitting —
  check for route-specific JS chunks, look for dynamic `import()` calls
- **Cookie decryption fails on Linux**: try `"peanuts"` as the hardcoded password
- **gh release create fails**: check the releases repo exists and you have push access

When you do need to ask the user, be specific:
- "I found two tokens — `accessToken` and `idToken`. Can you check DevTools → Network
  → click any API request → Headers, and tell me which token appears? Just the header
  name and first 20 chars of the value."

Start with Phase 1 now. Scan localStorage and cookies, show every auth-looking value,
confirm which works against the API, then proceed automatically through all phases.
```
