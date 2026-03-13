# corp laptop setup
_created: 2026-03-13_

---

## install sequence

```bash
brew install nb bash zellij
# nb       — notes vault, no API key needed
# bash     — kills nb's bash version warning
# zellij   — terminal multiplexer
# NO llm   — needs personal API key, skip on corp
```

## claude cli — SSO auth (no API key needed)

```bash
brew install node
npm install -g @anthropic-ai/claude-code
claude /login
# opens browser → corp SSO → done
# never set ANTHROPIC_API_KEY on corp machine
```

## nb shared notebook — one-time setup

```bash
nb init
nb notebooks add shared git@github.com:j-a-marin/nb-shared.git
nb shared:sync
# verify:
nb shared:list
```

SSH key for GitHub — if corp machine doesn't have one:
```bash
ssh-keygen -t ed25519 -C "corp-nb-sync"
cat ~/.ssh/id_ed25519.pub   # add to github.com → Settings → SSH keys
ssh -T git@github.com       # verify
```

## zshrc — corp-safe section (no llm references)

Copy only this block from personal ~/.zshrc to corp ~/.zshrc.
DO NOT copy llm-snap, llm-log-on, llm-log-off (require personal API key).

```zsh
# ============================================
# NB — Insight Capture (corp-safe, no llm)
# ============================================

nb-insight() {
  local title="" open_editor=0 notebook="shared"  # default: shared on corp
  for arg in "$@"; do
    case "$arg" in
      -e) open_editor=1 ;;
      -p) notebook="home" ;;   # -p (personal) flag for home: notebook
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
  local title="" notebook="shared"  # default: shared on corp
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
    [[ "$notebook" == "shared" ]] && echo "   (run 'nb shared:sync' to push)"
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
corp nb commands (shared: is default notebook):
  nb-insight [title]       snapshot → shared:
  nb-insight [title] -p    snapshot → home: (personal, local only)
  nb-thread  <title>       start/append thread → shared:
  nb-thread  <title> -p    start/append thread → home:
  nb-thread                append to last thread
  nbsync                   push/pull shared notebook

from claude cli (SSO):
  !nb-insight \"title\"
  !nb-thread  \"title\"
  !nbsync
"'
```

## key difference from personal machine

| | personal Mac | corp laptop |
|---|---|---|
| default notebook | `home:` | `shared:` |
| `llm` available | yes | no |
| `llm-snap` | yes | no |
| `nb-insight` | yes | yes |
| `nb-thread` | yes | yes |
| `nb shared:sync` | yes | yes |
| claude cli auth | API key | SSO (`claude /login`) |
