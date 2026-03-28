# API Error Fix Report: NVIDIA Key Misconfiguration Guard

## Date
2026-03-28

## 1. Problem Description
The Agent Zero system was generating repeated `litellm.exceptions.AuthenticationError` tracebacks in the background logs after every agent response. These errors occurred specifically in the memory extension `_51_memorize_solutions.py`.

**Error Message:**
litellm.exceptions.AuthenticationError: OpenAIException - Incorrect API key provided: nvapi-...

**Impact:**
- Background threads crashed repeatedly, flooding logs with tracebacks.
- Memory memorization of solutions failed silently or crashed the background task.
- System appeared unstable due to continuous error logging.

## 2. Root Cause Analysis
- **Configuration Mismatch:** The environment variable `API_KEY_OPENAI` in `/a0/usr/.env` was set to an NVIDIA API key (starting with `nvapi-`).
- **Endpoint Mismatch:** The system's LLM configuration was set to use the `openai` provider (`openai/qwen/qwen3.5-122b-a10b`), which routes requests to OpenAI's default endpoint.
- **Result:** The system attempted to authenticate with OpenAI using an NVIDIA key, resulting in a 401 Authentication Error.
- **Lack of Guardrails:** The background extension code (`_51_memorize_solutions.py`) did not catch this specific error, causing the background thread to crash and log the full traceback.

## 3. Solution Implemented
A defensive coding patch was applied to TWO memory extensions to gracefully handle the `AuthenticationError` and bypass the failing step without crashing.

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
3. **Long-term Fix:** Update `_model_config/config.json`:
   - Set `utility_model.api_base` to `https://integrate.api.nvidia.com/v1` if NVIDIA models are intended.

## 5. Related Files
- **Skill Created:** `/a0/usr/workdir/Nvidia_Utility_Routing_and_API_Guard/SKILL.md`
- **Patched File 1:** `/a0/plugins/_memory/extensions/python/monologue_end/_51_memorize_solutions.py`
- **Patched File 2:** `/a0/plugins/_memory/extensions/python/message_loop_prompts_after/_50_recall_memories.py`

## 6. Developer Notes
This fix is a **temporary guard** to prevent system instability while the root configuration issue is resolved. Future improvements should include a global configuration validator that prevents the system from starting with mismatched keys.
