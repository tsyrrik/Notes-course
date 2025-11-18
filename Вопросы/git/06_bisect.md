# git bisect

## Простыми словами
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
