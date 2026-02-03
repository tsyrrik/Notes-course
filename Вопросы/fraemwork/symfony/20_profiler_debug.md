## Вопрос: Profiler и debug
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
в dev-окружении работает Web Profiler Toolbar и страница профайлера; можно смотреть запросы/SQL/логи/почту. Панель подключает `framework.profiler` и `debug` пакеты.

## Ответ
```bash
php bin/console debug:router
php bin/console debug:container
php bin/console debug:config framework
```

## Примеры

1. В профайлере найти N+1 запросы в вкладке «Doctrine».
2. `debug:router` для проверки конкретного маршрута.
3. `debug:container` для поиска сервиса.

## Доп. теория

1. Профайлер включают только в dev/test.
2. В prod debug‑пакеты должны быть отключены.
