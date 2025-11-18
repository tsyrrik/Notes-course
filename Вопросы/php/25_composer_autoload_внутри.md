# Что ставит Composer внутри
Простыми словами: `composer install` кладёт зависимости в `vendor/` и генерирует файлы автозагрузки (`vendor/autoload.php`, `autoload_runtime.php`). Подключаете `vendor/autoload.php` — классы пакетов и ваши (из секций autoload) будут подгружаться автоматически.
