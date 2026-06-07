# spawned skill

The agent-facing guide for deploying and managing infrastructure on [spawned.ai](https://spawned.ai), packaged as a Claude Code plugin.

## Install (Claude Code)

```
/plugin marketplace add spawned-ai/spawned-skill
/plugin install spawned@spawned
```

Then invoke it any time with `/spawned`, or just ask Claude to deploy something to spawned.

## Use without installing

Point any agent at the always-current guide:

```
Read https://spawned.ai/SKILL.md and help me set up infrastructure with Spawned
```

## Staying up to date

Update to the latest published version any time with:

```
/plugin marketplace update spawned
/plugin update spawned@spawned
```

If the CLI's behavior ever disagrees with the skill, that's usually the signal to
update — the skill itself prompts you to do so when it notices a mismatch.

## Releasing a new version (maintainers)

`/plugin update` installs a new version only when `version` is higher, so **every
release must bump `version` in `.claude-plugin/plugin.json`** (semver).

1. Edit `skills/spawned/SKILL.md`.
2. Bump `version` in `.claude-plugin/plugin.json`.
3. Commit and push to `main`.

## Layout

```
.claude-plugin/
  plugin.json        plugin manifest (name, version, hooks)
  marketplace.json   marketplace catalog (this repo is its own marketplace)
skills/spawned/
  SKILL.md           the skill (also served live at spawned.ai/SKILL.md)
```
