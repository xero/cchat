```
 ┌──────────▄▄▄▄▄▄▄─┐
 │  ▄▄▄▄▄▄  █cchat█ │
 │ ▄█~██~█▄ ▀█▀▀▀▀▀ │
 │  ▀█▀▀█▀  ▀       │
 └──────────────────┘
```

# cchat

a terminal interface for [claude code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview) that uses your existing plan subscription. no api tokens needed.

cchat wraps the `claude` cli in a prompt_toolkit shell with vi keybindings, session persistence, file injection, and a planning workflow that produces task files for autonomous agents.

## why

claude code's interactive mode is great, but it takes over your terminal. cchat gives you a persistent chat session you control — attach files, switch modes, plan tasks, hand them off to agents, all without leaving your shell.

the key insight: `claude --print` lets you have conversations using your plan tokens, not api credits. cchat manages the sessions, context, and workflow around that.

## install

```sh
# deps
pip install prompt_toolkit

# claude code (if you don't have it)
npm install -g @anthropic-ai/claude-code

# cchat itself — it's a single file
curl -o ~/.local/bin/cchat https://raw.githubusercontent.com/xero/cchat/main/cchat.py
chmod +x ~/.local/bin/cchat
```

## usage

```sh
cchat                    # interactive session (named "default")
cchat myproject          # named session
cchat myproject --plan   # start in plan mode
cchat -a "quick question" # one-shot, no session
echo "explain this" | cchat -a - --file src/app.js  # pipe + file
```

### profiles

cchat supports multiple claude accounts via profiles. set `CCHAT_PROFILE=work` to switch, or edit `~/.claude_chat/config.json`:

```json
{
  "profiles": {
    "personal": { "bin": "/usr/local/bin/claude", "env": {} },
    "work": {
      "bin": "/usr/local/bin/claude",
      "env": { "CLAUDE_CONFIG_DIR": "~/.claudework" }
    }
  },
  "default_profile": "personal"
}
```

the `env` dict is merged into the subprocess environment, so you can point each profile at a different claude config directory with separate auth.

## modes

### chat

the default. fast, no tool access, just `--print`. good for questions, reviews, quick back-and-forth.

### plan

`/plan` — read-only access to your codebase via `--permission-mode plan`. the agent can read files, grep, check versions, but can't write anything. use this for researching and designing tasks.

### plan (danger)

`/plan danger` — full tool access via `--dangerously-skip-permissions`. the agent can install packages, write test scripts, run your test suite, fetch docs from the web — everything it needs to thoroughly validate a plan before writing it.

> **⚠ danger mode should only be used inside a sandbox.** a devcontainer, docker sandbox, or vm where the blast radius is contained. see [sandboxing](#sandboxing) below.

## commands

| command | description |
|---|---|
| `/attach <path>` | attach file(s) or zip to next message |
| `/skill <path>` | inject a SKILL.md into next message |
| `/system [txt]` | show or set system prompt |
| `/plan [danger]` | switch to planning mode |
| `/chat` | switch to chat mode |
| `/handoff [name]` | write task.md when plan is ready |
| `/compact` | reset session, start fresh |
| `/clear` | alias for /compact |
| `/sessions` | list saved sessions |
| `/usage` | session stats |
| `/help` | show help |
| `/q` `/quit` `/exit` | save and quit |

all commands tab-complete. type `/` then tab to see the full list with descriptions.

## keybindings

| key | action |
|---|---|
| `ctrl+g` | send message ("go") |
| `enter` | newline (insert mode) or submit /command |
| `F4` | toggle vi/emacs mode |
| `ctrl+c` | interrupt running command |
| `ctrl+d` | save and quit |

vi mode is on by default. the prompt shows `❯` in insert mode and `❮` in normal mode. `dd`, `ciw`, `yy`, `p` — all the usual vi motions work in the input buffer.

## the planning workflow

the core workflow cchat is built for:

1. **chat** — discuss the problem, get a code review, explore options
2. **plan** — switch to `/plan` (or `/plan danger` in a sandbox). the agent reads your codebase, installs test deps, validates assumptions, works out the approach with you
3. **handoff** — run `/handoff` when the plan is solid. the agent writes a `task.md` with numbered steps, gate tests, and a definition of done
4. **execute** — hand the task to an autonomous agent: `claude -p "read task.md and begin" --dangerously-skip-permissions`

the planning agent does the intellectual work. the executing agent follows instructions. this separation means the executor doesn't need to make judgment calls — everything was decided during planning.

### task format

`/handoff` produces a structured task file:

- **goal** — what and why
- **orientation** — files to read first, with exact paths
- **steps** — numbered, imperative, each with a gate test
- **definition of done** — verifiable checklist
- **what not to do** — specific prohibitions
- **orchestration** — (optional) sub-agent coordination for parallel work
- **raising an issue** — when to stop and ask instead of guess

### orchestration

for complex tasks, the task file can instruct the executing agent to act as a project manager — spawning sub-agents for independent workstreams, monitoring gate tests, and merging results. no extra tooling needed; the orchestration instructions are just part of the task.

## sandboxing

danger mode (`/plan danger`) gives the planning agent full system access. **do not run this on your host machine.** use one of:

- [anthropic's devcontainer](https://code.claude.com/docs/en/devcontainer) — the official reference setup
- [trail of bits devcontainer](https://github.com/trailofbits/claude-code-devcontainer) — security-focused sandbox with `devc` cli
- [docker sandboxes](https://docs.docker.com/ai/sandboxes/agents/claude-code/) — docker's built-in claude code sandbox support

the sandbox provides filesystem isolation and network firewall rules. cchat runs inside the container, claude runs inside the container, and your host machine is untouched.

## files

```
~/.claude_chat/
├── config.json          # profiles and settings
├── default.json         # session state (per session name)
├── default.history      # input history (per session name)
└── myproject.json       # another session
```

sessions persist across restarts. `/compact` clears the session state but keeps the history file. sessions are keyed by name — `cchat foo` and `cchat bar` are independent.

## status bar

the bottom toolbar shows live session state:

- mode icon (green ≡ for chat, cyan ✱ for plan, red ✱ for danger)
- session name, profile, code session id
- turn count, skill count
- armed context count (`ctx:3` when files are attached)
- quick reference (`^g send`, `/help`)

## CLAUDE.md

if a `CLAUDE.md` or `.claude/CLAUDE.md` exists in the current directory, cchat loads it and appends it to the system prompt via `--append-system-prompt`. this lets you give project-specific context to every message without attaching it manually.

## license

[CC0](https://creativecommons.org/publicdomain/zero/1.0/) — public domain. do whatever you want.

## author

xero harrison [x-e.ro](https://x-e.ro) · [github.com/xero](https://github.com/xero)
