# Fusion Claw Builder

An agent skill for Oracle Fusion AI Agent Studio. Use it as the final step after an agentic app is built: it adds Fusion Claw autonomous outcomes to that app by editing the app's artifacts in code, with no computer use.

- **Part A:** codes the read and action capabilities Claw will call into the app's workflows (CAPABILITY trigger, routing, calculation contract, idempotent actions with readback).
- **Part B:** codes the Autonomous Outcomes settings (goals, skills, additional autonomous agents, example tasks) into the app file, then saves and reads them back through the workspace CLI.

It also covers the tests to run in code and the Outcome Receipt checks to run on screen, and how to hand back the actual state.

## Requirements

- An AI Agent Studio workspace with its `aistudio` skill and bundled CLI.
- An agentic app that is already built.
- A test environment. Publishing stays a human step in Studio.

## Install

Clone into your agent's skills folder so the folder name is `fusion-claw-builder`:

```bash
# Codex
git clone <this repository URL> ~/.codex/skills/fusion-claw-builder

# Claude Code
git clone <this repository URL> ~/.claude/skills/fusion-claw-builder
```

Then ask the agent to "use $fusion-claw-builder" (Codex) or `/fusion-claw-builder` (Claude Code) on your built app.

## Files

- `SKILL.md`: the skill.
- `agents/openai.yaml`: display metadata for Codex.

## Status

Field names and behaviour were observed on AI Agent Studio release 26C. Items marked [UNVERIFIED] in the skill are not confirmed; verify everything against your current workspace skill before use. This is not an Oracle product and is not supported by Oracle.
