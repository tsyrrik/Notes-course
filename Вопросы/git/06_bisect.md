## Вопрос: git bisect

## Простой ответ
- Бинарный поиск коммита с багом: отметь хороший и плохой, git проведёт по серединам до виновника.

```bash
git bisect start
git bisect bad HEAD          # плохой коммит
git bisect good v1.2.0       # известный хороший
# на каждом шаге тестируешь, отмечаешь good/bad
git bisect good
git bisect bad
git bisect reset             # вернуться обратно
```

Можно автоматизировать скриптом: `git bisect run ./test.sh`.

## Ответ

### Что такое git bisect

`git bisect` — инструмент бинарного поиска коммита, который внёс баг. Вместо того чтобы вручную проверять десятки или сотни коммитов, bisect делит историю пополам и за O(log N) шагов находит виновника. Для 1000 коммитов нужно всего ~10 проверок.

### Алгоритм работы

1. Вы указываете «плохой» коммит (где баг есть) и «хороший» (где бага нет).
2. Git переключается на коммит посередине между ними.
3. Вы проверяете — работает или нет — и отмечаете `good` или `bad`.
4. Git снова делит оставшийся диапазон пополам.
5. Процесс повторяется, пока не будет найден первый «плохой» коммит.

### Ручной bisect

```bash
# Начать bisect
git bisect start

# Отметить текущий коммит как плохой
git bisect bad HEAD

# Отметить известный хороший коммит (тег, хэш, ветка)
git bisect good v2.0.0

# Git переключается на средний коммит, вы тестируете...
# Если баг есть:
git bisect bad

# Если бага нет:
git bisect good

# Git сужает диапазон и переключает на следующий...
# Повторяем good/bad, пока не найдём виновника

# Git выведет:
# abc1234 is the first bad commit
# commit abc1234
# Author: developer@example.com
# Date:   Mon Jan 15 10:30:00 2024
#     Рефакторинг обработки заказов

# Вернуться к исходному состоянию
git bisect reset

# Посмотреть лог bisect (все шаги)
git bisect log

# Воспроизвести bisect из лога
git bisect replay bisect-log.txt
```

### Автоматический bisect

Если есть скрипт/тест, который определяет наличие бага (exit code 0 = good, не 0 = bad), bisect можно полностью автоматизировать:

```bash
git bisect start
git bisect bad HEAD
git bisect good v2.0.0

# Автоматический запуск: скрипт возвращает 0 (good) или 1 (bad)
git bisect run ./test.sh

# Примеры скриптов для автоматизации:

# Запуск конкретного теста
git bisect run php vendor/bin/phpunit --filter=testOrderTotal

# Проверка, что файл содержит/не содержит строку
git bisect run grep -q "buggy_function" src/Order.php

# Собственный скрипт
git bisect run bash -c '
    composer install --quiet 2>/dev/null
    php vendor/bin/phpunit --filter=testLogin
'
```

### Специальные exit-коды для run

| Exit code | Значение |
|---|---|
| 0 | Коммит хороший (good) |
| 1-124, 126-127 | Коммит плохой (bad) |
| 125 | Коммит нельзя протестировать — skip |

```bash
# Скрипт с поддержкой skip (если коммит не компилируется)
#!/bin/bash
make build 2>/dev/null || exit 125   # skip если не собирается
./run_test.sh                         # 0=good, 1=bad
```

### Пропуск коммитов

Иногда средний коммит нельзя протестировать (не компилируется, сломан по другой причине):

```bash
# Пропустить текущий коммит
git bisect skip

# Пропустить диапазон коммитов
git bisect skip abc1234..def5678
```

### Bisect с терминами old/new

Bisect можно использовать не только для багов, но и для поиска любого изменения (например, когда производительность упала). Вместо `good/bad` можно использовать произвольные термины:

```bash
git bisect start --term-old=fast --term-new=slow
git bisect slow HEAD
git bisect fast v1.0.0
# ... проверяем производительность ...
git bisect fast    # быстро
git bisect slow    # медленно
```

### Практический пример: поиск регрессии

```bash
# 1. Тесты падают на main, но 100 коммитов назад всё работало
git bisect start
git bisect bad main
git bisect good main~100

# 2. Автоматический поиск с PHPUnit
git bisect run php vendor/bin/phpunit tests/Unit/OrderTest.php

# 3. Git находит виновный коммит за ~7 шагов (log2(100) ≈ 7)
# abc1234 is the first bad commit

# 4. Смотрим что изменилось
git show abc1234

# 5. Завершаем bisect
git bisect reset
```

### Советы

- Перед bisect убедитесь, что рабочий каталог чистый (`git stash` если нужно).
- Для автоматизации пишите детерминированные скрипты — bisect повторяет их многократно.
- `git bisect visualize` (или `git bisect view`) откроет `gitk` с текущим диапазоном поиска.
- После `bisect reset` вы вернётесь на ту ветку, где были до начала bisect.

## Примеры
```bash
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
```

## Доп. теория
- `git bisect run` резко ускоряет поиск, если тесты автоматизируемы.
- Всегда завершайте `git bisect reset`, чтобы вернуть репозиторий в исходное состояние.
