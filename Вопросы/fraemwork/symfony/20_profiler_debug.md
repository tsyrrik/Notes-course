## Вопрос: Profiler и debug
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
в dev-окружении работает Web Profiler Toolbar и страница профайлера; можно смотреть запросы/SQL/логи/почту. Панель подключает `framework.profiler` и `debug` пакеты.

## Ответ
```bash
php bin/console debug:router
php bin/console debug:container
php bin/console debug:config framework
```
