

| Команда | Что делает |
|----------|-------------|
| `git fetch` | скачивает новые коммиты с remote, **не сливая** их в текущую ветку |
| `git pull` | делает `fetch`, а затем merge (или rebase, если включено) |


## Примеры

```bash
git fetch origin main
git log main..origin/main
# посмотреть, что изменилось

git merge origin/main
# вручную влить обновления
````

Так безопаснее, чем `git pull`, потому что ты контролируешь merge.

---

## Настройка pull под rebase

```bash
git config --global pull.rebase true
```

Тогда `git pull` будет подтягивать изменения и применять твои коммиты поверх них, без merge-коммитов.

