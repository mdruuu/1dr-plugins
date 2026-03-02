# Browser-to-Plugin Prompt

A reusable prompt for building a 1dr plugin from any website you're logged into.
The result is a single `cli.ts` that plugs into `1dr` via the plugin registry.

Usage: tell Claude "build me a 1dr plugin for [SITE URL]" — it will follow these phases automatically.

---

```
I want to build a 1dr plugin for this website: [PASTE URL]

I am logged into this site in my browser. Build me a fully working plugin that fetches
data from it programmatically. Do NOT ask me to open DevTools, paste curl commands,
or do anything manually — figure everything out from the URL alone.

---

## Phase 1 — Extract Auth Token

Scan both localStorage and cookies across all browsers and profiles on this machine.

### 1a — localStorage (LevelDB)

Chromium stores localStorage as LevelDB at:
- macOS: `~/Library/Application Support/{browser}/{profile}/Local Storage/leveldb/`
- Linux: `~/.config/{browser}/{profile}/Local Storage/leveldb/`
- Windows: `%LOCALAPPDATA%/{browser}/User Data/{profile}/Local Storage/leveldb/`

Copy the directory to /tmp before reading (browser locks it while open).
Use the `classic-level` npm package to iterate all entries.
Filter for keys matching the exact origin: `_https://the-site.com\x00\x01*`
Strip the leading `\x01` byte from every value (Chromium's internal type prefix).

Look for values that are:
- JWTs (start with `eyJ` in base64)
- Long opaque strings that look like tokens (>32 chars)
- UUIDs used as session identifiers

Note the key names — they reveal the auth scheme:
- `CognitoIdentityServiceProvider.*` → AWS Cognito
- `firebase:authUser:*` → Firebase Auth
- Keys containing `token`, `auth`, `session`, `access`, `refresh`, `id`

### 1b — Cookies (SQLite)

Chromium stores cookies at:
- macOS: `~/Library/Application Support/{browser}/{profile}/Cookies`
- Linux: `~/.config/{browser}/{profile}/Cookies`
- Windows: `%LOCALAPPDATA%/{browser}/User Data/{profile}/Network/Cookies`

Firefox stores cookies at:
- macOS: `~/Library/Application Support/Firefox/Profiles/*default*/cookies.sqlite`
- Linux: `~/.mozilla/firefox/*default*/cookies.sqlite`
- Windows: `%APPDATA%/Mozilla/Firefox/Profiles/*default*/cookies.sqlite`

Copy to /tmp before reading. Query:
```sql
SELECT name, value, encrypted_value, host_key
FROM cookies
WHERE host_key LIKE '%.domain.com' OR host_key = 'domain.com'
```

For Chromium browsers, values are AES-128-CBC encrypted:
- macOS: get master password via `security find-generic-password -s "{Browser} Safe Storage" -w`
- Linux: password is usually `"peanuts"` (default) or from libsecret
- Derive key: PBKDF2-SHA1(password, salt=`"saltysalt"`, iterations=1003, keylen=16)
- Decrypt: strip the `"v10"` prefix, use 16 null bytes as IV

Firefox cookies are plaintext — no decryption needed.

Ignore pure analytics cookies: `_ga`, `_gcl_*`, `_gid`, `_fbp`, `_hjid`, `__utm*`.
Focus on session-looking names: anything containing `session`, `token`, `auth`, `jwt`,
`sid`, `access`, `refresh`, or the site's own name.

### 1c — Browsers and profiles to scan

Check these browser application dirs (skip if not installed):
- `Google/Chrome`, `Google/Chrome Beta`, `Google/Chrome Canary`
- `Microsoft Edge`, `Microsoft Edge Beta`
- `Arc/User Data`
- `BraveSoftware/Brave-Browser`
- `Chromium`
- `Vivaldi`
- `Opera Software/Opera Stable`
- Firefox (all profiles matching `*.default*` or `*.default-release*`)

For each Chromium browser, scan these profiles: `Default`, `Profile 1`, `Profile 2`,
`Profile 3` — stop at first missing one.

### 1d — Identify auth scheme and test it

From what you found, determine how the site sends auth to its API.
Try these in order until one gets a non-401 response from any API endpoint:

1. JWT in POST body: `{ "auth": token }` or `{ "token": token }` or `{ "jwt": token }`
2. Authorization header: `Authorization: Bearer <token>`
3. Cookie header: `Cookie: name=value; name2=value2`
4. Custom header (look for hints in key names): `X-Auth-Token`, `X-API-Key`, etc.

To find an API endpoint to test against, look in the page HTML for any fetch/XHR
call or API base URL — even a partial one is enough to probe.

**Show me what you found in each store and which auth method works before proceeding.**

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
  `1dr <name> AAPL` → snapshot view
  `1dr <name> AAPL history` → historical data
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

Cookies don't usually need refresh logic — valid until the server invalidates them.
Just re-scan the browser on failure.

### Credential storage

Store in `~/.1dr/plugins/<name>/config.json` (never in `~/.1dr/` root):

```typescript
import { homedir } from 'os';
import { join } from 'path';
const PLUGIN_DIR = join(homedir(), '.1dr', 'plugins', NAME);
const CONFIG = join(PLUGIN_DIR, 'config.json');
```

For passwords/credentials needed for full re-auth:
- macOS: `security add-generic-password -s "<name>-plugin" -a "<user>" -w "<password>"`
- Linux fallback: `${PLUGIN_DIR}/.credentials` with `chmod 600`

---

## Phase 4 — Build the Plugin

This is a 1dr plugin — a single `cli.ts` that runs via `#!/usr/bin/env bun`.
Do NOT build a multi-file CLI. Everything goes in one file.

### Plugin contract (non-negotiable)

```typescript
#!/usr/bin/env bun

import { homedir } from 'os';
import { join } from 'path';

const NAME = '<name>';
const DESCRIPTION = '<specific description: data, source, format>';
const PLUGIN_DIR = join(homedir(), '.1dr', 'plugins', NAME);

const handlers: Record<string, (args: string[]) => Promise<void>> = {
  // add your commands here
};

const args = process.argv.slice(2);
const command = args[0];

if (command === '--summary') {
  console.log(`DESCRIPTION: ${DESCRIPTION}`);
  console.log(`COMMANDS: ${Object.keys(handlers).join(', ')}`);
  process.exit(0);
}

if (command === '--help' || command === '-h' || !command) {
  console.log(`${NAME} — ${DESCRIPTION}\n`);
  console.log(`Usage: ${NAME} <command> [options]\n`);
  console.log('Commands:');
  Object.keys(handlers).forEach(c => console.log(`  ${c}`));
  process.exit(!command ? 1 : 0);
}

const handler = handlers[command];
if (handler) {
  await handler(args.slice(1));
} else {
  console.error(`Unknown command: ${command}`);
  process.exit(1);
}
```

### Auth pattern (3-tier, inside cli.ts)

```typescript
const CONFIG = join(PLUGIN_DIR, 'config.json');

async function getToken(): Promise<string> {
  // 1. Check cache
  try {
    const cached = JSON.parse(await Bun.file(CONFIG).text());
    if (cached.token && cached.expiresAt > Date.now()) return cached.token;
    // 2. Refresh if possible
    if (cached.refreshToken) {
      const fresh = await refreshToken(cached.refreshToken);
      await saveConfig(fresh);
      return fresh.token;
    }
  } catch {}
  // 3. Full re-auth from browser scan or stored credentials
  const found = await extractFromBrowser();
  await saveConfig(found);
  return found.token;
}
```

### Cache pattern (SQLite via bun:sqlite)

```typescript
import { Database } from 'bun:sqlite';
import { mkdirSync } from 'fs';

mkdirSync(PLUGIN_DIR, { recursive: true });
const db = new Database(join(PLUGIN_DIR, 'cache.db'));
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

- **stdout only** — all data output goes to stdout
- **stderr only** — all errors go to `console.error()`
- **Exit codes** — 0 = success, non-zero = error
- **Format** — markdown preferred; JSON only if the agent will parse it programmatically
- **Never** write to `~/.1dr/` root; only to `~/.1dr/plugins/<name>/`

**Preferred output format:**
```markdown
# Apple Inc (AAPL) — Estimates

| Metric  | FY2025E | FY2026E |
|---------|---------|---------|
| Revenue | $409.0B | $432.5B |
| EPS     | $7.42   | $8.01   |
```

### Error handling

- 401/403 → `"Not authenticated. Run: <name> login"`
- 404 → `"Not found: <id>"`
- 429 → `"Rate limited — try again in Xs"`
- Network error → `"Could not reach <service>. Check your connection."`
- Parsing failure → `"Parsing failed — site layout may have changed."`

---

## Phase 5 — Test and Release

```bash
# Test locally before releasing
bun run cli.ts --summary      # verify metadata format
bun run cli.ts --help         # verify help text
bun run cli.ts <command>      # test with real data

# Link for manual binary testing
bun link   # makes 'your-plugin' available in PATH

# Release (compiles all platforms, creates GitHub release, bumps registry version)
./scripts/release.sh patch
```

After release, users install via:
```bash
1dr plugin install your-plugin          # public registry
1dr plugin install <private-url>        # private registry
```

The skill file updates automatically on next interactive startup.

---

## If You Get Stuck

- **400 on an endpoint**: extract more JS context around that URL string —
  the body construction is always within a few hundred chars of the URL string.
- **Token not working**: check if there are multiple token types (idToken vs accessToken)
- **Can't find endpoints in main bundle**: site may use code splitting —
  check for route-specific JS chunks, look for dynamic `import()` calls
- **Cookie decryption fails on Linux**: try `"peanuts"` as the hardcoded password

When you do need to ask the user, be specific:
- "I found two tokens — `accessToken` and `idToken`. Can you check DevTools → Network
  → click any API request → Headers, and tell me which token appears? Just the header
  name and first 20 chars of the value."

Start with Phase 1 now. Scan localStorage and cookies, show every auth-looking value,
confirm which works against the API, then proceed automatically.
```
