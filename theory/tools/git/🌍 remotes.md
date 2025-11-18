Git позволяет работать сразу с несколькими удалёнными репозиториями.

---

## Добавление

```bash
git remote add origin git@github.com:user/project.git
git remote add gitlab git@gitlab.com:user/project.git
````

---

## Push в оба

```bash
git push origin main
git push gitlab main
```

```

---

## Просмотр

```bash
git remote -v
```

---

## Зачем нужно

- резервный репозиторий (например, GitLab CI + GitHub open source)
    
- зеркалирование для публичных и приватных клиентов
    

