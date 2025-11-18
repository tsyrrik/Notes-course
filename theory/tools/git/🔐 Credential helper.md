

Полезен если работаешь с репозиторием не через ssh, а через https.

Чтобы не вводить пароль при каждом push:

```bash
git config --global credential.helper store
```

или

```bash
git config --global credential.helper cache
```

(второй хранит токены временно, по умолчанию 15 минут).