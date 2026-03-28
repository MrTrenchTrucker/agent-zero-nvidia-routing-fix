# Skill: Nvidia_Utility_Routing_and_API_Guard

## Description
A specialized skill to handle NVIDIA API key misconfiguration errors with OpenAI endpoints, implement strict API usage guards, and optimize terminal operations to prevent token overuse and infinite retry loops.

## Context
This skill is activated when the system encounters `litellm.exceptions.AuthenticationError` involving `nvapi-` keys being sent to OpenAI endpoints. It also enforces best practices for API cost management and terminal efficiency.

## Procedures

### 1. Error Detection and Immediate Halt
- **Trigger:** Detection of `AuthenticationError` with message "Incorrect API key provided: nvapi-..." during `call_utility_model` or memory extension execution.
- **Action:**
  - Immediately halt the current action sequence.
  - Do NOT retry the failed call.
  - Log the error clearly: "Critical Configuration Error: NVIDIA API key detected on OpenAI endpoint."
  - Notify the user or superior of the misconfiguration.

### 2. Configuration Correction
- **Diagnosis:** Identify that `API_KEY_OPENAI` (or similar) in the environment is set to an NVIDIA key (`nvapi-...`) while the model provider is set to `openai`.
- **Resolution Options:**
  - **Option A (Provider Routing):** Update the LLM configuration to use the correct NVIDIA base URL (e.g., `https://integrate.api.nvidia.com/v1`) and set the provider to `nvidia` or `nvidia/nemotron`.
  - **Option B (Key Replacement):** Replace the `API_KEY_OPENAI` environment variable with a valid OpenAI-compatible key if an OpenAI model is required.
  - **Option C (Bypass):** Temporarily disable memory recall extensions (`_50_recall_memories.py`, `_51_memorize_solutions.py`) that rely on the utility model until the configuration is fixed.

### 3. API Cost and Token Budgeting
- **Rule:** Never blindly retry failing API or tool calls.
- **Implementation:**
  - **Token Limits:** Enforce strict token budgets for all LLM interactions. If a task approaches the limit, pause and request user input or summarize.
  - **Batching:** Group terminal commands where possible (e.g., `cmd1 && cmd2 && cmd3`) to reduce overhead.
  - **Retry Logic:** Implement a maximum retry count (e.g., 1 or 2) with exponential backoff. If a call fails twice, abort and report the error.
  - **Efficiency:** Prefer local file operations and Linux commands over API calls for simple tasks.

### 4. Memory Extension Safety
- **Guard:** Before invoking memory search or save extensions, verify the utility model configuration is valid.
- **Fallback:** If the utility model is misconfigured, skip memory operations and proceed with the task using only immediate context, logging the skip.

## Usage Instructions
1. Load this skill when encountering `AuthenticationError` with `nvapi-` keys.
2. Use the `code_execution_tool` to inspect `/a0/usr/.env` or relevant config files.
3. Apply the resolution steps above.
4. Resume operations only after configuration is corrected or bypassed.

## Files
- `SKILL.md` (This file)
