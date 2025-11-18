

[Gitleaks](https://github.com/gitleaks/gitleaks) ищет пароли, токены и ключи в коммитах.

---

## Установка и запуск

**Через Docker:**
```bash
docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect --source=/repo
````

**Через бинарник:**

```bash
brew install gitleaks
gitleaks detect --source .
```

---

## Настройка

Можно создать `.gitleaks.toml`, чтобы исключить ложные срабатывания.

---

## Применение в CI

Добавь шаг в GitHub Actions / GitLab CI до деплоя:

```yaml
- name: Gitleaks
  run: docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect --source=/repo --no-git
```

