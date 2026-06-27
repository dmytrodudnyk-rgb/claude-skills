# claude-skills

Personal [Claude Code](https://claude.com/claude-code) skills, packaged as a plugin
marketplace so they sync across machines. Each skill is its own plugin.

## Layout

```
.claude-plugin/marketplace.json        # marketplace manifest (name: claude-skills)
plugins/
  adversarial-pr-review/               # a plugin == one skill
    .claude-plugin/plugin.json
    skills/
      adversarial-pr-review/           # SKILL.md + supporting files
```

## Install on a machine

```
/plugin marketplace add <owner>/claude-skills
/plugin install adversarial-pr-review@claude-skills
```

Replace `<owner>` with the GitHub owner of this repo.

## Update everywhere

After pushing changes:

```
/plugin marketplace update claude-skills
/plugin update adversarial-pr-review@claude-skills
```

## Add a new skill

1. `mkdir -p plugins/<skill-name>/skills/<skill-name>` and add a `SKILL.md`.
2. Add `plugins/<skill-name>/.claude-plugin/plugin.json` (name + description + author).
3. Append an entry to the `plugins` array in `.claude-plugin/marketplace.json`
   with `"source": "./plugins/<skill-name>"`.
4. Commit, push, then `/plugin install <skill-name>@claude-skills` on each machine.
