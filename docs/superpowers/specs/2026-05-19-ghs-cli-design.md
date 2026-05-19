# `ghs` — gh substring search CLI

## Problem

GitHub search tokenizes on word boundaries. `gh search repos foo bar` returns repos matching "foo" OR "bar" as whole tokens, missing repos like "quxbarbazfoo" which contain both substrings embedded in compound/camelCase names.

## Solution

`ghs` — a bash script extension for `gh` that performs AND substring matching on repo metadata, with fzf interactive selection.

## Interface

```
ghs <term1> [term2] ... [termN]
```

Invoked via `zsh -c "ghs foo bar"`.

## Flow

1. Run `gh search repos <term> --sort stars --limit 200 --json name,owner,description,url,stargazersCount,pushedAt,repositoryTopics` for each term
2. Merge results, dedupe by URL
3. Client-side filter: keep repos where ALL terms appear as substrings (case-insensitive) in `name + description + topics`
4. Secondary filters: `stargazersCount > 1`, `pushedAt >= 2025-01-01`
5. Sort by stars desc, keep top 100
6. Pipe into fzf with tab-separated format: `#N  ★STARS  owner/repo  description<TAB>url`
7. On selection: extract URL, open via `gh repo view --web "$url"`, loop back to fzf
8. fzf stays open until explicit exit (Ctrl-C, Esc, q)

## Format

```
  #1  ★54321  owner/repo  Description text here	https://github.com/owner/repo
```

## Dependencies

- `gh` (GitHub CLI, authenticated)
- `jq`
- `fzf`

## Cache

Results cached to `~/.cache/ghs/<hash>.json` keyed by sorted terms hash. TTL: 5 minutes.

## File

`~/ghs/bin/ghs` — single executable bash script, 755.
