# git bisect

Простыми словами: бинарный поиск коммита, в котором появился баг — отмечаешь хороший и плохой, git сам водит по середине.

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
