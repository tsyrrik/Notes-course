
Позволяет забрать конкретный коммит из другой ветки.

```bash
git cherry-pick <commit-hash>
````

Можно перенести несколько:

```bash
git cherry-pick a1b2c3d e4f5g6h
```

Полезно для hotfix'ов: вытащить нужный коммит из feature и применить в main.