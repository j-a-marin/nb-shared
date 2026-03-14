# corp setup

```bash
# 1. connect (nb init already done — skip it)
nb notebooks add shared https://github.com/j-a-marin/nb-shared.git

# 2. lock push immediately (NDA)
cd ~/.nb/shared && git remote set-url --push origin NO_PUSH

# 3. pull and verify
nb shared:sync
nb shared:list
```
