## Вопрос: Routing и UrlManager
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Преобразует URL в маршрут `controller/action` и обратно, поддерживает «красивые» URL.

## Ответ

### Что делает UrlManager?

`UrlManager` — это компонент приложения, отвечающий за две основные задачи: **парсинг URL** (разбор входящего запроса в маршрут `controller/action` с параметрами) и **создание URL** (генерация ссылок из маршрута и параметров). Компонент настраивается в конфигурации приложения и доступен через `Yii::$app->urlManager`. По умолчанию Yii2 использует формат `?r=controller/action`, но большинство проектов переключаются на «красивые» URL (pretty URLs) через `enablePrettyUrl`.

### Настройка Pretty URL

Для работы красивых URL необходимо выполнить два условия: включить `enablePrettyUrl` в конфигурации и настроить веб-сервер для перенаправления всех запросов на `index.php`. Параметр `showScriptName` убирает `index.php` из URL, а `enableStrictParsing` включает строгий режим, при котором разрешены только явно описанные правила.

```php
'components' => [
    'urlManager' => [
        'enablePrettyUrl' => true,
        'showScriptName' => false,
        'enableStrictParsing' => false, // true — только описанные правила
        'suffix' => '.html',            // суффикс URL (опционально)
        'rules' => [
            // Правила маршрутизации
        ],
    ],
],
```

```apache
# .htaccess для Apache
RewriteEngine on
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . index.php
```

```nginx
# Конфигурация Nginx
location / {
    try_files $uri $uri/ /index.php$is_args$args;
}
```

### Правила маршрутизации (URL Rules)

Правила описывают соответствие между шаблоном URL и маршрутом. Yii2 проверяет правила сверху вниз — первое совпавшее правило применяется. В правилах можно использовать регулярные выражения для параметров, именованные параметры, HTTP-методы и хосты.

```php
'rules' => [
    // Простое правило: /about → site/about
    'about' => 'site/about',

    // С параметром: /post/42 → post/view?id=42
    'post/<id:\d+>' => 'post/view',

    // Несколько параметров: /posts/php/2 → post/index?tag=php&page=2
    'posts/<tag:[\w-]+>/<page:\d+>' => 'post/index',
    'posts/<tag:[\w-]+>' => 'post/index',

    // Ограничение по HTTP-методу
    'PUT,PATCH post/<id:\d+>' => 'post/update',
    'DELETE post/<id:\d+>' => 'post/delete',
    'POST posts' => 'post/create',
    'GET posts' => 'post/index',

    // Правило с параметром по умолчанию
    [
        'pattern' => 'posts',
        'route' => 'post/index',
        'defaults' => ['page' => 1],
    ],

    // Правило с хостом (для поддоменов)
    [
        'pattern' => 'profile/<id:\d+>',
        'route' => 'user/profile',
        'host' => 'http://admin.<domain>',
    ],
],
```

### Создание URL в коде

Для генерации URL Yii2 предоставляет хелпер `yii\helpers\Url` и метод `Url::to()`. Маршруты, начинающиеся с `/`, являются абсолютными (от корня приложения), без `/` — относительными (от текущего модуля). Всегда используйте хелперы для генерации URL — это гарантирует корректную работу при изменении правил маршрутизации.

```php
use yii\helpers\Url;

// Генерация URL по маршруту
echo Url::to(['post/view', 'id' => 42]);       // /post/42
echo Url::to(['post/index', 'tag' => 'php']);   // /posts/php
echo Url::to(['/site/about']);                   // /about

// Абсолютный URL
echo Url::to(['post/view', 'id' => 42], true);  // https://example.com/post/42
echo Url::to(['post/view', 'id' => 42], 'https'); // принудительно https

// Текущий URL
echo Url::to(['']);         // текущий маршрут
echo Url::current();       // текущий URL с параметрами
echo Url::home();          // главная страница

// В контроллере — редирект
return $this->redirect(['post/view', 'id' => $model->id]);
```

### REST URL Rules

Для REST API Yii2 предоставляет специальный класс `yii\rest\UrlRule`, который автоматически создает набор маршрутов для стандартных CRUD-операций. Одна строка конфигурации генерирует правила для GET (index, view), POST (create), PUT/PATCH (update), DELETE (delete).

```php
'rules' => [
    // Одно правило создает набор REST-маршрутов
    ['class' => 'yii\rest\UrlRule', 'controller' => 'api/post'],

    // С дополнительными actions
    [
        'class' => 'yii\rest\UrlRule',
        'controller' => 'api/post',
        'extraPatterns' => [
            'POST publish' => 'publish',
            'GET search' => 'search',
        ],
    ],

    // Несколько контроллеров
    [
        'class' => 'yii\rest\UrlRule',
        'controller' => ['api/user', 'api/post', 'api/comment'],
        'pluralize' => true, // /users, /posts, /comments
    ],
],
```

### URL-правила с классом

Для сложной логики маршрутизации можно создать собственный класс правила, реализующий интерфейс `yii\web\UrlRuleInterface`. Это полезно, когда правила зависят от данных в БД (например, slug статьи) или требуют нестандартной логики парсинга.

```php
namespace app\components;

use yii\web\UrlRuleInterface;
use yii\base\BaseObject;

class PageUrlRule extends BaseObject implements UrlRuleInterface
{
    public function createUrl($manager, $route, $params)
    {
        if ($route === 'page/view' && isset($params['slug'])) {
            return $params['slug'];
        }
        return false;
    }

    public function parseRequest($manager, $request)
    {
        $pathInfo = $request->getPathInfo();
        // Проверяем, есть ли страница с таким slug в БД
        if (\app\models\Page::find()->where(['slug' => $pathInfo])->exists()) {
            return ['page/view', ['slug' => $pathInfo]];
        }
        return false;
    }
}

// Использование в конфиге
'rules' => [
    ['class' => 'app\components\PageUrlRule'],
],
```

### Практические советы

Размещайте более специфичные правила выше более общих — Yii2 использует первое совпавшее правило. Для отладки маршрутизации используйте Debug Toolbar — панель «Router» покажет, какое правило применилось. Кэшируйте правила маршрутизации на продакшене через `'cache' => 'cache'` в настройках UrlManager — это ускоряет парсинг, особенно при большом количестве правил. Избегайте создания URL-строк вручную — всегда используйте `Url::to()` или `Html::a()`.

## Примеры
1. Правило `'post/<id:\d+>' => 'post/view'` даёт URL `/post/42`.
2. REST‑маршруты через `yii\rest\UrlRule` для `api/post`.
3. Редирект: `return $this->redirect(['post/update', 'id' => 5]);`.

## Доп. теория
1. Порядок правил важен: первое совпадение выигрывает.
2. `enableStrictParsing=true` вернёт 404 для неописанных правил.
