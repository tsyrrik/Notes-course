# Docker
- Контейнеризует приложение с зависимостями; образ (image) → контейнер (запущенный экземпляр).
- Основное: `Dockerfile` (рецепт), `docker-compose` (несколько сервисов), volume (данные вне контейнера), network (связь).
- Команды: `docker build/run/ps/stop/start/exec`.

```dockerfile
FROM php:8.3-cli
WORKDIR /app
COPY . .
RUN composer install --no-dev
CMD ["php", "artisan", "serve", "--host=0.0.0.0"]
```

```bash
docker build -t myapp .
docker run -p 8000:8000 myapp
```
