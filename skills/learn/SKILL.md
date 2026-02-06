---
description: Learn a new skill from the current conversation or interactive interview, and save it to ~/.claude/skills
---

# Learn a New Skill

Capture operational knowledge as a reusable Claude skill through an interactive interview.

## Workflow

### 1. Get the skill topic

Ask the user:
- **"What skill do you want to capture? Give me a short name and a one-sentence description."**

Use the name to derive a slug (lowercase, hyphens, e.g. `debug-k8s-oom`). Use the description for the YAML frontmatter `description` field.

### 2. Interview

Ask 2-4 questions to understand the skill. Adapt questions based on answers. Good starting questions:

- "What problem does this skill solve? When would you reach for it?"
- "Walk me through the key steps. What do you actually do?"
- "Any gotchas, common mistakes, or things that surprised you?"

If the current conversation already contains relevant context (e.g. you just finished debugging something together), reference it directly instead of asking redundant questions. Pull details from the conversation.

Keep it conversational. Stop interviewing when you have enough to write a useful skill.

### 3. Research (if needed)

After the interview, assess whether the skill needs external information that wasn't covered in the conversation. If so, research before drafting:

- **Web search**: look up official docs, CLI references, API details, best practices
- **Codebase exploration**: explore relevant code, config files, or project structure
- **Documentation**: read READMEs, man pages, or tool-specific guides

Skip this step if the interview and conversation context already provide everything needed. Ask the user if you're unsure whether research is needed.

### 4. Draft the skill

Generate a `SKILL.md` with:

```markdown
---
description: <one-line description for skill discovery>
---

# <Skill Title>

<Content based on interview - no enforced structure, adapt to the topic>
```

Let the content structure emerge naturally from what the user described. Use code blocks, commands, and concrete examples wherever possible. Keep it practical and actionable.

### 5. Review

Present the full draft to the user. Ask: **"Here's the draft. Want me to save it as-is, or would you like changes?"**

Iterate if they want changes. Do NOT save until the user approves.

### 6. Save

Write the approved skill to `~/.claude/skills/<slug>/SKILL.md`.

Confirm: **"Skill saved to `~/.claude/skills/<slug>/SKILL.md`. You can invoke it with `/<slug>`."**
