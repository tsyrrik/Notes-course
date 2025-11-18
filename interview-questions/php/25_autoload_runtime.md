# Что ставит Composer внутри
`composer install` создаёт `vendor/` и файлы `autoload.php`, `autoload_runtime.php`, плюс зависимости. После `require_once 'vendor/autoload.php';` автозагрузка работает для зависимостей и ваших классов (если прописаны в autoload секции).
