# RunningHub Image Skill (Working Summary)

This document summarizes the **working** RunningHub image-generation skill setup used with OpenClaw Web UI.

## Goal

Enable OpenClaw chat to generate images via RunningHub with stable behavior in Web UI, avoiding:
- missing script path errors
- repeated API-key confirmation loops
- fake placeholder API keys passed by model output

## Final Working Architecture

- Skill file path (workspace override):
  - `/root/.openclaw/workspace/skills/runninghub/SKILL.md`
- Script path (workspace executable target):
  - `/root/.openclaw/workspace/scripts/runninghub.py`
- Config key source:
  - `~/.openclaw/openclaw.json`
  - `skills.entries.runninghub.env.RUNNINGHUB_API_KEY`

Reason: OpenClaw agent runtime executes commands from workspace (`/root/.openclaw/workspace`) and initially failed when skill referenced other roots.

## Skill Behavior That Works

`runninghub.py` supports:
- `text-to-image`
- `image-to-image`
- `image-to-video`
- `text-to-video`

For image generation in chat, the stable path is:
1. user asks in Web UI
2. agent reads runninghub skill
3. agent executes:
   - `python3 /root/.openclaw/workspace/scripts/runninghub.py ...`
4. script submits async task, polls result, downloads output
5. script prints:
   - `MEDIA:/absolute/path/to/output.jpg`
6. Web UI can render image preview from MEDIA attachment line

## Critical Fixes Applied

### 1) Script location mismatch fix

Problem:
- agent tried `/root/.openclaw/workspace/scripts/runninghub.py`
- file originally existed elsewhere

Fix:
- copy script into workspace scripts path

### 2) API key fallback fix

Problem:
- skill env injection was not always reliable in chat session

Fix in script:
- key resolution order:
  1. `--api-key` if valid non-placeholder
  2. `RUNNINGHUB_API_KEY` env
  3. `~/.openclaw/openclaw.json` at:
     `skills.entries.runninghub.env.RUNNINGHUB_API_KEY`

### 3) Placeholder key guard

Problem:
- model occasionally used placeholder like `your_api_key_here`

Fix:
- script treats known placeholder strings as invalid and falls back to env/config key

### 4) Tool output rendering path

Requirement for image preview in OpenClaw:
- include `MEDIA:/absolute/path/to/file` in tool output

Verified:
- emitted `MEDIA:/tmp/runninghub-output/space-cat-webui.jpg`

## Stable Prompt Pattern (Web UI)

Use directive style prompts to reduce confirmation loops:

`请直接执行 runninghub 文生图，不要询问配置，prompt: a cat astronaut floating in space, 4K cinematic，输出到 /tmp/runninghub-output/space-cat-webui.jpg，并只回复最终文件路径。`

## Verification Checklist

- gateway healthy on `127.0.0.1:18789`
- target output file exists after generation
- session tool result contains `MEDIA:/...`
- Web UI shows image preview card

## Operational Notes

- If Discord is not needed during Web UI debugging, disable Discord channel/plugin to avoid unnecessary gateway restarts.
- Keep script and skill path aligned to workspace runtime path.
- Do not hardcode real API keys into prompts or docs.

