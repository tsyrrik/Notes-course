# Profiler и debug

Простыми словами: в dev-окружении работает Web Profiler Toolbar и страница профайлера; можно смотреть запросы/SQL/логи/почту. Панель подключает `framework.profiler` и `debug` пакеты.

```bash
php bin/console debug:router
php bin/console debug:container
php bin/console debug:config framework
```
