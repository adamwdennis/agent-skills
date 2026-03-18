# agent-skills

Open-source agent skills collection. Cross-agent compatible (Claude Code, Cursor, Codex, etc).

## Skill Structure

```
skills/{skill-name}/
├── SKILL.md              # Required: skill definition
└── templates/            # Optional: output templates, reference files
    └── *.md
```

Each skill is a self-contained directory under `skills/`. The directory name IS the skill name.

## SKILL.md Format

```yaml
---
name: skill-name                    # Required: kebab-case, matches directory name
description: >                      # Required: 1-3 sentence description
  What this skill does and when to use it.
user-invocable: true                # Required: true if called via /skill-name
argument-hint: "<arg> | --flag"     # Optional: shown in help/autocomplete
---
```

The body after frontmatter is the skill logic — instructions the agent follows when invoked. Write it as clear, sequential steps with concrete examples of expected output.

## Conventions

### Naming
- **Kebab-case**, verb-noun: `trace-dataflows`, `audit-dataflows`, `generate-schemas`
- Directory name must match frontmatter `name`

### Frontmatter
- `name` and `description` are required (CI validates this)
- `description` should be scannable — lead with the action verb
- `user-invocable: true` for skills triggered via `/skill-name`

### Skill Body
- Lead with a one-line summary of what the skill does
- Use `##` headers to break into logical sections
- Write steps as imperative instructions ("Read the manifest", "Dispatch agents")
- Include exact output format examples in fenced code blocks
- Specify error handling — what to do when things fail
- Reference templates via relative path (`templates/output-template.md`)

### Templates
- Place in `templates/` subdirectory within the skill
- Use placeholder syntax: `{variable_name}` for values the skill fills in
- Include comments explaining the structure

### Examples
- Repo-level `examples/` directory for shared artifacts (manifests, configs)
- Fully commented with all available fields shown
- Must parse as valid YAML/JSON (CI validates this)

## CI Validation

`.github/workflows/validate.yml` checks on every push/PR:
- Each `skills/*/` has a `SKILL.md` with valid YAML frontmatter
- Frontmatter contains `name` and `description`
- Files in `examples/` parse as valid YAML
- `README.md` exists

## Adding a New Skill

1. Create `skills/{skill-name}/SKILL.md` with frontmatter + body
2. Add templates in `skills/{skill-name}/templates/` if needed
3. Add example configs to `examples/` if the skill uses external config
4. Update `README.md` with skill name, description, and usage example
5. CI validates structure on PR
