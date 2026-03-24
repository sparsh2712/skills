# Skills

A collection of Claude skills for software development.

## Available Skills

| Skill | Description |
|-------|-------------|
| [fastapi-backend-ddd](skills/fastapi-backend-ddd/) | Domain-Driven Design implementation guidelines for async Python backends |
| [tdd-for-ddd](skills/tdd-for-ddd/) | Test-Driven Development workflow for DDD backends — RED-GREEN-REFACTOR with fakes, no mocks |

## Install

```bash
/plugin marketplace add sparsh2712/skills
```

Then install individual skills or the full collection as needed.

## Adding a New Skill

1. Create a new directory under `skills/`:
   ```
   skills/my-new-skill/
   └── SKILL.md
   ```

2. Add a `SKILL.md` with YAML frontmatter (see `template/SKILL.md` for the skeleton):
   ```yaml
   ---
   name: my-new-skill
   description: What the skill does and when to trigger it.
   ---
   # My New Skill
   Instructions here.
   ```

3. Optionally add a `resources/` directory for supplementary docs the skill references.

4. Register the skill in `.claude-plugin/marketplace.json` by adding an entry to the `plugins` array:
   ```json
   {
     "name": "my-new-skill",
     "description": "Short description",
     "source": "./skills/my-new-skill",
     "strict": false,
     "skills": ["./skills/my-new-skill"]
   }
   ```

## Structure

```
skills/
  fastapi-backend-ddd/   # DDD for async Python backends
    SKILL.md
    resources/
  tdd-for-ddd/           # TDD workflow for DDD backends
    SKILL.md
    resources/
template/
  SKILL.md               # Skeleton for new skills
```
