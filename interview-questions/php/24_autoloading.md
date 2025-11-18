# Автозагрузка классов
Обычно через Composer (PSR-4 правила в `composer.json`). После `composer install` — подключите `vendor/autoload.php`, классы загружаются автоматически по `use`.
Без Composer: регистрируйте функцию в `spl_autoload_register()` и подключайте файлы по имени класса/пути.
