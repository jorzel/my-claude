---
description: Scan a project and recommend which Claude skills to symlink into the project's .claude/skills/
---

# Skill Picker

Pick the right subset of skills for a project. Keep it lean — 5-10 skills max.

## Sources

1. **Local skills**: `.claude/skills/` in the project — project-specific, not symlinked
2. **Global skills**: `~/.claude/skills/` — your personal, hand-crafted skills
3. **Awesome skills repo**: `~/Coding/antigravity-awesome-skills/skills/` — community catalog

Local skills are always kept. Global skills take priority over awesome skills when both cover the same topic.

## Workflow

### 1. Scan the project

Analyze the project to understand:

- **Tech stack**: languages, frameworks, runtimes (check `go.mod`, `package.json`, `pyproject.toml`, `Cargo.toml`, `Makefile`, `Dockerfile`, etc.)
- **Architecture**: monolith vs microservices, DDD patterns, API style (REST/gRPC), infrastructure (k8s, Docker)
- **Workflow**: CI/CD setup, testing patterns, documentation conventions

### 2. Inventory skills

- **Local**: list skills already in `.claude/skills/` that are real directories (not symlinks). These are kept as-is.
- **Global**: list skills from `~/.claude/skills/` (read descriptions from YAML frontmatter)
- **Awesome**: list skills from `~/Coding/antigravity-awesome-skills/skills/` (read descriptions from YAML frontmatter)
- Deduplicate — if multiple sources cover the same topic, prefer local > global > awesome

### 3. Recommend

For each non-local skill, assess relevance across three dimensions:

| Dimension | Match criteria |
|-----------|---------------|
| **Tech stack** | Skill targets a language/framework used in this project |
| **Architecture** | Skill supports patterns present in the codebase |
| **Workflow** | Skill supports development/operational workflows used here |

Present a table:

| Skill | Source | Status | Reason |
|-------|--------|--------|--------|
| `skill-name` | local | Already present | — |
| `skill-name` | global | Recommended | Brief explanation |
| `skill-name` | awesome | Recommended | Brief explanation |

Cap total (local + recommended) at 10. If fewer than 5, note skill gaps worth filling with `/learn` as local skills.

### 4. Confirm with user

Ask the user to approve or adjust the selection before making any changes.

### 5. Symlink approved skills

```bash
mkdir -p .claude/skills
# For global skills:
ln -s ~/.claude/skills/<skill-slug> .claude/skills/<skill-slug>
# For awesome skills:
ln -s ~/Coding/antigravity-awesome-skills/skills/<skill-slug> .claude/skills/<skill-slug>
```

Skip skills that are already present (local or already linked). Report what was linked.

### 6. Summary

Print the final list of project skills (local + symlinked) and any suggested gaps worth filling with `/learn`.
