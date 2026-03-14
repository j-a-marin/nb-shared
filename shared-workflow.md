# shared: curation workflow
_created: 2026-03-13_

The shared: repo is a curated, current snapshot — not a full sync.
Corp sees exactly what you choose to put there, nothing more.

---

## write locally first, always

```bash
nb-insight "topic"          # → home: only, never shared:
nb-thread  "topic"          # → home: only, never shared:
```

Inspect the note. Decide if it's worth sharing.

---

## selectively add to shared:

```bash
# copy a curated note from home: to shared:
cp ~/.nb/home/my-note.md ~/.nb/shared/

# stage and commit just that file — nb sync won't interfere
cd ~/.nb/shared
git add my-note.md
git commit -m "add my-note"
git push
```

## remove a file from corp's view:

```bash
cd ~/.nb/shared
git rm old-note.md
git commit -m "remove old-note"
git push
# corp machine: nb shared:sync → file disappears from their vault
```

## check exactly what corp will see before pushing:

```bash
cd ~/.nb/shared
git status              # what's staged vs unstaged
git diff HEAD           # what changed
ls -1                   # current working tree — this is what corp gets
git log --oneline -5    # recent history
```

---

## nb shared:sync is still safe for pull

`nb shared:sync` on corp does a `git pull` only (push is disabled).
On personal Mac, avoid `nb shared:sync` for the shared: notebook —
use direct git instead so you control exactly what gets committed.

---
