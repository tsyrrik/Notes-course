# Процессы: просмотр, управление, фон/демоны

Коротко: процесс — это выполняемая программа с PID. Важные атрибуты: `PID`, `PPID`, `USER`, `TTY`, `%CPU`, `%MEM`, `STAT` (состояние), `COMMAND`.

## Как смотреть процессы
- Все процессы (BSD-стиль): `ps aux | less`
- Дерево процессов: `ps -ef | head`, `pstree -p` (пакет `pstree`)
- Поиск по имени: `pgrep -fl php`, `ps aux | grep nginx`
- «Живой» мониторинг: `top` или `htop` (удобнее, пакет `htop`)
- Кто слушает порт: `ss -ltnp | grep :80` или `sudo lsof -i :5432`
- Служба (systemd): `systemctl status nginx`, логи: `journalctl -u nginx -f`

## Как завершать процессы
- Мягко (по умолчанию SIGTERM): `kill <PID>`
- Жёстко (SIGKILL, осторожно): `kill -9 <PID>`

## Коды выхода (exit status)
- Что это: число 0–255, с которым процесс завершает работу; оболочка сохраняет его в переменной `$?`.
- `0` — успех (нормальное завершение).
- `1` — общая/неуточнённая ошибка.
- `2` — неправильное использование команды/аргументов (часто для встроенных команд оболочки).
- `126` — команда найдена, но не может быть исполнена (нет прав или файл неисполняемый).
- `127` — команда не найдена (ошибка PATH/опечатка).
- `128+N` — процесс завершён сигналом `N`:
  - `130 = 128+2` — SIGINT (нажат Ctrl+C),
  - `137 = 128+9` — SIGKILL (часто «Killed»/OOM или `kill -9`),
  - `139 = 128+11` — SIGSEGV (segmentation fault).
- `255` — код вне диапазона или `exit -1` (интерпретируется как 255).
- Проверить код последней команды: `echo $?`. В скриптах явно завершайтесь `exit N`.
## Фоновые задания и «джобы»
- Запуск в фоне: `cmd &` → посмотреть: `jobs` → вернуть на передний план: `fg %1` → снова в фон: `bg %1`
- Приостановить текущую задачу: `Ctrl+Z`, затем `bg` для продолжения в фоне

## Отделение от терминала (долгие задачи)
- Надёжный шаблон: `nohup cmd >>/var/log/cmd.log 2>&1 & disown`
- Альтернатива: `setsid cmd >/dev/null 2>&1 &`
- Для постоянной работы удобнее использовать `tmux`/`screen` или оформить как службу (`systemd`)

## Демоны и systemd
- Запуск/остановка: `systemctl start|stop|restart <service>`
- Статус/логи: `systemctl status <service>`, `journalctl -u <service> -n 200 -f`
- Автозапуск: `systemctl enable|disable <service>`

Мини‑юнит (пример):
```
# /etc/systemd/system/app-worker.service
[Unit]
Description=App worker
After=network.target

[Service]
ExecStart=/usr/bin/php /var/www/app/bin/console app:worker
Restart=always
User=www-data
WorkingDirectory=/var/www/app
StandardOutput=append:/var/log/app/worker.log
StandardError=append:/var/log/app/worker.err

[Install]
WantedBy=multi-user.target
```
Затем: `systemctl daemon-reload && systemctl enable --now app-worker`

## Практика для PHP backend
- PHP-FPM: `systemctl status php*-fpm`, логи ошибок PHP — в настройках FPM/INI
- Nginx master/worker: `ps aux | grep nginx`, логи: `/var/log/nginx/*`
- Воркеры/крон: оформляйте через `systemd` или `supervisor` (`supervisorctl status|restart`)

## См. также
- [[Частые команды Linux]]
- [[Переключение пользователей и root]]
- [[Потоки (STDIN, STDOUT, STDERR)]]
- [[Пайпы и конвейеры (pipeline)]]
