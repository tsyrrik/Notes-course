## Вопрос: Fast-forward merge

## Простой ответ
- Слияние без merge-коммита, если целевая ветка не имеет своих коммитов.

```bash
git checkout main
git merge --ff-only feature   # обновит указатель main на коммиты feature без merge-коммита
```

Когда использовать: чистая линейная история. Если нужна явная отметка слияния — `--no-ff`.

## Ответ

### Что такое fast-forward merge

Fast-forward (FF) merge — это особый случай слияния, когда целевая ветка не имеет собственных коммитов после точки ветвления. В этом случае Git просто перемещает указатель целевой ветки вперёд на последний коммит сливаемой ветки. Merge-коммит не создаётся, история остаётся линейной.

### Когда возможен fast-forward

Fast-forward возможен только когда текущая ветка является прямым предком (ancestor) ветки, которую сливают. То есть все коммиты целевой ветки уже содержатся в сливаемой ветке:

```
# До merge — main не менялся после создания feature:
main:    A -- B
                \
feature:          C -- D -- E

# После fast-forward merge:
main:    A -- B -- C -- D -- E   (указатель main просто переместился)
```

Если main имеет свои коммиты, fast-forward невозможен:
```
# main изменился — fast-forward невозможен:
main:    A -- B -- F
                \
feature:          C -- D -- E
# Нужен merge-коммит или rebase
```

### Команды и флаги

```bash
# Обычный merge — Git сам решит: FF если можно, merge-коммит если нельзя
git checkout main
git merge feature

# Только fast-forward — если FF невозможен, merge отменяется
git merge --ff-only feature

# Запретить fast-forward — всегда создавать merge-коммит
git merge --no-ff feature

# Настройка по умолчанию для конкретной ветки
git config branch.main.mergeoptions "--no-ff"

# Глобальная настройка (все merge)
git config --global merge.ff false        # всегда --no-ff
git config --global merge.ff only         # всегда --ff-only
```

### --ff vs --no-ff vs --ff-only

| Флаг | Поведение | Merge-коммит |
|---|---|---|
| `--ff` (default) | FF если возможно, иначе merge-коммит | Только при необходимости |
| `--no-ff` | Всегда создаёт merge-коммит | Всегда |
| `--ff-only` | Только FF, иначе ошибка | Никогда |

### Визуальная разница в истории

```bash
# С fast-forward: линейная история
git log --oneline --graph
# * e5f6a7b (HEAD -> main) Добавить валидацию
# * c3d4e5f Создать форму
# * a1b2c3d Initial commit

# С --no-ff: видно, что была feature-ветка
git log --oneline --graph
# *   f6g7h8i (HEAD -> main) Merge branch 'feature'
# |\
# | * e5f6a7b Добавить валидацию
# | * c3d4e5f Создать форму
# |/
# * a1b2c3d Initial commit
```

### Когда что использовать

- **`--ff` (default)** — для простых случаев, когда не важна визуализация ветвления.
- **`--no-ff`** — когда важно видеть в истории, что была feature-ветка. Популярно в Git Flow. Merge-коммит служит «точкой отмены» — можно revert одним коммитом.
- **`--ff-only`** — в CI/CD пайплайнах и при работе с `main`, чтобы гарантировать, что ветка была rebase перед merge. Если FF невозможен — merge не произойдёт, и разработчик должен сначала rebase.

### Практический workflow

```bash
# Типичный workflow: rebase + ff-only
git checkout feature
git rebase main               # привести feature к актуальному main
git checkout main
git merge --ff-only feature   # гарантированно линейная история

# Git Flow: всегда --no-ff для feature веток
git checkout develop
git merge --no-ff feature/login
git branch -d feature/login
```

### Связь с pull/fetch

`git pull` по умолчанию делает fetch + merge. Если remote ветка ушла вперёд, а локальных коммитов нет — произойдёт fast-forward. Можно настроить pull на rebase вместо merge:

```bash
git pull --ff-only              # только если FF возможен
git pull --rebase               # rebase вместо merge
git config --global pull.ff only   # по умолчанию --ff-only для pull
```

## Примеры
```bash
git merge --ff-only feature
```

## Доп. теория
- Fast‑forward не создаёт merge‑коммит, поэтому история выглядит линейной.
- `--no-ff` полезен, если нужно видеть факт отдельной фичи в истории.
