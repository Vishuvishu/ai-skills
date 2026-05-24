# ai-skills

A collection of skills for Claude — drop-in instruction sets that make Claude smarter at specific tasks.

---

## Available Skills

| Skill | What it does |
|---|---|
| [`node-security`](./skills/node-security/) | Getting scraped, spammed, or hit by bots? Use this skill and ask the Claude or any AI agent to harden your Node.js backend — from IP banning and rate limiting to CAPTCHA and secure headers. For turnstile you also need the changes in the frontend |

---

## What is a skill?

A skill is a folder with a `SKILL.md` file inside. The AI reads it at the start of a conversation and follows its instructions whenever the task is relevant — no manual activation needed.

---

## Installation

Pick the skills you want from the [`skills/`](./skills/) folder, then follow the steps for your tool.

- [Cursor, GitHub Copilot, Antigravity IDE — and any tool supporting `.agents/skills/`](#cursor-github-copilot-antigravity-ide)
- [Claude.ai](#claudeai)
- [IBM Bob](#ibm-bob)

---

### Cursor, GitHub Copilot, Antigravity IDE

These tools all follow the [Agent Skills open standard](https://agentskills.io) and auto-discover skills from `.agents/skills/` in your project root. The steps are the same for all of them.

**Project-level (recommended):**

Copy the skill folder(s) you want from [`skills/`](./skills/) into `.agents/skills/` in your project:

```
your-project/
└── .agents/
    └── skills/
        └── node-security/      ← paste the whole folder here
            ├── SKILL.md
            └── resources/
                └── ...
```

The agent picks them up automatically when you start a new conversation.

**Global (all projects):**

Do the same but under `~/.agents/skills/` instead.

**Tool-specific notes:**

These tools might also support the tools specific folders instead of `.agents`, but using the `.agents` is ideal way.
| Tool | Also checks | Global path |
|---|---|---|
| Cursor | `.cursor/skills/` | `~/.cursor/skills/` |
| GitHub Copilot | `.github/skills/`, `.claude/skills/` | `~/.copilot/skills/` |
| Antigravity IDE | — | `~/.gemini/antigravity/skills/` |

So if you already have a `.cursor/skills/` or `.github/skills/` folder, you can put skills there instead — it works the same way.

---

### Claude.ai

Claude.ai supports skills via **Projects**. It uses a single text box, so paste the `SKILL.md` contents rather than copying the folder.

1. Open [claude.ai](https://claude.ai) and create or open a **Project**
2. Click **Project settings** → **Custom instructions**
3. Open the `SKILL.md` of each skill you want, copy its contents, and paste into the box
4. Hit **Save**

Claude will apply the skills to every conversation in that project automatically.

> To use multiple skills, paste them one after another, separated by a blank line.

---

### IBM Bob

> **Note:** Skills require **Advanced mode** in Bob. Make sure it's enabled before starting.

Copy the skill folder(s) you want from [`skills/`](./skills/) into `.bob/skills/` in your project:

```
your-project/
└── .bob/
    └── skills/
        └── node-security/      ← paste the whole folder here
            ├── SKILL.md
            └── resources/
                └── ...
```

Bob will automatically activate the skill when relevant. If not you can also mention liek this in your prompt's start "read the skill-name ( the skills name only ) before proceeding"

**Global (all projects):** Do the same but under `~/.bob/skills/` instead.

> To skip Bob's activation prompt, go to **Settings → Auto-Approve → Enable "Always allow skills"**.

---

## Skill not activating?

If your tool isn't picking up a skill automatically, you can always trigger it manually by referencing the skill file directly in your message:

> *"Follow the instructions in `.agents/skills/node-security/SKILL.md` and review my app for security issues."*

Or just paste the contents of `SKILL.md` directly into the chat as context. Every tool supports this as a fallback.

---

## License

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)