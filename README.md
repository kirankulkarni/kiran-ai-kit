# kiran-ai-kit

Personal Claude Code plugins and skills, packaged as a plugin marketplace.

The repo is a marketplace containing a single bundle plugin, `kiran-ai-kit`, which carries every
skill. Installing the plugin gets you all of them.

## Install

```bash
# in Claude Code
/plugin marketplace add kirankulkarni/kiran-ai-kit
/plugin install kiran-ai-kit@kiran-ai-kit
```

The repo is private, so `git` needs credentials that can read it — the `gh` CLI login or an SSH key
on the machine is enough.

To update later:

```bash
/plugin marketplace update kiran-ai-kit
```

## Skills

| Skill | What it does |
|---|---|
| [`gauntlet`](plugins/kiran-ai-kit/skills/gauntlet/SKILL.md) | Adversarial plan → build → review pipeline. A model from a different family (Codex) attacks the plan and then the code. Has a light tier (one review pass over a finished diff) and a full tier (five-stage pipeline with bounded critic rounds). |

### gauntlet

Two tiers, because independent review is cheap and planning loops are not:

- **Light** — you do the work as normal, then hand the finished diff to Codex once. Minutes, one
  `codex exec` call, no subagents. Fits any substantive change.
- **Full** — a five-stage pipeline (data assessment → plan → plan critique → build → code review →
  tests), capped rounds, explicit `APPROVE`/`REVISE` verdicts. Costs hours; only on explicit request.

Requires the [`codex` CLI](https://github.com/openai/codex) on `PATH` for the review stages. Without
it the skill still describes the method, but you lose the cross-family independence that is the whole
point.

Supporting material lives alongside the skill:

- `assets/` — prompt templates for the plan critic, the code reviewer, and the data assessment
- `references/operations.md` — full invocation details, read before stage 2
- `evals/` — trigger and behaviour evals from developing the skill

## Layout

```
kiran-ai-kit/
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest, lists the plugins
└── plugins/
    └── kiran-ai-kit/
        ├── .claude-plugin/
        │   └── plugin.json       # plugin manifest
        └── skills/
            └── gauntlet/
                ├── SKILL.md      # frontmatter (name, description) + body
                ├── assets/
                ├── references/
                └── evals/
```

## Local development

Skills are edited here and symlinked into `~/.claude/skills/` so changes take effect immediately,
with no reinstall step:

```bash
ln -s ~/Documents/personal-work/repos/kiran-ai-kit/plugins/kiran-ai-kit/skills/gauntlet \
      ~/.claude/skills/gauntlet
```

Don't do both — if the plugin is also installed on this machine, the same skill shows up twice.
Symlink for the machine you develop on; install the plugin everywhere else.

## Adding a skill

1. Create `plugins/kiran-ai-kit/skills/<name>/SKILL.md` with YAML frontmatter:

   ```yaml
   ---
   name: <name>
   description: <when Claude should reach for this — the triggering conditions matter more than the summary>
   ---
   ```

2. Put anything the skill loads on demand in `references/`, and anything it copies or fills in
   under `assets/`. Keep `SKILL.md` itself short; it is always in context once triggered.
3. Add a row to the Skills table above.
4. Bump `version` in `plugins/kiran-ai-kit/.claude-plugin/plugin.json`.

The [`skill-creator`](https://github.com/anthropics/claude-plugins-official) plugin scaffolds and
evaluates skills if you'd rather not do it by hand.
