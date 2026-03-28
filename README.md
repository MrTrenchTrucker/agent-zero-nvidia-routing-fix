# Agent Zero v1.x — NVIDIA API Routing Failure: Root Cause, Autonomous Response, and Hardening

**Document Type:** Community Disclosure / Bug Report
**Date:** 2026-03-28
**Intended Audience:** Agent Zero Core Development Team
**Classification:** Public (no secrets contained — all credentials redacted)
**Summary:** A misconfigured `utility_model.api_base` (empty string) caused cascading `AuthenticationError` floods. Agent Zero correctly diagnosed the problem and attempted autonomous self-repair. The self-repair logic was sound, but the patch was written with incorrect Python indentation (1-space vs. 8-space), causing an `IndentationError` that blocked all agent responses. Full manual recovery, root-cause elimination, persistence hardening, and a patch survival mechanism were applied. A custom APU Governor plugin kept the agent partially operational for 24+ hours throughout the incident.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Configuration at Time of Incident](#2-system-configuration-at-time-of-incident)
3. [Timeline of Events](#3-timeline-of-events)
4. [Root Cause Analysis](#4-root-cause-analysis)
5. [Agent Zero's Autonomous Response](#5-agent-zeros-autonomous-response)
6. [APU Governor Plugin — Resilience Contribution](#6-apu-governor-plugin--resilience-contribution)
7. [Manual Recovery — Step-by-Step](#7-manual-recovery--step-by-step)
8. [Post-Recovery Functional Test (Chat #9)](#8-post-recovery-functional-test-chat-9)
9. [Developer Recommendations](#9-developer-recommendations)
10. [Appendix A — Corrected Extension Files](#appendix-a--corrected-extension-files)
11. [Appendix B — A0's Self-Generated Documentation](#appendix-b--a0s-self-generated-documentation)

---

## 1. Executive Summary

On 2026-03-28, a production Agent Zero v1.x deployment suffered a cascading failure originating from a single misconfigured field: `utility_model.api_base` was set to an empty string (`""`) in `_model_config/config.json`. Agent Zero's LiteLLM integration silently falls back to `api.openai.com` when `api_base` is empty. Because the deployment was using an NVIDIA NIM API key (prefixed `nvapi-`), every background memory call was rejected by OpenAI with a 401 `AuthenticationError`.

Agent Zero correctly identified the repeated error, produced a detailed root-cause analysis document, created a reusable skill for handling this error class, and attempted to patch both affected extension files (`_50_recall_memories.py` and `_51_memorize_solutions.py`) with defensive `try/except AuthenticationError` guards. The patch logic was correct. However, the Python code was written with **1-space indentation instead of 8-space**, producing syntactically invalid files. On the next agent load, the extension loader raised `IndentationError` for `_50_recall_memories.py`, which **blocked all agent responses entirely**.

Throughout this period — spanning approximately 24+ hours of degraded operation — a custom **APU Governor** plugin (Automatic Polling Utility) kept the agent partially alive by providing background task throttling and polling independent of the broken memory extensions.

Full recovery required:
1. Rewriting both extension files with correct indentation (preserving A0's guard logic)
2. Fixing the actual root cause: setting `utility_model.api_base` to the correct NVIDIA NIM endpoint
3. Adding a persistence hardening layer (bind mounts, ctx_init.sh patch mechanism)
4. Adding `privileged: true` for full kernel capabilities inside the container

Post-recovery functional testing confirmed: memory save/recall, sub-agent delegation, and live host Docker socket access all working correctly.

**Key requests for the development team:**
- Startup config validator: detect empty or mismatched `api_base` fields
- Graceful extension loader: skip broken extensions (SyntaxError/IndentationError) rather than blocking all responses
- Self-repair safety: validate patched files with `py_compile` before activating
- Default compose template: include bind mounts for `/a0/usr/chats/`, `/a0/usr/scheduler/`, `/a0/usr/plugins/`

---

## 2. System Configuration at Time of Incident

| Field | Value |
|-------|-------|
| Agent Zero version | v1.x (`agent0ai/agent-zero:latest`) |
| Container name | `agent-zero-admin-throttled` |
| Container restart policy | `"no"` (on-demand) |
| LLM provider | NVIDIA NIM via LiteLLM |
| Chat model endpoint | `https://integrate.api.nvidia.com/v1` |
| Chat model name | `openai/qwen/qwen3.5-122b-a10b` (openai provider, NVIDIA base URL) |
| Utility model endpoint | `""` **(empty string — root cause)** |
| Utility model name | `openai/qwen/qwen3.5-122b-a10b` |
| API key prefix | `nvapi-***REDACTED***` |
| Container image | `agent0ai/agent-zero:latest` |
| Docker socket | Mounted at `/var/run/docker.sock` |
| Privileged mode | Not set at time of incident (added during recovery) |
| Web UI port | 50070 |
| SECRET_KEY | `***REDACTED***` |

**Volume mounts at time of incident (relevant):**

| Host Path | Container Path | Persists? |
|-----------|---------------|-----------|
| `AppData/workdir` | `/a0/usr/workdir` | Yes |
| `AppData/memory` | `/a0/usr/memory` | Yes |
| `AppData/plugins/apu_governor` | `/a0/usr/plugins/apu_governor` | Yes |
| *(not mounted)* | `/a0/usr/chats/` | **No — ephemeral** |
| *(not mounted)* | `/a0/usr/scheduler/` | **No — ephemeral** |
| *(not mounted)* | `/a0/usr/plugins/` (rest) | **No — ephemeral** |

The ephemeral nature of `/a0/usr/plugins/` meant that `_model_config/config.json` was also ephemeral — lost on container recreate. This is addressed in Section 7.5.

---

## 3. Timeline of Events

```
T-0:    utility_model.api_base set to "" in _model_config/config.json
        (chat_model.api_base correctly set to https://integrate.api.nvidia.com/v1)

T+0:    LiteLLM routes utility model calls to api.openai.com (silent fallback for empty api_base)
        Every background memory call (recall + memorize) authenticates with nvapi-*** key
        OpenAI rejects: AuthenticationError: Incorrect API key provided: nvapi-***

T+hours: Error repeats after every agent response
         Chat model unaffected — agent continues responding to users
         APU Governor plugin throttles background tasks, preventing resource exhaustion
         Agent partially functional throughout

T+~12h: Agent Zero detects the repeated AuthenticationError pattern
        Produces: API_ERROR_FIX_REPORT.md (self-written root cause analysis)
        Produces: Nvidia_Utility_Routing_and_API_Guard/SKILL.md (reusable skill)
        Attempts autonomous patch of _51_memorize_solutions.py and _50_recall_memories.py
        Adds try/except AuthenticationError guards — CORRECT LOGIC
        BUG: Patch written with 1-space Python indentation instead of 8-space

T+~24h: _50_recall_memories.py IndentationError blocks all agent responses
        _51_memorize_solutions.py secondary IndentationError
        Agent Zero no longer able to respond

T+24h:  Manual recovery initiated (remote session via ZeroTier)
        Diagnosis → Fix → Root cause elimination → Persistence hardening → Verify
        Full recovery completed same day
```

---

## 4. Root Cause Analysis

### 4.1 Primary Root Cause — Empty `utility_model.api_base`

In `_model_config/config.json`:

```json
{
  "chat_model": {
    "provider": "openai",
    "name": "qwen/qwen3.5-122b-a10b",
    "api_base": "https://integrate.api.nvidia.com/v1"
  },
  "utility_model": {
    "provider": "openai",
    "name": "qwen/qwen3.5-122b-a10b",
    "api_base": ""
  }
}
```

The `chat_model` was correctly configured. The `utility_model` had `api_base: ""`.

**LiteLLM behavior:** When `api_base` is an empty string, LiteLLM treats it as unset and uses the provider's default endpoint. For `provider: "openai"`, the default is `https://api.openai.com/v1`. There is no warning or error at startup — the misconfiguration is silent until the first API call.

**Result:** All utility model calls (memory recall query generation, memory consolidation, memory filter validation) were routed to OpenAI's API. The NVIDIA NIM key (`nvapi-***`) was rejected with HTTP 401 `AuthenticationError`.

### 4.2 Secondary Root Cause — Self-Repair Indentation Error

Agent Zero's autonomous self-repair attempted to write `try/except AuthenticationError` guards into both extension files. The repair logic was architecturally correct. The error was in execution: the code was output with **1-space indentation** instead of the required **8-space indentation** (method body level inside an async method).

**Before patch (original code — correct indentation):**
```python
    async def search_memories(self, log_item, loop_data, **kwargs):
        # ...setup code...

        if set["memory_recall_query_prep"]:
            query = await self.agent.call_utility_model(...)  # 12-space indent
```

**After A0's patch (incorrect indentation — 1-space):**
```python
    async def search_memories(self, log_item, loop_data, **kwargs):
        # ...setup code...

 if set["memory_recall_query_prep"]:   # 1-SPACE — IndentationError
  try:
   query = await self.agent.call_utility_model(...)
```

Python's extension loader raises `IndentationError` on import of the file, which propagates upward and blocks the entire extension loading system — not just the affected extension.

### 4.3 Systemic Gaps

| Gap | Impact |
|-----|--------|
| No startup validation of `utility_model.api_base` | Silent misconfiguration runs indefinitely |
| LiteLLM `api_base: ""` silently falls back to OpenAI | No user-visible warning, hard to diagnose |
| Extension loader does not catch `SyntaxError`/`IndentationError` per-extension | One broken file blocks ALL extensions and ALL responses |
| No write-validate-commit cycle for self-repair patches | A0's patch was never `py_compile`-checked before activation |
| No rollback mechanism if a patched file fails | Once broken, manual intervention required |
| Most of `/a0/usr/` not volume-mounted by default | Chats, plugins, scheduler data lost on container recreate |

---

## 5. Agent Zero's Autonomous Response

Despite the system failure that ultimately resulted from the self-repair attempt, Agent Zero's autonomous diagnostic work was impressive and accurate. The following documents were produced by A0 independently, stored in its persistent working directory, and survived the incident.

### 5.1 `API_ERROR_FIX_REPORT.md` (A0's self-written root cause analysis)

> *Reproduced verbatim from `/a0/usr/workdir/API_ERROR_FIX_REPORT.md`:*

---

```markdown
# API Error Fix Report: NVIDIA Key Misconfiguration Guard

## Date
2026-03-28

## 1. Problem Description
The Agent Zero system was generating repeated `litellm.exceptions.AuthenticationError` tracebacks
in the background logs after every agent response. These errors occurred specifically in the memory
extension `_51_memorize_solutions.py`.

**Error Message:**
litellm.exceptions.AuthenticationError: OpenAIException - Incorrect API key provided: nvapi-...

**Impact:**
- Background threads crashed repeatedly, flooding logs with tracebacks.
- Memory memorization of solutions failed silently or crashed the background task.
- System appeared unstable due to continuous error logging.

## 2. Root Cause Analysis
- **Configuration Mismatch:** The environment variable `API_KEY_OPENAI` in `/a0/usr/.env` was set to an
  NVIDIA API key (starting with `nvapi-`).
- **Endpoint Mismatch:** The system's LLM configuration was set to use the `openai` provider
  (`openai/qwen/qwen3.5-122b-a10b`), which routes requests to OpenAI's default endpoint.
- **Result:** The system attempted to authenticate with OpenAI using an NVIDIA key, resulting in a
  401 Authentication Error.
- **Lack of Guardrails:** The background extension code (`_51_memorize_solutions.py`) did not catch
  this specific error, causing the background thread to crash and log the full traceback.

## 3. Solution Implemented
A defensive coding patch was applied to TWO memory extensions to gracefully handle the
`AuthenticationError` and bypass the failing step without crashing.

### Changes Made
**File 1:** `/a0/plugins/_memory/extensions/python/monologue_end/_51_memorize_solutions.py`
**File 2:** `/a0/plugins/_memory/extensions/python/message_loop_prompts_after/_50_recall_memories.py`

**1. Added Imports (to both files):**
import litellm
from litellm.exceptions import AuthenticationError

**2. Wrapped Utility Calls in Try-Except Blocks:**
The calls to `self.agent.call_utility_model()` in both files were wrapped to catch `AuthenticationError`.

**Logic (applied to both files):**
- If `AuthenticationError` is caught and the error message contains `"nvapi-"`:
  - Log a warning: "SKIPPED: Invalid API Key (NVIDIA key on OpenAI endpoint)"
  - Log a message: "Memory [operation] skipped due to configuration error..."
  - **Return gracefully** from the function, preventing the thread crash.
- If a different `AuthenticationError` occurs, re-raise it to maintain existing error handling.
- Catch any other generic exceptions to prevent future crashes.

## 4. Verification Steps
1. **Trigger:** Perform a task that generates a solution to be memorized (e.g., complete a coding task).
2. **Observation:**
   - The background log should show: `SKIPPED: Invalid API Key (NVIDIA key on OpenAI endpoint)`.
   - **No** traceback stack trace should appear.
   - The main agent response should complete successfully.
3. **Long-term Fix:** Update `/a0/usr/.env`:
   - Set `API_KEY_OPENAI` to a valid OpenAI-compatible key, OR
   - Update the LLM provider configuration to use the NVIDIA endpoint
     (`https://integrate.api.nvidia.com/v1`) if NVIDIA models are intended.

## 5. Related Files
- **Skill Created:** `/a0/usr/workdir/Nvidia_Utility_Routing_and_API_Guard/SKILL.md`
  - Defines the logic for handling this error and enforcing token budgeting.
- **Patched File:** `/a0/plugins/_memory/extensions/python/monologue_end/_51_memorize_solutions.py`

## 6. Developer Notes
This fix is a **temporary guard** to prevent system instability while the root configuration issue is
resolved. The memory extension will skip memorization until the API key is corrected. Future
improvements should include a global configuration validator that prevents the system from starting
with mismatched keys.
```

---

### 5.2 `Nvidia_Utility_Routing_and_API_Guard/SKILL.md` (A0's self-created reusable skill)

> *Reproduced verbatim from `/a0/usr/workdir/Nvidia_Utility_Routing_and_API_Guard/SKILL.md`:*

---

```markdown
# Skill: Nvidia_Utility_Routing_and_API_Guard

## Description
A specialized skill to handle NVIDIA API key misconfiguration errors with OpenAI endpoints, implement
strict API usage guards, and optimize terminal operations to prevent token overuse and infinite retry
loops.

## Context
This skill is activated when the system encounters `litellm.exceptions.AuthenticationError` involving
`nvapi-` keys being sent to OpenAI endpoints. It also enforces best practices for API cost management
and terminal efficiency.

## Procedures

### 1. Error Detection and Immediate Halt
- **Trigger:** Detection of `AuthenticationError` with message "Incorrect API key provided: nvapi-..."
  during `call_utility_model` or memory extension execution.
- **Action:**
  - Immediately halt the current action sequence.
  - Do NOT retry the failed call.
  - Log the error clearly: "Critical Configuration Error: NVIDIA API key detected on OpenAI endpoint."
  - Notify the user or superior of the misconfiguration.

### 2. Configuration Correction
- **Diagnosis:** Identify that `API_KEY_OPENAI` (or similar) in the environment is set to an NVIDIA
  key (`nvapi-...`) while the model provider is set to `openai`.
- **Resolution Options:**
  - **Option A (Provider Routing):** Update the LLM configuration to use the correct NVIDIA base URL
    (e.g., `https://integrate.api.nvidia.com/v1`) and set the provider to `nvidia` or
    `nvidia/nemotron`.
  - **Option B (Key Replacement):** Replace the `API_KEY_OPENAI` environment variable with a valid
    OpenAI-compatible key if an OpenAI model is required.
  - **Option C (Bypass):** Temporarily disable memory recall extensions (`_50_recall_memories.py`,
    `_51_memorize_solutions.py`) that rely on the utility model until the configuration is fixed.

### 3. API Cost and Token Budgeting
- **Rule:** Never blindly retry failing API or tool calls.
- **Implementation:**
  - **Token Limits:** Enforce strict token budgets for all LLM interactions. If a task approaches the
    limit, pause and request user input or summarize.
  - **Batching:** Group terminal commands where possible (e.g., `cmd1 && cmd2 && cmd3`) to reduce
    overhead.
  - **Retry Logic:** Implement a maximum retry count (e.g., 1 or 2) with exponential backoff. If a
    call fails twice, abort and report the error.
  - **Efficiency:** Prefer local file operations and Linux commands over API calls for simple tasks.

### 4. Memory Extension Safety
- **Guard:** Before invoking memory search or save extensions, verify the utility model configuration
  is valid.
- **Fallback:** If the utility model is misconfigured, skip memory operations and proceed with the
  task using only immediate context, logging the skip.

## Usage Instructions
1. Load this skill when encountering `AuthenticationError` with `nvapi-` keys.
2. Use the `code_execution_tool` to inspect `/a0/usr/.env` or relevant config files.
3. Apply the resolution steps above.
4. Resume operations only after configuration is corrected or bypassed.

## Files
- `SKILL.md` (This file)
- (Optional) Scripts for config validation can be added to `scripts/`
```

---

### 5.3 Assessment of A0's Autonomous Response

**What A0 got right:**
- Correctly identified the `nvapi-`/OpenAI endpoint mismatch as root cause
- Correctly identified both affected extension files
- Correctly chose the `try/except AuthenticationError` approach with nvapi- string detection
- Correctly differentiated between NVIDIA-key-on-OpenAI errors (skip gracefully) vs. other auth errors (re-raise)
- Produced well-structured documentation and a reusable skill
- Did not attempt to retry the failing calls endlessly (followed token budgeting rules)

**What failed:**
- The Python patch was written with 1-space indentation instead of 8-space, making the file syntactically invalid
- No `py_compile` or syntax validation step was performed before the patch was activated
- The indentation error in the higher-priority `_50_recall_memories.py` (which runs in the main agent loop) was more catastrophic than the original bug

**Conclusion for dev team:** The self-repair reasoning was accurate. The failure was at the **code execution / string formatting layer**, not the reasoning layer. A post-write `py_compile` validation step would have prevented the catastrophic outcome.

---

## 6. APU Governor Plugin — Resilience Contribution

The **APU Governor** (Automatic Polling Utility Governor) is a custom Agent Zero plugin developed for this deployment. It is mounted as a persistent, isolated volume at `/a0/usr/plugins/apu_governor` — separate from the main plugins directory.

### Role During the Incident

- Provided background task throttling and polling independent of the broken memory extensions
- Managed API call rate limiting, preventing the repeated `AuthenticationError` floods from consuming all available resources or triggering rate limits on the NVIDIA NIM free tier
- Kept Agent Zero responsive to user input for approximately 24+ hours despite:
  - Repeated background `AuthenticationError` on every response
  - Memory recall and memorization completely failing
  - Background threads crashing repeatedly
- Continued operating correctly even after the `IndentationError` was introduced in the memory extensions, because it is in a completely separate plugin directory

### Why It Survived

The APU Governor plugin has its own dedicated bind mount (`/a0/usr/plugins/apu_governor`). This means it is not affected by changes to the parent `/a0/usr/plugins/` directory, and it loads independently of the memory extension subsystem. It never touched the broken files.

### Recommendation for Dev Team

The APU Governor architecture — an isolated, separately-mounted plugin volume — represents a robust pattern for resilient background monitoring. We recommend this be documented as the preferred pattern for critical background plugins that must survive extension-layer failures. Consider shipping a lightweight equivalent in the default Agent Zero distribution.

---

## 7. Manual Recovery — Step-by-Step

Recovery was performed remotely via ZeroTier overlay network (local LAN was unreachable). SSH to the CONTAK-01 server was performed using Python `paramiko` (Git Bash SSH was unavailable due to a libcrypto issue on the recovery workstation).

### 7.1 — Diagnosis

**Commands run:**
```bash
docker logs agent-zero-admin-throttled 2>&1 | grep -i "indentation\|syntaxerror\|error" | tail -30
docker exec agent-zero-admin-throttled cat /a0/usr/plugins/_model_config/config.json
```

**Findings:**
1. `IndentationError` at `_50_recall_memories.py` line 96 — confirmed BLOCKING crash
2. `utility_model.api_base` was `""` in config.json — confirmed root cause
3. Both extension files had 1-space indentation in the patched blocks

### 7.2 — Fix `_50_recall_memories.py` (BLOCKING)

**File:** `/a0/plugins/_memory/extensions/python/message_loop_prompts_after/_50_recall_memories.py`

The `if set["memory_recall_query_prep"]:` block (~line 96 to ~133) was rewritten with correct 8-space indentation, preserving A0's AuthenticationError guard logic and adding a generic Exception catch.

**Why bash heredoc failed:** The file contains Python f-strings with single quotes inside double-quoted strings (e.g., `f"area == '{Memory.Area.MAIN.value}'"` and `f"area == '{Memory.Area.SOLUTIONS.value}'"`) which conflict with shell quoting. The heredoc approach was abandoned in favor of:

1. Write corrected file locally using a file write tool
2. Syntax-check: `python -m py_compile _50_recall_memories_fixed.py` (zero errors)
3. SFTP transfer to `/tmp/_50_recall_memories_fixed.py` on CONTAK-01
4. `docker cp /tmp/_50_recall_memories_fixed.py agent-zero-admin-throttled:/a0/plugins/_memory/extensions/python/message_loop_prompts_after/_50_recall_memories.py`

The corrected block is included in [Appendix A](#appendix-a--corrected-extension-files).

### 7.3 — Fix `_51_memorize_solutions.py` (Secondary)

**File:** `/a0/plugins/_memory/extensions/python/monologue_end/_51_memorize_solutions.py`

The inner `try/except` block wrapping `call_utility_model` was re-indented from 1-space to 12-space (inside outer `try:` at 8-space, inside `async def memorize`). Same delivery method as Step 7.2.

### 7.4 — Fix `utility_model.api_base` (Root Cause Elimination)

**File:** `/a0/usr/plugins/_model_config/config.json`

**Before:**
```json
"utility_model": {
    "provider": "openai",
    "name": "qwen/qwen3.5-122b-a10b",
    "api_base": ""
}
```

**After:**
```json
"utility_model": {
    "provider": "openai",
    "name": "qwen/qwen3.5-122b-a10b",
    "api_base": "https://integrate.api.nvidia.com/v1"
}
```

This eliminates the `AuthenticationError` entirely, making A0's defensive guards a safety net rather than a band-aid.

**Important note for A0 v1.x users:** API keys in v1.x are managed through the web UI, not `.env` or `settings.json`. If you see `API_KEY_OPENAI` in `.env`, it may be a legacy field from v0.x that is no longer used for authentication in v1.x. Enter API keys through the web UI interface.

### 7.5 — Persistence Hardening

**Discovery:** Most of `/a0/usr/` was not volume-mounted, meaning chats, scheduler tasks, and user plugins were lost on `docker compose up --force-recreate`.

**Solution:** Added 3 new bind mounts to the docker-compose service definition (additive — all existing mounts preserved unchanged):

```yaml
# NEW mounts added
- /path/to/AppData/chats:/a0/usr/chats:rw
- /path/to/AppData/scheduler:/a0/usr/scheduler:rw
- /path/to/AppData/usr_plugins:/a0/usr/plugins:rw
# apu_governor subdir mount still takes precedence via Docker nested mount rule
- /path/to/AppData/plugins/apu_governor:/a0/usr/plugins/apu_governor:rw
```

**Docker nested mount behavior:** When both a parent directory (`/a0/usr/plugins`) and a subdirectory (`/a0/usr/plugins/apu_governor`) are bind-mounted, Docker gives precedence to the more-specific (subdirectory) mount. The `apu_governor` plugin continued to load from its own dedicated host directory correctly.

Additional hardening:
- `privileged: true` added to compose for full kernel capabilities
- `/:/hostfs` changed from `:rslave` to `:rw` for full host filesystem write access

### 7.6 — Patch Survival via `ctx_init.sh`

**Problem discovered:** `/a0/plugins/` is populated by `copy_A0.sh` at container startup from `/git/agent-zero`. This overwrites any manual patches unless they are pre-placed first.

**Container startup sequence (critical to understand):**
```
docker compose up
  └── command: [ "/bin/bash", "-c", "[ -f /a0/usr/workdir/ctx_init.sh ] && bash /a0/usr/workdir/ctx_init.sh; exec /exe/initialize.sh $BRANCH" ]
        └── 1. ctx_init.sh runs      ← /a0/ is EMPTY at this point
        └── 2. initialize.sh → supervisord → run_A0.sh → copy_A0.sh
                └── copy_A0.sh: if [ ! -f /a0/run_ui.py ]; then cp -rn /git/agent-zero/. /a0/; fi
                      └── cp -rn = no-clobber: WILL NOT overwrite files that already exist
        └── 3. A0 web server starts
```

**Key insight:** `ctx_init.sh` runs before `/a0/` is populated. Files written to `/a0/plugins/` during `ctx_init.sh` are preserved because `copy_A0.sh` uses `cp -rn` (no-clobber). Files must be **pre-placed** — not post-placed.

**Critical detail:** `mkdir -p` is required before each `cp` because `/a0/plugins/` does not exist when `ctx_init.sh` runs. Without `mkdir -p`, the `cp` silently fails.

**Solution added to `ctx_init.sh`:**
```bash
# Restore patched memory extension files (fix for indentation bug + NVIDIA API guard)
PATCHES_DIR="/a0/usr/workdir/patches"
AFTER_DIR="/a0/plugins/_memory/extensions/python/message_loop_prompts_after"
MONOLOGUE_DIR="/a0/plugins/_memory/extensions/python/monologue_end"

if [ -f "$PATCHES_DIR/_50_recall_memories.py" ]; then
    mkdir -p "$AFTER_DIR"
    cp "$PATCHES_DIR/_50_recall_memories.py" "$AFTER_DIR/_50_recall_memories.py"
    echo "[ctx_init] Patched _50_recall_memories.py pre-placed"
fi
if [ -f "$PATCHES_DIR/_51_memorize_solutions.py" ]; then
    mkdir -p "$MONOLOGUE_DIR"
    cp "$PATCHES_DIR/_51_memorize_solutions.py" "$MONOLOGUE_DIR/_51_memorize_solutions.py"
    echo "[ctx_init] Patched _51_memorize_solutions.py pre-placed"
fi
```

Fixed files stored at: `AppData/workdir/patches/` (persistent volume) — so they survive container recreates and are available to `ctx_init.sh` on every startup.

### 7.7 — Container Recreate and Verification

```bash
docker compose -f /path/to/docker-compose.yml up -d --force-recreate agent-zero-admin-throttled
```

**Verification checks run:**
```bash
# No IndentationErrors
docker logs agent-zero-admin-throttled 2>&1 | grep -i indentation
# (empty — confirmed)

# Privileged mode
docker inspect agent-zero-admin-throttled --format '{{.HostConfig.Privileged}}'
# true

# Chats mounted
docker exec agent-zero-admin-throttled ls /a0/usr/chats/ | wc -l
# 8 (previous chats preserved)

# API base correct
docker exec agent-zero-admin-throttled python3 -c "
import json
with open('/a0/usr/plugins/_model_config/config.json') as f:
    c = json.load(f)
print('chat:', c['chat_model']['api_base'])
print('util:', c['utility_model']['api_base'])
"
# chat: https://integrate.api.nvidia.com/v1
# util: https://integrate.api.nvidia.com/v1
```

All checks passed.

### 7.8 — Recovery Script Updates

`contak_scripts_sync.sh` was updated to automatically back up the docker-compose file on each sync:

```bash
mkdir -p "$PERSISTENT/compose_backups"
cp -p /path/to/docker-compose.yml "$PERSISTENT/compose_backups/agent0_compose_current.yml" 2>/dev/null || true
```

### 7.9 — Image Backup

```bash
# Commit running container state
docker commit agent-zero-admin-throttled agent-zero-admin-throttled:stable-20260328

# Export as gzip archive
docker save agent-zero-admin-throttled:stable-20260328 | gzip > /tmp/a0_admin_stable_20260328.tar.gz

# Move to persistent Nextcloud-accessible storage
mv /tmp/a0_admin_stable_20260328.tar.gz /path/to/nextcloud/Docker_Configs/
```

Resulting archive: ~2.7 GB. Verified visible in Nextcloud after `occ files:scan`.

---

## 8. Post-Recovery Functional Test (Chat #9)

After recovery, a 3-part functional test was conducted in a fresh chat session (Chat ID: `FBjRIKI1`, 2026-03-28 ~13:00 UTC). The following is the meaningful log sequence from that session with all secrets redacted.

---

**Phase 1 — Pre-Key Entry (AuthenticationErrors visible)**

Agent Zero received "hello world" messages before the API key was entered in the web UI. Logs showed repeated AuthenticationErrors:

```
[warning] litellm.exceptions.AuthenticationError: OpenAIException -
The api_key client option must be set either by passing api_key to the client
or by setting the OPENAI_API_KEY environment variable
```

Agent Zero self-diagnosed and reported to the user:

> 🚨 **Critical Configuration Error Detected** — OpenAI API Key is missing.
> Error: `litellm.exceptions.AuthenticationError`
> Instructions to fix: set OPENAI_API_KEY env var in Docker or Agent Zero config.

*(Note: A0 v1.x requires keys to be entered via the web UI, not env vars — this message reflects the pre-v1.x guidance. The actual fix was entering the key in the web UI.)*

---

**Phase 2 — API Key Confirmed Working**

User: *"ah yes we fixed some things you must of seen it. since you are replying the key is working! We need to check basic functions. do a tool call to record a test memory, then have a sub agent recall the test memory, then spool up another sub agent to run a terminal command on the native host machine outside your container: docker ps"*

Agent Zero's planning response:
> Thoughts: "The user confirmed the API key is working. Step 1: Record a test memory using memory_save. Step 2: Delegate to subordinate (researcher profile) to recall memory. Step 3: Delegate to another subordinate (developer profile) to run docker ps."

---

**Phase 3 — 3-Part Functional Test**

**Test 1: Memory Save**
- Tool: `memory_save`
- Input: `"TEST_MEMORY_001: This is a test memory created on 2026-03-28 to verify agent memory recall and cross-container command execution capabilities."`
- Result: ✅ Memory saved with ID `qCd5WSJ4DZ`

**Test 2: Sub-Agent Memory Recall (Researcher profile)**
- Agent Zero delegated to a Researcher subordinate
- Subordinate ran `memory_load` with query `"TEST_MEMORY_001"` → returned empty (no vector match by that query string)
- **However:** The subordinate's `[EXTRAS]` context already contained the full memory content, injected by the automatic `_50_recall_memories.py` extension
- Subordinate correctly reported: *"The memory_load tool search returned no results. However, the system context [EXTRAS] explicitly contains the memory content: TEST_MEMORY_001..."*
- Result: ✅ Memory content confirmed present and accessible

*(Developer note: This is expected behavior. The recall extension injects relevant memories into `[EXTRAS]` before the agent even runs a tool search. The `memory_load` tool is for explicit retrieval by query; the extension handles automatic injection. Both paths are working correctly.)*

**Test 3: Sub-Agent Docker Command (Developer profile)**
- Agent Zero delegated to a Developer subordinate
- Subordinate ran: `docker ps` via `code_execution_tool` (runtime: `terminal`)
- Result: ✅ Full docker ps output returned — 22 containers listed, Docker socket confirmed accessible

```
CONTAINER ID   IMAGE                          COMMAND        STATUS        NAMES
0494d88845af   agent0ai/agent-zero:latest    "/bin/bash..."  Up 27 min    agent-zero-admin-throttled
2e2d83e7ecc4   ghcr.io/berriai/litellm:...   "..."           Up 2 hrs     litellm
[...19 additional containers...]
```

---

**Agent Zero's Final Report:**

> ✅ **All Basic Function Tests Passed Successfully!**
>
> **1. Memory Recording ✅**
> - Memory saved with ID `qCd5WSJ4DZ`
> - Content confirmed in system
>
> **2. Memory Recall via Sub-Agent ✅**
> - Content found via system context [EXTRAS] — direct ID lookup had a minor quirk but content confirmed present
>
> **3. Docker ps via Sub-Agent ✅**
> - 22 containers listed successfully
> - Docker socket is correctly mounted

---

## 9. Developer Recommendations

These are submitted for consideration in the next Agent Zero release.

### 9.1 Startup Configuration Validator *(HIGH PRIORITY)*

Add a startup check that validates model configuration before the agent begins serving requests:

```python
# Pseudocode
def validate_model_config(config):
    for model_name in ['chat_model', 'utility_model']:
        model = config.get(model_name, {})
        api_base = model.get('api_base', '')
        api_key = get_api_key(model)

        if not api_base:
            log.warning(f"{model_name}.api_base is empty — will fall back to provider default")

        if api_key and api_key.startswith('nvapi-') and 'openai.com' in (api_base or ''):
            raise ConfigurationError(
                f"{model_name}: NVIDIA API key detected but endpoint is OpenAI. "
                f"Set api_base to 'https://integrate.api.nvidia.com/v1'"
            )
```

At minimum, log a prominent warning if `api_base` is empty for a non-default-provider key.

### 9.2 Graceful Extension Loader *(HIGH PRIORITY)*

The current extension loader appears to raise `IndentationError`/`SyntaxError` globally when any extension file is syntactically invalid, blocking all extensions. The loader should instead catch per-extension import errors:

```python
# Pseudocode
for extension_file in extension_files:
    try:
        load_extension(extension_file)
    except (IndentationError, SyntaxError) as e:
        log.error(f"Extension {extension_file} failed to load: {e} — skipping")
        continue  # Don't block the entire system
    except Exception as e:
        log.warning(f"Extension {extension_file} raised unexpected error: {e}")
        continue
```

This change alone would have prevented the catastrophic outcome — A0 would have continued responding while the memory extensions were broken.

### 9.3 Self-Repair Safety — Validate Before Activating *(HIGH PRIORITY)*

When Agent Zero writes a patch to a Python file, it should validate syntax before making the file active:

```python
# Pseudocode — after writing patch
import py_compile, tempfile, shutil

tmp = write_to_temp_file(patch_content)
try:
    py_compile.compile(tmp, doraise=True)
    shutil.move(tmp, target_path)  # Only activate if valid
    log("Patch applied and validated successfully")
except py_compile.PyCompileError as e:
    os.remove(tmp)
    log.error(f"Patch validation failed: {e} — original file preserved")
    # Report failure to user, do not activate broken patch
```

### 9.4 Atomic Writes for Self-Repair

Use write-to-temp-then-rename (`os.replace`) for all self-repair patches. This prevents partial-write corruption if the process is interrupted mid-write.

```python
tmp_path = target_path + '.tmp'
with open(tmp_path, 'w') as f:
    f.write(patch_content)
# Validate here (see 9.3), then:
os.replace(tmp_path, target_path)
```

### 9.5 Default Compose Template — Persistence Mounts

The default `docker-compose.yml` template should include bind mounts for:

```yaml
volumes:
  - ./data/chats:/a0/usr/chats:rw          # User chat history
  - ./data/scheduler:/a0/usr/scheduler:rw  # Scheduled tasks
  - ./data/plugins:/a0/usr/plugins:rw      # User plugins
  - ./data/memory:/a0/usr/memory:rw        # Vector memory (already common)
  - ./data/workdir:/a0/usr/workdir:rw      # Working directory (already common)
```

Without these mounts, users lose chat history, installed plugins, and scheduled tasks on every `--force-recreate`.

### 9.6 LiteLLM `api_base: ""` Silent Fallback

Document clearly (or add a guard) that `api_base: ""` in model configuration causes LiteLLM to fall back silently to the provider's default endpoint (`api.openai.com` for `provider: "openai"`). This is counterintuitive and hard to diagnose. Options:

- Raise `ConfigurationError` if `api_base` is empty string (not null/undefined)
- Add a startup log entry: `"utility_model.api_base is empty — using OpenAI default endpoint"`

### 9.7 Core AuthenticationError Handling

The defensive pattern that Agent Zero wrote into both extension files:

```python
except AuthenticationError as e:
    if "nvapi-" in str(e):
        log_item.update(heading="SKIPPED: NVIDIA key on OpenAI endpoint")
        return  # Graceful skip
    else:
        raise  # Re-raise other auth errors
```

...should live in the core extension runner or utility model caller, not patched file-by-file. Any extension that calls `call_utility_model` should benefit from this guard automatically.

---

## Appendix A — Corrected Extension Files

### A.1 — `_50_recall_memories.py` — Corrected Block

The following is the corrected `if set["memory_recall_query_prep"]:` block (8-space indentation, inside `async def search_memories`). This block replaced the 1-space version written by A0's self-repair:

```python
        # if query preparation by AI is enabled
        if set["memory_recall_query_prep"]:
            try:
                # call util llm to generate search query from the conversation
                query = await self.agent.call_utility_model(
                    system=system,
                    message=message,
                )
                query = query.strip()
                log_item.update(query=query)
            except AuthenticationError as e:
                # Guard against NVIDIA API key being sent to OpenAI endpoint
                if "nvapi-" in str(e):
                    log_item.update(heading="SKIPPED: Invalid API Key (NVIDIA key on OpenAI endpoint)")
                    log_item.update(content="Memory recall skipped due to configuration error. "
                                            "Please fix provider routing (api_base).")
                    query = ""
                else:
                    # Re-raise if it is a different authentication error
                    raise
            except Exception as e:
                # Catch any other unexpected errors
                err = errors.format_error(e)
                self.agent.context.log.log(
                    type="warning", heading="Recall memories extension error:", content=err
                )
                query = ""

            # no query, no search
            if not query:
                log_item.update(
                    heading="Failed to generate memory query",
                )
                return

        # otherwise use the message and history as query
        else:
            query = user_instruction + "\n\n" + history
```

**Required imports added to file header:**
```python
import litellm
from litellm.exceptions import AuthenticationError
```

### A.2 — `_51_memorize_solutions.py` — Corrected Block

The following is the corrected `try/except` block around `call_utility_model` inside `async def memorize` (12-space indentation — inside outer `try:` at 8-space):

```python
            # call util llm to find solutions in history
            try:
                solutions_json = await self.agent.call_utility_model(
                    system=system,
                    message=msgs_text,
                    background=True,
                )
            except AuthenticationError as e:
                # Guard against NVIDIA API key being sent to OpenAI endpoint
                if "nvapi-" in str(e):
                    log_item.update(heading="SKIPPED: Invalid API Key (NVIDIA key on OpenAI endpoint)")
                    log_item.update(content="Memory memorization skipped due to configuration error. "
                                            "Please fix provider routing (api_base).")
                    return
                else:
                    raise
            except Exception as e:
                # Catch any other unexpected errors to prevent thread crashes
                log_item.update(heading="SKIPPED: Memory memorization failed")
                log_item.update(content=f"Error: {str(e)}")
                return
```

**Required imports added to file header:**
```python
import litellm
from litellm.exceptions import AuthenticationError
```

---

## Appendix B — A0's Self-Generated Documentation

Both documents (`API_ERROR_FIX_REPORT.md` and `Nvidia_Utility_Routing_and_API_Guard/SKILL.md`) are reproduced in full in [Section 5](#5-agent-zeros-autonomous-response). They are stored at:

- `/a0/usr/workdir/API_ERROR_FIX_REPORT.md`
- `/a0/usr/workdir/Nvidia_Utility_Routing_and_API_Guard/SKILL.md`

These files were written autonomously by Agent Zero during the incident, without human instruction, using its `code_execution_tool`. They represent A0's self-documented understanding of the problem and its structured approach to resolution.

---

*Document prepared 2026-03-28. All API keys, passwords, and credentials have been redacted. This document is safe for public distribution to the Agent Zero development community.*

---

## License

MIT License

Copyright (c) 2026 MrTrenchTrucker

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
