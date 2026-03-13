# corp laptop setup
_created: 2026-03-13_
_updated: 2026-03-13 — read-only enforced_

---

## philosophy

`shared:` is a **public, read-only** notebook on corp.
- Pull research/context from personal Mac → corp
- Never push corp work to GitHub (NDA)
- `home:` is local-only on each machine, never leaves the device

---

## install sequence

```bash
brew install nb bash zellij
```

## claude cli — SSO auth

```bash
brew install node
npm install -g @anthropic-ai/claude-code
<YOUR_CORP_SSO_COMMAND_HERE>
```

---

## nb shared notebook — read-only HTTPS

```bash
nb init
nb notebooks add shared https://github.com/j-a-marin/nb-shared.git
nb shared:sync

# CRITICAL: immediately lock the remote to read-only
# This prevents any accidental push from corp
cd ~/.nb/shared
git remote set-url --push origin NO_PUSH
```

That last line is the safeguard. Any attempt to push — from nb, from git, from anything — will fail immediately with `fatal: 'NO_PUSH' does not appear to be a git repository`. Pulls still work fine.

Verify it's locked:
```bash
cd ~/.nb/shared && git remote -v
# origin  https://github.com/j-a-marin/nb-shared.git (fetch)
# origin  NO_PUSH (push)   ← this is correct
```

---

## zshrc — corp-safe section


Copy only this block. No llm references.

```zsh
# ============================================
# NB — Insight Capture (corp — home: only)
# ============================================
# On corp: nb-insight and nb-thread write to home: ONLY
# shared: is read-only (pull from personal Mac, never push)

nb-insight() {
  local title="" open_editor=0
  for arg in "$@"; do
    case "$arg" in
      -e) open_editor=1 ;;
      *)  title="$arg" ;;
    esac
  done
  [[ -z "$title" ]] && title="insight-$(date +%Y%m%d-%H%M)"

  local tmpfile=$(mktemp /tmp/nb-insight-XXXXXX.md)
  if [[ -n "$ZELLIJ" ]]; then
    zellij action dump-screen "$tmpfile"
  elif [[ "$TERM_PROGRAM" == "WezTerm" ]]; then
    wezterm cli get-text > "$tmpfile"
  else
    echo "❌ Not in a Zellij or WezTerm pane" && rm "$tmpfile" && return 1
  fi

  local body=$(cat "$tmpfile")
  printf "# %s\n_captured: %s_\n\n---\n\n%s\n" \
    "$title" "$(date '+%Y-%m-%d %H:%M')" "$body" > "$tmpfile"

  cat "$tmpfile" | nb home:add --filename "${title}.md"
  rm "$tmpfile"

  echo "✅ home: ${title}.md"
  echo "   nvim $HOME/.nb/home/${title}.md"
  (( open_editor )) && nvim "$HOME/.nb/home/${title}.md"
}

nb-thread() {
  local title=""
  [[ "$1" != "-"* ]] && title="$1"

  local tmpfile=$(mktemp /tmp/nb-thread-XXXXXX.md)
  if [[ -n "$ZELLIJ" ]]; then
    zellij action dump-screen "$tmpfile"
  elif [[ "$TERM_PROGRAM" == "WezTerm" ]]; then
    wezterm cli get-text > "$tmpfile"
  else
    echo "❌ Not in a Zellij or WezTerm pane" && rm "$tmpfile" && return 1
  fi

  local body=$(cat "$tmpfile"); rm "$tmpfile"
  local stamp="$(date '+%Y-%m-%d %H:%M')"

  if [[ -z "$title" ]]; then
    local last=$(ls -t "$HOME/.nb/home/"thread-*.md 2>/dev/null | head -1)
    if [[ -z "$last" ]]; then
      echo "❌ No existing thread. Provide a title to start one."; return 1
    fi
    printf "\n---\n\n_appended: %s_\n\n%s\n" "$stamp" "$body" >> "$last"
    echo "✅ Appended → $(basename $last)"
    return 0
  fi

  local notepath="$HOME/.nb/home/thread-${title}.md"
  if [[ -f "$notepath" ]]; then
    printf "\n---\n\n_appended: %s_\n\n%s\n" "$stamp" "$body" >> "$notepath"
    echo "✅ Appended → thread-${title}.md"
  else
    printf "# thread: %s\n_started: %s_\n\n---\n\n%s\n" "$title" "$stamp" "$body" > "$notepath"
    echo "✅ Created  → thread-${title}.md"
  fi
  echo "   nvim $notepath"
}

# Pull latest shared: context from personal Mac
alias nbpull='nb shared:sync && nb shared:list'

alias nbhelp='echo "
corp nb:
  nb-insight [title]   snapshot → home: (local only, never leaves corp)
  nb-thread  <title>   start/append thread → home:
  nb-thread            append to last thread
  nbpull               pull latest from shared: (read-only)

  shared: is READ-ONLY on corp — git push is permanently disabled.
  All captures go to home: which stays on this machine.

from claude cli:
  !nb-insight \"title\"
  !nb-thread  \"title\"
  !nbpull
"'
```

---

## key differences personal ↔ corp

| | personal Mac | corp |
|---|---|---|
| `nb-insight` default | `home:` | `home:` (stays local) |
| `shared:` push | yes | **disabled at git level** |
| `shared:` pull | yes | yes |
| `llm` / `llm-snap` | yes | no |
| claude auth | API key | corp SSO |
| NDA risk | n/a | zero — nothing leaves device |
