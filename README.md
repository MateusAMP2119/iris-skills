# iris-skills

Claude Code skills for authoring against the [iris](https://github.com/MateusAMP2119/iris-catalog) engine.

Each top-level directory is one skill: a `SKILL.md` with YAML frontmatter
(`name`, `description`) that Claude Code loads on demand when the description
matches the task at hand.

## Skills

| Skill | What it covers |
| --- | --- |
| [`pipeline-authoring`](pipeline-authoring/SKILL.md) | Writing pipelines and lanes under the turn protocol — the no-time law, frame vocabulary, the fetch frame, declaration shape, provenance. References: [`lanes.md`](pipeline-authoring/references/lanes.md) (composers, order, the 2+ interlock, folder surfaces, lifetimes), [`scraping.md`](pipeline-authoring/references/scraping.md) (pace, backoff, endpoint probing). |

Skills use progressive disclosure: `SKILL.md` holds what is true for every
case, `references/` holds the per-case detail, read only when the task is
that case.

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
