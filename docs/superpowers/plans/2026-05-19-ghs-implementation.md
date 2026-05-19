# ghs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Single bash script `~/ghs/bin/ghs` that searches GitHub repos with AND substring matching via fzf

**Architecture:** Bash script wrapping `gh search repos` + `jq` filtering + `fzf` selection loop. Per-term queries merged client-side, filtered by substring match, sorted by stars.

**Tech Stack:** bash, gh, jq, fzf

---

### Task 1: Create script file

**Files:**
- Create: `~/ghs/bin/ghs`

- [ ] **Step 1: Create directory and write ghs script**

```bash
mkdir -p ~/ghs/bin
```

Write `~/ghs/bin/ghs`:

```bash
#!/data/data/com.termux/files/usr/bin/bash
set -euo pipefail

CACHE_DIR="${XDG_CACHE_HOME:-$HOME/.cache}/ghs"
CACHE_TTL=300
fzf_data=$(mktemp "${TMPDIR:-/tmp}/ghs_XXXXXX")

[ $# -ge 1 ] || { echo "Usage: ghs <term1> [term2] ..." >&2; exit 1; }

terms=("$@")
IFS=$'\n' sorted=($(sort <<<"${terms[*]}")); unset IFS
cache_key=$(echo -n "${sorted[*]}" | md5sum | cut -d' ' -f1)
cache_file="$CACHE_DIR/$cache_key.json"
mkdir -p "$CACHE_DIR"

now=$(date +%s)
if [ -f "$cache_file" ]; then
  mod=$(stat -c %Y "$cache_file")
  age=$((now - mod))
else
  age=$((CACHE_TTL + 1))
fi

if [ "$age" -gt "$CACHE_TTL" ]; then
  combined='[]'
  for term in "${terms[@]}"; do
    result=$(gh search repos "$term" --sort stars --limit 200 \
      --json fullName,description,url,stargazersCount,pushedAt 2>/dev/null || echo '[]')
    combined=$(jq -s 'add' <(echo "$combined") <(echo "$result"))
  done
  echo "$combined" | jq 'unique_by(.url)' > "$cache_file"
fi

jq -c '.[]' "$cache_file" | while read -r repo; do
  fullName=$(jq -r '.fullName' <<<"$repo")
  desc=$(jq -r '.description // ""' <<<"$repo")
  stars=$(jq -r '.stargazersCount' <<<"$repo")
  pushed=$(jq -r '.pushedAt' <<<"$repo")
  url=$(jq -r '.url' <<<"$repo")

  text=$(echo "$fullName $desc" | tr '[:upper:]' '[:lower:]')
  match_all=0
  for term in "${terms[@]}"; do
    term_lower=$(echo "$term" | tr '[:upper:]' '[:lower:]')
    if [[ "$text" == *"$term_lower"* ]]; then
      ((match_all++))
    fi
  done
  [ "$match_all" -eq "${#terms[@]}" ] || continue
  [ "$stars" -gt 1 ] || continue
  [[ "$pushed" > "2025-01-01" || "$pushed" == "2025-01-01" ]] || continue
  echo "$stars|$fullName|$desc|$url"
done | sort -t'|' -k1 -rn | head -100 | awk -F'|' '{n++; printf "  #%d  ★%s  %s  %s\t%s\n", n, $1, $2, $3, $4}' > "$fzf_data"

trap 'rm -f "$fzf_data"' EXIT

while true; do
  selected=$(fzf --ansi --with-nth=1..4 < "$fzf_data")
  [ -n "$selected" ] || break
  url=$(echo "$selected" | awk -F'\t' '{print $2}')
  gh repo view --web "$url" 2>/dev/null || termux-open-url "$url" 2>/dev/null || true
done
```

- [ ] **Step 2: Make executable**

```bash
chmod +x ~/ghs/bin/ghs
```

- [ ] **Step 3: Verify script is runnable**

```bash
ls -la ~/ghs/bin/ghs
file ~/ghs/bin/ghs
```

Expected: `~/ghs/bin/ghs` exists, executable, bash script.

### Task 2: Test invocation

- [ ] **Step 1: Run basic smoke test**

```bash
zsh -c "ghs golan" 2>&1 | head -5
```

Expected: Shows top repos matching "golan" in name/desc/topics (e.g. "golang", "goland"), using fzf.

- [ ] **Step 2: Run multi-term test**

```bash
zsh -c "ghs golang web" 2>&1 | head -5
```

Expected: Shows repos where both "golang" and "web" appear as substrings in name/desc/topics.
