# corp — first connection to nb-shared
_created: 2026-03-13_

---

## assumptions

- `nb` already installed on corp (`brew install nb bash`)
- No prior nb setup on corp yet (or run `nb init` first if needed)

---

## step 1 — connect shared: (read-only, no SSH key needed)

```bash
nb init
nb notebooks add shared https://github.com/j-a-marin/nb-shared.git
nb shared:sync
```

## step 2 — lock push immediately (NDA — nothing leaves corp)

```bash
cd ~/.nb/shared
git remote set-url --push origin NO_PUSH

# verify
git remote -v
# origin  https://github.com/j-a-marin/nb-shared.git (fetch)
# origin  NO_PUSH (push)
```

## step 3 — read the full setup doc

```bash
nb shared:list
nb shared:show "corp laptop setup" --print
```

That doc has everything else — zshrc block, nvim setup, SSO auth.
Follow it from there.

---

## going forward on corp

```bash
nbpull          # pull latest from personal Mac (alias in zshrc)
nb shared:list  # see what's available
nb shared:show <id> --print  # read any note
```

Writes stay local: `nb-insight` and `nb-thread` write to `home:` only.
