# Connecting Cline to a Local LM Studio Model

Using the **OpenAI Compatible** provider in Cline to reach a model served by LM Studio.

## Prerequisites

- LM Studio is running with its server started (the **Server** tab shows a green *Running* state).
- The model is loaded, and `http://<host>:1234/v1/models` returns it. In this setup the id is the full path:
  ```
  D:\LLMs\Qwen\Qwen2.5-Coder-14B-Instruct-GGUF\qwen2.5-coder-14b-instruct-q4_k_m.gguf
  ```

## Configuration

In **Cline settings**, set the API Provider to **OpenAI Compatible**, then fill in:

| Field       | Value |
|-------------|-------|
| Base URL    | `http://172.17.30.246:1234/v1` |
| API Key     | any dummy string, e.g. `lm-studio` (LM Studio ignores it, but the field may be required) |
| Model ID    | the exact id from `/v1/models` (see below) |

**Model ID** must match character-for-character, backslashes and all:
```
D:\LLMs\Qwen\Qwen2.5-Coder-14B-Instruct-GGUF\qwen2.5-coder-14b-instruct-q4_k_m.gguf
```
If it doesn't match exactly, you'll get a *model not found* error.

## The setting that matters most: context length

Cline's system prompt plus tool definitions are large — easily 10k–15k+ tokens before your task even begins. If LM Studio loaded the model with the default **4096**-token context, Cline can't fit its own instructions, the model receives a truncated prompt, and it starts emitting malformed tool calls — the "repeated tool call failures" loop.

**Fix:** In LM Studio, on the model's load settings, raise **Context Length** as high as your VRAM/RAM allows (aim for **16k–32k**). **Reload the model** after changing it.

## Address gotcha

`172.17.30.246` looks like a Docker/WSL bridge address.

- If VS Code and LM Studio are on the **same machine**, prefer `http://localhost:1234/v1` or `http://127.0.0.1:1234/v1` — simpler and avoids network issues.
- Use the `172.x` address only if Cline can't reach localhost (e.g. VS Code runs inside a container/WSL while LM Studio runs on the Windows host).

## First run tips

- Start Cline in **Plan mode** (it reads and plans without editing), then switch to **Act** once the plan looks right.
- Open a real project folder (**File → Open Folder**) before prompting.
- Keep tasks small and concrete. A 14B q4 coder model is capable but less robust at agentic tool-calling than frontier models, so tight, specific prompts work best.

## Quick troubleshooting

- **"model not found"** → Model ID doesn't match the `/v1/models` id exactly.
- **Garbled output / malformed tool calls** → Context length too low; raise it and reload.
- **Connection refused / timeout** → Wrong host address, or LM Studio server not running. Confirm `/v1/models` responds from the machine running VS Code.
