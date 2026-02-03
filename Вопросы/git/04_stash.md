## Вопрос: git stash

## Простой ответ
- Временно прячет незакоммиченные изменения в стек, чтобы переключиться; потом можно вернуть.

```bash
git stash push -m "wip login"   # сохранить
git stash list                  # посмотреть
git stash pop                   # применить и удалить из списка
git stash apply stash@{2}       # применить конкретный, не удаляя
git stash drop stash@{0}        # удалить
```

Опции: `-u` (включить неотслеживаемые файлы), `-k`/`--keep-index` (оставить индекс).

## Ответ

### Что такое stash

`git stash` — механизм временного сохранения незакоммиченных изменений в специальный стек. Это позволяет быстро переключить контекст: отложить текущую работу, перейти на другую ветку, сделать что-то срочное, а затем вернуть изменения. Stash работает по принципу стека (LIFO — Last In, First Out): последний сохранённый stash извлекается первым.

### Как stash работает внутри

При выполнении `git stash` Git создаёт два (или три) специальных коммита, которые не принадлежат ни одной ветке:
1. Коммит с состоянием индекса (staging area).
2. Коммит с состоянием рабочего каталога (working directory).
3. (Опционально, с `-u`) коммит с неотслеживаемыми файлами.

Эти коммиты хранятся в `refs/stash` и доступны через `stash@{N}`.

### Полный набор команд

```bash
# Сохранить изменения (tracked файлы)
git stash
git stash push                      # эквивалент

# Сохранить с описанием
git stash push -m "WIP: рефакторинг авторизации"

# Сохранить включая untracked файлы
git stash push -u -m "с новыми файлами"

# Сохранить ВСЁ, включая игнорируемые файлы (.gitignore)
git stash push -a -m "включая .env"

# Сохранить только конкретные файлы
git stash push -m "только модели" -- src/Models/User.php src/Models/Order.php

# Сохранить, оставив индекс (staged файлы) на месте
git stash push --keep-index -m "только unstaged"

# Интерактивный stash: выбрать конкретные hunks
git stash push -p

# Посмотреть список stash
git stash list
# stash@{0}: On feature: WIP: рефакторинг авторизации
# stash@{1}: On main: hotfix prep

# Применить последний stash и удалить из стека
git stash pop

# Применить конкретный stash без удаления
git stash apply stash@{2}

# Посмотреть содержимое stash (diff)
git stash show                      # краткий
git stash show -p                   # полный diff
git stash show -p stash@{1}         # конкретный stash

# Удалить конкретный stash
git stash drop stash@{0}

# Очистить все stash
git stash clear

# Создать ветку из stash (если stash конфликтует с текущим состоянием)
git stash branch new-feature stash@{0}
```

### Типичные сценарии

**Срочный hotfix во время работы над фичей:**
```bash
git stash push -m "WIP: фича X"
git checkout main
git checkout -b hotfix/critical-bug
# ... фикс ...
git commit -m "Fix critical bug"
git checkout feature/x
git stash pop
```

**Перенос незакоммиченных изменений на другую ветку:**
```bash
# Случайно начал работу не на той ветке
git stash
git checkout correct-branch
git stash pop
```

**Проверка чистого состояния (например, запуск тестов):**
```bash
git stash push --keep-index -m "проверка staged"
# Тесты запускаются только со staged изменениями
./vendor/bin/phpunit
git stash pop
```

### Важные нюансы

- **Stash привязан к репозиторию**, а не к ветке. Можно сохранить на одной ветке, применить на другой.
- **Конфликты при pop/apply.** Если текущее состояние конфликтует со stash, Git покажет конфликты. После разрешения stash не удаляется автоматически — нужно `git stash drop`.
- **Stash не бесконечный.** Не используйте stash как долговременное хранилище — для этого есть коммиты и ветки. Stash удобен для краткосрочного переключения контекста.
- **`pop` = `apply` + `drop`.** Если при `pop` возникнут конфликты, stash не удалится (drop не выполнится). Это безопасно.

### Stash vs commit vs branch

| Инструмент | Когда использовать |
|---|---|
| `git stash` | Быстрое переключение контекста (минуты/часы) |
| WIP-коммит | Если нужно запушить незавершённую работу |
| Отдельная ветка | Длительная параллельная работа |

## Примеры
```bash
git stash push -m "wip"
git checkout main
git stash pop
```

## Доп. теория
- `stash` — временное решение; для долговременной работы лучше ветка или WIP‑коммит.
- `stash` хранится в `refs/stash` и может быть восстановлен через reflog.
