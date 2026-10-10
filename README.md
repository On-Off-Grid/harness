<div align="center">

# 🎯 pi-hunt

### Autonomous bug-bounty hunting harness for the [pi](https://pi.dev) coding agent

*A coordinator + 25 specialist sub-agents, direct OpenRouter model access, local MCP servers, a vectorized knowledge base, and cost/recon optimizations — wired together for end-to-end, evidence-first vulnerability research.*

![pi](https://img.shields.io/badge/agent-pi.dev-6b46c1)
![model](https://img.shields.io/badge/model-Claude%20Opus%20%2F%20Sonnet-d97757)
![platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS-1f2937)
![shell](https://img.shields.io/badge/shell-bash-4eaa25?logo=gnubash&logoColor=white)
![python](https://img.shields.io/badge/python-3.11%2B-3776ab?logo=python&logoColor=white)
![node](https://img.shields.io/badge/node-22%2B-339933?logo=nodedotjs&logoColor=white)
![license](https://img.shields.io/badge/license-MIT-green)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen)

`bug-bounty` · `offensive-security` · `web-security` · `recon` · `llm-agents` · `rag` · `mcp` · `automation`

</div>

---

> [!WARNING]
> **Authorized testing only.** pi-hunt is an autonomous offensive-security tool. Use it **exclusively** against targets you own or are explicitly authorized to test under a bug-bounty program's scope and rules. You are responsible for staying in scope and obeying program policy and the law. The maintainers accept no liability for misuse.

---

## Table of contents

- [Why pi-hunt](#why-pi-hunt)
- [Quick start](#quick-start)
- [Post-install (one-time)](#post-install-one-time)
- [Usage](#usage)
- [Architecture](#architecture)
- [What's included](#whats-included)
- [Optimization layers](#optimization-layers)
- [Configuration](#configuration)
- [Requirements](#requirements)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

---

## Why pi-hunt

Most "AI hacking" setups are a single chat window. pi-hunt is a **team**: a coordinator plans the hunt, routes each surface to the right specialist, runs them in parallel, chains low-severity signals into criticals, and validates every candidate through a strict triager before it's written up — all locally, with full evidence trails.

| | |
|---|---|
| 🧠 **Coordinator + 23 specialists** | One brain delegates to focused hunters (SSRF, OAuth/ATO, RCE, IDOR/BOLA, race, cache, smuggling, XSS, SQLi, LLM/prompt-injection, recon, triage…). |
| 🔌 **6 MCP servers** | Browser (Chrome DevTools), Burp, Memory graph, GitHub, Medium, Gmail — plus a built-in **RAG** server. |
| 📚 **Vectorized knowledge base** | Semantic recall over a curated techniques KB, your past findings, the memory graph, and learned heuristics. |
| 💸 **Cost-tiered models** | Mechanical agents run on a cheaper model; reasoning/exploit/validation agents stay on the flagship. One toggle. |
| ⚡ **Smarter recon commands** | Auto-injects quality SecLists wordlists into `ffuf`/`gobuster`/etc. — without slowing down everyday commands. |
| 🛡️ **Evidence-first** | Every finding gets a self-contained, copy-paste-reproducible report with negative controls before it's kept. |

---

## Quick start

> [!IMPORTANT]
> The default installer deploys the direct OpenRouter runtime. Configure the
> OpenRouter credential locally after installation; never commit it.

### Option A — one command (recommended)

```bash
git clone https://github.com/<user>/pi-hunt.git && cd pi-hunt
./bootstrap.sh --with-extras
```

`bootstrap.sh` installs missing prerequisites (Node 22+, Python, rsync, git), runs the direct OpenRouter framework installer, and (with `--with-extras`) the recon toolchain + SecLists.

### Option B — Make

```bash
make dry      # preview everything, change nothing
make full     # framework + recon CLIs & SecLists (needs sudo)
make install  # framework only
```

### Option C — direct

```bash
./install.sh --dry-run    # preview
./install.sh              # framework only (prompts for the GitHub MCP token)
./extras.sh               # optional: recon CLIs + SecLists (needs sudo)
```

> [!TIP]
> Everything is **idempotent and reversible** — re-running backs up any file it replaces to `<file>.bak-<timestamp>`. Start with `--dry-run` to see every action first.

---

## Post-install (one-time)

These are per-machine secrets the installer can't carry for you (it reminds you too):

```bash
# 1. Configure the OpenRouter credential locally in the installed Pi model registry.
#    Keep credentials out of this repository.

# 2. GitHub MCP token (or re-run install with it):
GITHUB_MCP_TOKEN=ghp_xxx ./install.sh

# 3. Gmail MCP OAuth (optional, for inbox/OTP flows):
mkdir -p ~/.gmail-mcp   # drop gcp-oauth.keys.json + credentials.json here
npx @gongrzhe/server-gmail-autoauth-mcp auth

# 4. Burp (optional): install Burp Suite + the AI-Agent extension,
#    enable its MCP server on 127.0.0.1:9876, and add the proxy jar
#    (see mcp-servers/burp-mcp-proxy/README.md).
```

Make sure `~/.local/bin` is on your `PATH`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

---

## Usage

```bash
pi-hunt                                   # interactive — then type a hunt command
pi-hunt "Hunt acme for CRITICAL"          # one-shot
```

Inside a session:

```
Hunt <program> in HackerOne for CRITICAL
Hunt <program> in Intigriti for HIGH
Hunt <program> in YesWeHack for HIGH
```

The coordinator loads scope, maps the attack surface, routes surfaces to specialists, chains findings, validates them, and writes reports to `reports/<program>/…`.

Handy toggles:

```bash
pi-tier on        # cost tiering: mechanical agents -> cheaper model
pi-tier status    # show per-agent models
pi-4.8 on|off     # flip the flagship between Opus 4.8 / 4.7
pihunt-rag search "ssrf to cloud metadata"     # query the knowledge base
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         pi (coding agent)                          │
│                                                                    │
│   coordinator  ──delegates──▶  23 specialist sub-agents            │
│       │                         (ssrf, oauth, rce, idor, race,     │
│       │                          cache, smuggling, xss, sqli,      │
│       │                          llm, recon, triager, …)           │
│       │                                                            │
│   extensions:  subagent (ensemble/swarm) · cmd-optimizer ·         │
│                mcp-bridge · auto-continue · compact-on-46          │
└───────┬───────────────────────────────────────────────┬──────────┘
        │ Anthropic API                                  │ MCP
        ▼                                                ▼
┌────────────────┐                        ┌──────────────────────────┐
│  OpenRouter     │  Claude Opus/Sonnet    │  MCP servers              │
│  Messages API   │◀──────────────────────│  browser-live · burp ·    │
│  (direct)       │                        │  memory · github ·        │
└────────────────┘                        │  medium · gmail ·         │
                                           │  pihunt-rag (knowledge)   │
                                           └──────────────────────────┘
```

pi calls OpenRouter directly through its Anthropic-compatible Messages API. MCP servers,
browser automation, Burp, memory/RAG, and subagent processes remain local to `pi`.
`kiro-gateway/` is retained as legacy/reference-only material and is not on the
default runtime path.

---

## What's included

| Path | Installs to | What it is |
|---|---|---|
| `bin/` | `~/.local/bin/` | `pi-hunt` (direct OpenRouter) · `pi-4.8` (flagship toggle) · `pi-tier` (cost tiering); `pi-kiro` is legacy-only |
| `pi-agent/` | `~/.pi/agent/` | `models.json`, `settings.json`, **24 agents**, extensions (subagent ensemble/swarm, mcp-bridge, auto-continue, compact-on-46, **cmd-optimizer**, session-memory, **continuous-learning**), coordinator prompt |
| `rag/` | `~/.pi-hunt-rag/` + `~/opencode/pihunt-rag-mcp/` | **RAG + knowledge graph** — vectorized search, techniques KB, MCP server ([details](rag/README.md)) |
| `kiro-gateway/` | — | Legacy Anthropic↔Kiro bridge retained for historical/reference use only |
| `kiro-settings/mcp.json` | `~/.kiro/settings/mcp.json` | MCP config (token is a `${GITHUB_MCP_TOKEN}` placeholder) |
| `mcp-servers/` | various | browser-live, chrome-devtools, medium-rss, burp-proxy, pihunt-rag server code |
| `skills/` | `~/.kiro/skills/` | 7 skills: auth-attack-patterns, chain-patterns, critical-endpoints, framework-exploits, recon-checklist, writeup-intel, business-impact |
| `steering/`, `prompts/` | `~/.kiro/…`, `~/.config/opencode/…` | workflow rules + the coordinator brain |
| `systemd/` | `~/.config/systemd/user/` | gateway auto-start + self-heal |
| `extras.sh` | — | optional recon toolchain installer (httpx, katana, dnsx, naabu, subfinder, nuclei, ffuf, sqlmap, gau, git-dumper, **SecLists**, …) |

### Bundled pi packages

Hunt-relevant packages from the [pi package catalog](https://pi.dev/packages), installed automatically by `install.sh` (and pinned in `pi-agent/settings.json`). Manage with `pi list` / `pi update --extensions` / `pi remove npm:<pkg>`.

| Package | Type | What it adds |
|---|---|---|
| [`pi-casefile`](https://pi.dev/packages/pi-casefile) | extension | Offensive-security **case tracker** — log bug-bounty/CTF/audit findings as you hunt |
| [`pi-recon-tools`](https://pi.dev/packages/pi-recon-tools) | tools | Recon toolkit: headers audit, SSL check, DNS, port scan, tech fingerprint, path discovery (`nmap`/`openssl`/`dig`) |
| [`@artale/pi-pentest`](https://pi.dev/packages/@artale/pi-pentest) | tools | `nmap` wrapper + vulnerability checks |
| [`pi-redteam`](https://pi.dev/packages/pi-redteam) | ext+skills | 10 specialist security agents for vuln assessment & pentest |
| [`@vigolium/piolium`](https://pi.dev/packages/@vigolium/piolium) | ext+skills | Multi-phase security audits with isolated-context specialist sub-agents |
| [`pi-web-access`](https://pi.dev/packages/pi-web-access) | ext+skills | Zero-config web search/fetch/scrape/PDF/GitHub-clone (Exa MCP, no key; optional provider keys) |

> These run with full system access (like any pi package). The recon/pentest ones shell out to standard CLIs (`nmap`, `openssl`, `dig`) that `extras.sh` installs. `pi-web-access` works out of the box; add keys in `~/.pi/web-search.json` for more providers.

---

## Optimization layers

Three layers make hunts cheaper and sharper without dulling the agents that find bugs.

### 1. Model tiering — `pi-tier`

Mechanical agents run on a cheaper **fast** model; reasoning/exploit/validation agents stay on the **flagship**.

| Tier | Agents | Default model |
|---|---|---|
| ⚙️ helper (mechanical) | scope-loaders, recon, test-account-manager, visual-security, payload-researcher, wordlist-curator | `anthropic/claude-sonnet-5` |
| 🧠 flagship (reasoning) | remaining specialists | `anthropic/claude-opus-4.8` |

Large-context compaction uses `anthropic/claude-sonnet-4.6` as its conservative summarizer.

```bash
pi-tier on      # apply        pi-tier status              # show
pi-tier off     # reset        pi-tier set-fast <model>    # change fast model
```

Composes with `pi-4.8` (which flips the flagship) — tiering tracks whichever flagship is active.

### 2. Command optimization — `cmd-optimizer` extension

A `tool_call` hook that auto-injects mode-aware **SecLists** wordlists + benign flags into discovery tools (`ffuf`, `feroxbuster`, `gobuster`, `dirsearch`, `wfuzz`, `dnsx`) **only when no wordlist is given**.

✅ Never touches everyday commands (`grep`, `ls`, `cat`, `curl`, `git`, …) — they exit on the first token, so normal latency is unchanged.
✅ Respects wordlists you already chose · idempotent · fail-safe · config-driven.

```bash
PIHUNT_CMDOPT=0                       # disable
~/.pi-hunt-rag/data/cmdopt.json       # disabledTools[] / extraTools{}
```

[Details →](pi-agent/extensions/cmd-optimizer/README.md)

### 3. RAG + knowledge graph — `pihunt-rag`

Semantic recall over four sources: a curated **techniques** KB (what top hunters do per class), the **Memory** graph, your **past findings**, and **model notes** (heuristics recorded for reuse). model2vec embeddings (CPU/numpy, ~30 MB, offline hashed-ngram fallback) in a SQLite vector store.

```bash
pihunt-rag search "oauth account takeover" --source techniques
pihunt-rag search "what did I find on acme" --program acme --source findings
pihunt-rag record "## title" "a reusable lesson"
```

MCP tools: `rag_search`, `rag_record_technique`, `rag_graph_add`, `rag_ingest`, `rag_stats`. [Details →](rag/README.md)

### 4. Continuous learning — `continuous-learning` extension

Turns hunt sessions into confidence-scored **instincts** and promotes the strong ones into the RAG (`model-notes`) so future hunts recall them via `rag_search`. Captures two cheap, deterministic signals: tagged lines the agents emit (`INSTINCT:` / `LESSON:` / `DEADEND:`, with optional `#domain` and `when <context>`) and repeated security-tool workflows. Complements `session-memory` (which persists/restores *session context*) — this learns *reusable behaviors*.

```bash
/instinct "pull source maps before fuzzing" --domain recon --when "target is Next.js"
/instinct-status              # ★ = promoted to the RAG
PIHUNT_LEARNING=0             # disable · PIHUNT_LEARNING_THRESHOLD tunes promotion (default 0.70)
```

[Details →](pi-agent/extensions/continuous-learning/README.md)

---

## Configuration

| Env / file | Purpose |
|---|---|
| `~/.pi/agent/settings.json` | default model, thinking level, retries, compaction |
| `~/.pi/agent/models.json` | OpenRouter model catalog and local credential placeholder |
| `~/.pi/agent/settings.json` | OpenRouter default provider/model, thinking, retries, compaction |
| `~/.pi-hunt-rag/data/cmdopt.json` | command-optimizer rules |
| `~/.pi-hunt-rag/data/instincts.jsonl` | learned instinct store (continuous-learning) |
| `PIHUNT_CMDOPT=0` | disable command optimization |
| `PIHUNT_LEARNING=0` | disable continuous learning · `PIHUNT_LEARNING_THRESHOLD` sets promotion cutoff (default `0.70`) |

---

## Requirements

**Required:** Node 22+, npm, Python 3.11+, `curl`, `rsync`, `git`, and a local OpenRouter credential configured outside version control.

**Optional:** Java (Burp MCP) · Chromium (browser-live) · the recon toolchain via `./extras.sh`.

`bootstrap.sh` installs the required ones for you on apt/dnf/pacman/brew systems.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `pi-hunt: command not found` | add `~/.local/bin` to `PATH` (see [Post-install](#post-install-one-time)) |
| OpenRouter authentication/model error | verify the local OpenRouter credential and that `~/.pi/agent/models.json` contains the selected model |
| RAG search empty / model missing | `pihunt-rag ingest` to rebuild; it falls back to an offline embedder if the model isn't downloaded |
| GitHub MCP 401 | set a valid token: `GITHUB_MCP_TOKEN=ghp_xxx ./install.sh` |
| Node too old | `bootstrap.sh` installs Node 22 via NodeSource on apt; otherwise upgrade manually |

---

## FAQ

**Is this safe to run against any site?** No. Only targets you own or are authorized to test under a bug-bounty scope. See the warning at the top.

**Does it submit reports automatically?** No. It produces local, reproducible report folders. You review and submit.

**Do I need all the MCP servers?** No. Memory + RAG + browser-live cover most flows; Burp and Gmail are optional.

**Can I use my own models?** Yes — edit `pi-agent/models.json` and `settings.json`; the default path is direct OpenRouter.

**Will it run offline?** The RAG falls back to an offline embedder, but the agent itself needs the model gateway (network).

---

## Contributing

PRs welcome. Good first contributions: new entries in `rag/techniques/techniques.yaml`, new specialist agents in `pi-agent/agents/`, or `cmd-optimizer` rules for more tools. Please keep secrets out of commits (`.gitignore` already excludes `.env`, tokens, and runtime state).

---

## License

[MIT](LICENSE). The retained legacy `kiro-gateway` subtree has its own upstream license.

<div align="center">
<sub>Built for researchers who'd rather design the hunt than babysit the tooling.</sub>
</div>
