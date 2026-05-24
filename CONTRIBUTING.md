# Contributing to ai-skills

Thank you for contributing to `ai-skills`! This project is a curated, public collection of high-quality skills for AI agents (including Claude, Cursor, GitHub Copilot, IBM Bob, and Antigravity IDE).

By submitting a skill, you are helping developers automate complex workflows and get better, more predictable output from their agentic coding tools.

---

## What is a Skill?

A **skill** is a self-contained instruction set packaged in a folder with a `SKILL.md` file at its root. AI tools that support the Agent Skills open standard automatically read these files and activate them when the user's task or context matches the skill's triggers.

---

## Workflow for Adding a New Skill

```
1. Conceptualize  ──►  2. Bootstrap (Skill Creator)  ──►  3. Refine SKILL.md  ──►  4. Add Resources  ──►  5. Register & Submit
   Define target          Create base skeleton               Apply agent-optimized     Move deep detail       Update README.md
   use cases & triggers   using Anthropic's tool             imperatives & "why"       to resources/          and submit a PR
```

### 1. Bootstrap Using the Anthropic Skill Creator
To save time and ensure your initial structure conforms to AI-native standards, we highly recommend bootstrapping your new skill using the **[Anthropic Skill Creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator)**. 

If you are using Claude or an AI agent that has access to the skill-creator, you can simply instruct the agent:
> *"Use the Anthropic Skill Creator skill to bootstrap a new skill for [your skill concept]."*

It will help you auto-generate a structured starting skeleton matching best practices.

### 2. Folder Structure
Every skill must reside under the `skills/` directory and use the following layout:

```
skills/
└── skill-name/          ← Lowercase, hyphenated (matches the frontmatter name)
    ├── SKILL.md         ← REQUIRED: Core prompt, triggers, and key directives
    └── resources/       ← OPTIONAL: For deep details, config files, guides
        ├── guide.md     ← Loaded by the agent on-demand, not upfront
        └── config.json
```

---

## Formatting Guidelines for `SKILL.md`

Your `SKILL.md` file must adhere to these strict rules to ensure AI agents parse, trigger, and execute it effectively:

### 1. Frontmatter
Every `SKILL.md` starts with a YAML frontmatter block containing `name` and `description`:

```yaml
---
name: skill-name
description: A detailed, pushy description in the third person explaining when the agent should trigger this skill. Include specific user trigger words and scenarios.
---
```

#### Rules for the `description` Field:
* **Write in Third Person**: Start with an active verb (e.g. *"Implement, audit, and debug..."* or *"Analyzes genomic data..."*).
* **Include Specific Trigger Phrases**: List phrases users are highly likely to say (e.g., *"use when user mentions X, Y, or Z"*).
* **Be Pushy & Clear**: AI agents tend to under-trigger skills. Make the criteria highly explicit and broad enough to capture relevant context (e.g., *"trigger even if the user does not mention the term security explicitly"*).
* **No "When to Use" in the Body**: All routing and triggering information must reside inside the description frontmatter. The body should only contain active directives.

### 2. Keep the Core Prompt Under 500 Lines
* Keep `SKILL.md` concise (strictly **under 500 lines**).
* If you have complex guides, configuration files, lists of regex patterns, or code templates, move them to the `resources/` folder.
* **Why**: Loading massive context upfront wastes context window and confuses the agent. Reference resource files from `SKILL.md` so the agent reads them **on-demand** only when executing that specific sub-task.

### 3. Imperative and Explanatory Instructions
* Write your rules using imperative language (e.g., *"Use X"*, *"Always do Y"*, *"Never do Z"*).
* **Always explain the WHY**: Agents apply rules much more successfully when they understand the context and reason behind them.
  * *Bad*: "Set X-Content-Type-Options to nosniff."
  * *Good*: "Always set `X-Content-Type-Options: nosniff` because it prevents browsers from MIME-sniffing a response away from the declared content-type, blocking script injection attacks."

### 4. General Portability
* Do **not** hardcode environment secrets, personal credentials, or project-specific directory paths.
* Keep skills generic enough to be dropped into any workspace using that technology stack.

---

## Submission Checklist

Before opening a Pull Request, verify the following:

- [ ] Folder name matches the frontmatter `name` exactly (lowercase, hyphenated).
- [ ] Frontmatter `description` contains explicit, agent-optimized trigger phrases in third person.
- [ ] No "when to use" instructions are written inside the `SKILL.md` body.
- [ ] `SKILL.md` is under 500 lines.
- [ ] Complex implementations or large datasets are split into the `resources/` folder.
- [ ] All resource files are referenced in `SKILL.md` with explicit instructions on when the agent should read them.
- [ ] Instructions use the imperative mood and explain the *why* behind core rules.
- [ ] Added your new skill to the **Available Skills** table in the [README.md](./README.md).
