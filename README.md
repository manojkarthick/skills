# Custom agent skills

My collection of reusable agent skills for personal workflows.

## Available skills

- [`uv-python-cli`](skills/uv-python-cli/SKILL.md) — create a distributable Python CLI with `uv`, `uv_build`, a `src/` layout, and a `[project.scripts]` entry point.

Add future skills as separate directories under `skills/` and list them here.

## Install a skill

After publishing this repository to GitHub, install an individual skill with:

```bash
npx skills add manojkarthick/skills --skill SKILL_NAME
```

Replace `SKILL_NAME` with a directory name under `skills/`. The `skills` CLI supports Codex, Claude Code, Cursor, and other compatible agents.

For a manual installation, copy the selected skill directory into the skills directory used by your agent.

## Development

Validate a skill's metadata:

```bash
python /path/to/skill-creator/scripts/quick_validate.py skills/SKILL_NAME
```

Skills may include an `evals/evals.json` file with prompts and expectations for behavioral testing.
