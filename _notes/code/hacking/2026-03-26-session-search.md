---
title: Fuzzy-searching coding agent sessions
feed: show
tags: hacking
---

Coding agents don't offer a way to search past sessions by content. `session-search` lets you fuzzy-search across all your agent history, preview the full thread, and resume it directly.

It's a bash script that uses `jq` to extract user and assistant messages from session files, normalises them into `u:` / `a:` lines, and pipes everything into `fzf`. The preview pane shows the full conversation. Selecting a result opens that session in the right agent -- `pi --session` or `claude --resume` -- so you pick up where you left off.

Currently supports [pi](https://github.com/badlogic/pi-mono) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Dependencies are `bash`, `jq`, and `fzf`.

# Source

The script lives in [cosk](https://github.com/xrfr/cosk) under `bin/session-search`.

# TODO

- [ ] add screenshot
