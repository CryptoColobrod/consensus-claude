# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1.0] - 2026-07-19

- Provider-agnostic external critic — any CLI, or manual paste-through to whatever model you
  have — was gemini-only before.
- Agents inherit the current session's model — was hardcoded to `opus` before.
- Verifiable Decision Record written to `./consensus-runs/` on by default.
- Install block mkdir fix — `~/.claude/agents/` was never created, breaking first-time installs.
- Cost warning and "When to use this skill" section added.
- Uninstall/update docs added.
- Community files added: CONTRIBUTING.md, weak-verdict issue template.
- Affiliation disclaimer added.

## [1.0.0] - 2026-07-19

Initial public release.
