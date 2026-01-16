# dwim

Do What I Mean. A self-learning CLI shim.

## The Problem

Agents try intuitive commands that don't exist. `khal delete` seems obvious but isn't a real command. Each failed attempt wastes tokens and requires approval loops. Building bespoke CLI wrappers for every tool is tedious and you don't know what surface area you actually need until you've been using things for a while.

## The Solution

A shim that interprets intent, executes what you meant, and gradually compiles itself into native implementations based on actual usage patterns.

```bash
dwim calendar delete "Test event"
# → interprets intent
# → executes: rm /path/to/matching.ics && vdirsyncer sync
# → logs the mapping
# → (optionally) opens PR to add native `calendar delete` subcommand
```

Day 1: Everything goes through LLM interpretation.
Day 30: Common paths are native bash, LLM only fires for novel intents.

## Layers

### Layer 1: Immediate Execution
- Typo correction (like `thefuck`)
- Clear intent → just do it
- All executions logged for pattern analysis

### Layer 2: Scope Awareness
Track calling PID, cwd, git remote. Distinguish:
- **Project-local** (`dwim reset-containers`) → `./.dwim/`
- **User-level** (`dwim calendar add`) → `~/.config/dwim/`
- **Upstream-universal** (`dwim usage`) → PR to this repo

### Layer 3: Async Clarification
Can't do interactive prompts (agents can't respond). Instead:
```
$ dwim calendar something-ambiguous
dwim: intent unclear
  clarifying questions written to: /tmp/dwim-abc123
  run: dwim retry /tmp/dwim-abc123 '{"answers": ...}'
  (clarification will be cached for future requests)
```

### Layer 4: Self-Optimization
- Periodic scans of usage logs
- Identify patterns, reduce LLM calls
- Auto-generate native implementations for high-frequency intents
- Suggest promotions: "you've called dwim 50 times for calendar ops, consider adding `cal` to your CLAUDE.md"

## Design Principles

**Discovery mechanism, not final architecture.** dwim exists to reveal what tools you actually need. Once patterns stabilize, graduate them to real tools.

**Gradual compilation.** LLM interpretation is expensive. Every successful interpretation should make the next call cheaper. The tool gets faster as it learns.

**Scope-aware by default.** Not everything is universal. `dwim` in your exocortex project might mean different things than `dwim` in fort-nix.

**Logging over guessing.** When you don't know what interface to build, log what agents try to use and let the interface emerge.

## Non-Goals

- Not a framework
- Not a replacement for well-defined tools
- Not trying to be clever about everything forever

The goal is to eventually need dwim less, not more.

## v0.1 Sketch

```bash
#!/usr/bin/env bash
set -euo pipefail

DWIM_LOG="${DWIM_LOG:-$HOME/.local/share/dwim/usage.log}"
mkdir -p "$(dirname "$DWIM_LOG")"

# Log the attempt
echo "$(date -Iseconds) [$$] $(pwd) $*" >> "$DWIM_LOG"

# Check for native implementation first
if [[ -x "$HOME/.config/dwim/commands/$1" ]]; then
  exec "$HOME/.config/dwim/commands/$1" "${@:2}"
fi

if [[ -x "./.dwim/commands/$1" ]]; then
  exec "./.dwim/commands/$1" "${@:2}"
fi

# Fall back to LLM interpretation
claude -p "You are dwim (Do What I Mean), a CLI intent interpreter.

The user invoked: dwim $*
Working directory: $(pwd)

If the intent is clear, execute it and return the result.
If the intent is ambiguous, write clarifying questions to a temp file and return instructions for 'dwim retry'.
If you can infer a reusable pattern, note it for potential native implementation.

Be concise. Execute, don't explain."
```

## Usage Patterns to Watch For

Once running, `dwim usage` should reveal:
- Most common command prefixes (calendar, git, docker, etc.)
- Scope distribution (what's project-local vs universal)
- Clarification frequency (what intents are ambiguous)
- Graduation candidates (what should become native tools)

## Related

- [thefuck](https://github.com/nvbn/thefuck) - typo correction
- The "desire-path logging" concept from Wicket
- Principle of Least Surprise (Matz/Ruby philosophy)
