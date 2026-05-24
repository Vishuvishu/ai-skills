## Describe Your Skill

Please provide a brief overview of the skill you are adding or updating:
* **Skill Name**: 
* **Target Scenarios**: 
* **Supported Agents/IDE environments tested**: 

---

## Technical Checklist

Please verify that your submission adheres to the `ai-skills` standards:

- [ ] **Naming Conventions**: The folder name under `skills/` is entirely lowercase, hyphenated, and matches the YAML frontmatter `name` field exactly.
- [ ] **Frontmatter Description**: The `description` field is written in the third person, contains explicit agent trigger phrases, and is written to be proactive/pushy.
- [ ] **No Triggers in Body**: The body of the `SKILL.md` does *not* contain "when to use" instructions; all trigger logic is strictly confined to the frontmatter description block.
- [ ] **Keep it Lean**: The `SKILL.md` file is strictly under 500 lines. 
- [ ] **Resource Isolation**: Large configurations, massive templates, checklists, or extensive integration guides are placed under the `resources/` folder.
- [ ] **On-demand Loading**: The `SKILL.md` body references resource files with clear instructions on *when* the agent should read them rather than loading them upfront.
- [ ] **Actionable Rules**: Directives use imperative language ("Always do X", "Never do Y") and explain the *why* (context/rationale) so agents apply them correctly.
- [ ] **Secrets & Portability**: No environment-specific variables, paths, database credentials, or secret keys are hardcoded.
- [ ] **Registry Update**: The new skill has been registered in the **Available Skills** table in the main [README.md](../README.md) with a clear, concise summary of what it does.

---

## Validation & Testing

How did you test this skill? (e.g. Cursor, Claude Projects, Antigravity IDE, etc.) Please describe the prompts used and if the agent triggered and followed instructions successfully:
* 
