# corp laptop setup
_created: 2026-03-13_

---

## philosophy

`shared:` notebook is a **public** GitHub repo.
Zero auth required to clone. No SSH keys, no tokens, no PATs.

Rules:
- never put personal info, keys, credentials, or sensitive research here
- inspect before you sync: `nb shared:show <id> --print` → `nb shared:sync`
- `home:` stays local-only on each machine, never touches GitHub

---

## install sequence

```bash
brew install nb bash zellij
# NO llm — needs personal API key, skip on corp
```

## claude cli — SSO auth

```bash
brew install node
npm install -g @anthropic-ai/claude-code
claude /login
# opens browser → corp SSO → done
```

## nb shared notebook — HTTPS, no auth needed

```bash
nb init
nb notebooks add shared https://github.com/j-a-marin/nb-shared.git
nb shared:sync
nb shared:list
```

That's it. No SSH key. No token. Pulls and pushes over HTTPS anonymously for reads.

> **Note on push from corp:** HTTPS pushes still need auth.
> For read-only access from corp, HTTPS works anonymously.
> To push from corp, use a fine-grained PAT scoped to this repo only:
> https://github.com/settings/tokens?type=beta
> Then: `git remote set-url origin https://j-a-marin:<TOKEN>@github.com/j-a-marin/nb-shared.git`
> Or just treat corp as read-only — capture notes locally, sync from personal Mac.

---

## zshrc — corp-safe section

Copy only this block. Skip everything llm-related.

```zsh
# ============================================
# NB — Insight Capture (corp-safe)
# ============================================

nb-insight() {
  local title="" open_editor=0 notebook="shared"
  for arg in "$@"; do
    case "$arg" in
      -e) open_editor=1 ;;
      -p) notebook="home" ;;
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

  cat "$tmpfile" | nb ${notebook}:add --filename "${title}.md"
  rm "$tmpfile"

  local notepath="$HOME/.nb/${notebook}/${title}.md"
  echo "✅ ${notebook}: ${title}.md"
  echo "   nvim $notepath"
  [[ "$notebook" == "shared" ]] && echo "   (run 'nb shared:sync' to push)"
  (( open_editor )) && nvim "$notepath"
}

nb-thread() {
  local title="" notebook="shared"
  for arg in "$@"; do
    case "$arg" in
      -p) notebook="home" ;;
      *)  title="$arg" ;;
    esac
  done

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
    local last=$(ls -t "$HOME/.nb/${notebook}/"thread-*.md 2>/dev/null | head -1)
    if [[ -z "$last" ]]; then
      echo "❌ No existing thread. Provide a title to start one."; return 1
    fi
    printf "\n---\n\n_appended: %s_\n\n%s\n" "$stamp" "$body" >> "$last"
    echo "✅ Appended → $(basename $last)"
    echo "   nvim $last"
    return 0
  fi

  local notepath="$HOME/.nb/${notebook}/thread-${title}.md"
  if [[ -f "$notepath" ]]; then
    printf "\n---\n\n_appended: %s_\n\n%s\n" "$stamp" "$body" >> "$notepath"
    echo "✅ Appended → thread-${title}.md"
  else
    printf "# thread: %s\n_started: %s_\n\n---\n\n%s\n" "$title" "$stamp" "$body" > "$notepath"
    echo "✅ Created  → thread-${title}.md"
  fi
  echo "   nvim $notepath"
  [[ "$notebook" == "shared" ]] && echo "   (run 'nb shared:sync' to push)"
}

alias nbsync='nb shared:sync'
alias nbhelp='echo "
corp nb (shared: default):
  nb-insight [title]     snapshot → shared:
  nb-insight [title] -p  snapshot → home: (local only)
  nb-thread  <title>     start/append thread → shared:
  nb-thread              append to last thread
  nbsync                 push/pull shared

from claude cli:
  !nb-insight \"title\"
  !nb-thread  \"title\"
  !nbsync
"'
```

---

## key differences personal ↔ corp

| | personal Mac | corp |
|---|---|---|
| default notebook | `home:` | `shared:` |
| push auth | SSH key | HTTPS PAT (or read-only) |
| `llm` / `llm-snap` | yes | no |
| `nb-insight` / `nb-thread` | yes | yes |
| claude auth | `ANTHROPIC_API_KEY` | SSO `claude /login` |
