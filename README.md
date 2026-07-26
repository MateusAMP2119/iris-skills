# iris-skills

Claude Code skills for authoring against the [iris](https://github.com/MateusAMP2119/iris-catalog) engine.

Each top-level directory is one skill: a `SKILL.md` with YAML frontmatter
(`name`, `description`) that Claude Code loads on demand when the description
matches the task at hand.

## Skills

| Skill | What it covers |
| --- | --- |
| [`pipeline-authoring`](pipeline-authoring/SKILL.md) | Writing pipelines and lanes under the turn protocol — declaration shape, the fetch frame, script-owned pacing, scraping etiquette, provenance. |

## Install

Skills live in `.claude/skills/` (per project) or `~/.claude/skills/` (all
projects). Symlink to keep them updatable from one checkout:

```sh
git clone https://github.com/MateusAMP2119/iris-skills.git
ln -s "$PWD/iris-skills/pipeline-authoring" ~/.claude/skills/pipeline-authoring
```

Or copy the directory in if you'd rather pin a version.

## Adding a skill

One directory, one `SKILL.md`, frontmatter with a `description` that names the
trigger cases — that description is the only thing Claude sees before deciding
to read the body, so write it for retrieval, not for prose.
