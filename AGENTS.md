# AGENTS.md

Guidance for AI agents working in this repository.

## About this repo

This is the **signaturit-integration** Claude Code plugin. The manifest lives at
`.claude-plugin/plugin.json`; the skill lives under `skills/signaturit-integration/`.

## Versioning — bump on every plugin change

On **any** change to the plugin, update `version` in `.claude-plugin/plugin.json`
following [semantic versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`):

- **PATCH** (`x.y.Z`) — backwards-compatible fixes and small edits: metadata/manifest
  tweaks, icon, docs/README wording, reference-file corrections, eval adjustments.
- **MINOR** (`x.Y.0`) — backwards-compatible additions: new skill capability, new
  reference material, meaningfully expanded behavior.
- **MAJOR** (`X.0.0`) — backwards-incompatible changes: renaming/removing the skill,
  changing its trigger contract, or any change that breaks existing users' expectations.

Bump the version in the same change as the edit that warrants it, before committing.
