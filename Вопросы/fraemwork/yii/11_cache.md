## Вопрос: Кэширование
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Кэш — отдельный компонент, можно кэшировать данные, страницы и фрагменты.

## Ответ

### Как устроено кэширование?

Кэширование в Yii2 — это многоуровневая система, построенная на компонентном подходе. Основа — компонент `cache`, доступный через `Yii::$app->cache`. Yii2 поддерживает три типа кэширования: **кэширование данных** (произвольные значения), **фрагментное кэширование** (части HTML-страницы), **страничное кэширование** (целые страницы) и **HTTP-кэширование** (заголовки Last-Modified, ETag). Кэш работает через единый интерфейс `yii\caching\CacheInterface`, что позволяет легко переключать бэкенд.

### Драйверы кэша

Yii2 поддерживает множество бэкендов для хранения кэша. Выбор зависит от требований проекта: для разработки подойдет `FileCache`, для продакшена — `Redis` или `Memcached`.

```php
// config/web.php
'components' => [
    // FileCache — для разработки и небольших проектов
    'cache' => [
        'class' => 'yii\caching\FileCache',
        'cachePath' => '@runtime/cache',
        'defaultDuration' => 3600,
    ],

    // Redis — для продакшена, поддерживает кластеризацию
    'cache' => [
        'class' => 'yii\redis\Cache',
        'redis' => [
            'hostname' => 'localhost',
            'port' => 6379,
            'database' => 0,
        ],
        'defaultDuration' => 3600,
    ],

    // Memcached
    'cache' => [
        'class' => 'yii\caching\MemCache',
        'useMemcached' => true,
        'servers' => [
            ['host' => '127.0.0.1', 'port' => 11211, 'weight' => 100],
        ],
    ],

    // DbCache — хранение в таблице БД
    'cache' => [
        'class' => 'yii\caching\DbCache',
        'cacheTable' => '{{%cache}}',
    ],

    // DummyCache — отключение кэша (для тестов)
    'cache' => [
        'class' => 'yii\caching\DummyCache',
    ],
],
```

### Кэширование данных

Основные методы: `get()`, `set()`, `delete()`, `exists()`, `flush()`. Метод `getOrSet()` — самый удобный: он проверяет кэш и вычисляет значение только при промахе.

```php
// getOrSet — самый частый паттерн
$posts = Yii::$app->cache->getOrSet('popular-posts', function () {
    return Post::find()
        ->where(['status' => Post::STATUS_ACTIVE])
        ->orderBy('views_count DESC')
        ->limit(10)
        ->asArray()
        ->all();
}, 3600); // TTL в секундах

// Ручное управление
Yii::$app->cache->set('user-stats-42', $stats, 600); // 10 минут
$stats = Yii::$app->cache->get('user-stats-42');       // false если нет

// Удаление
Yii::$app->cache->delete('user-stats-42');

// Очистка всего кэша
Yii::$app->cache->flush();

// Кэш с зависимостями (dependency)
use yii\caching\DbDependency;
use yii\caching\TagDependency;

// Инвалидируется при изменении данных в таблице
$dep = new DbDependency([
    'sql' => 'SELECT MAX(updated_at) FROM post',
]);
$posts = Yii::$app->cache->getOrSet('all-posts', function () {
    return Post::find()->all();
}, 3600, $dep);

// Тегированный кэш — инвалидация по тегу
$dep = new TagDependency(['tags' => ['posts', 'post-42']]);
Yii::$app->cache->set('post-42', $postData, 0, $dep);

// Инвалидация по тегу — сбросит все записи с этим тегом
TagDependency::invalidate(Yii::$app->cache, 'posts');
TagDependency::invalidate(Yii::$app->cache, 'post-42');
```

### Фрагментное кэширование

Кэширование части HTML-страницы прямо в view. Удобно для тяжелых блоков (сайдбар, топ-меню, список категорий), которые не меняются при каждом запросе.

```php
// views/site/index.php

// Простое фрагментное кэширование
<?php if ($this->beginCache('sidebar', ['duration' => 3600])) : ?>
    <div class="sidebar">
        <?= $this->render('_categories', ['categories' => $categories]) ?>
        <?= $this->render('_popular_posts') ?>
        <?= $this->render('_tags_cloud') ?>
    </div>
<?php $this->endCache(); endif; ?>

// С зависимостями
<?php if ($this->beginCache('top-menu', [
    'duration' => 0, // бессрочно
    'dependency' => new \yii\caching\DbDependency([
        'sql' => 'SELECT MAX(updated_at) FROM menu_item',
    ]),
])) : ?>
    <?= $this->render('_menu') ?>
<?php $this->endCache(); endif; ?>

// С вариациями (разный кэш для разных параметров)
<?php if ($this->beginCache('posts-list', [
    'duration' => 600,
    'variations' => [
        Yii::$app->language,          // по языку
        Yii::$app->request->get('page', 1), // по странице
        Yii::$app->user->isGuest ? 'guest' : 'auth',
    ],
])) : ?>
    <?= \yii\widgets\ListView::widget([
        'dataProvider' => $dataProvider,
        'itemView' => '_post',
    ]) ?>
<?php $this->endCache(); endif; ?>
```

### Страничное кэширование

Кэширование целых страниц через `PageCache` filter в контроллере. Подходит для статических или редко обновляемых страниц.

```php
use yii\filters\PageCache;
use yii\caching\DbDependency;

public function behaviors()
{
    return [
        'pageCache' => [
            'class' => PageCache::class,
            'only' => ['index', 'about'],
            'duration' => 3600,
            'variations' => [
                Yii::$app->language,
                Yii::$app->request->get('page'),
            ],
            'dependency' => new DbDependency([
                'sql' => 'SELECT MAX(updated_at) FROM post',
            ]),
        ],
    ];
}
```

### HTTP-кэширование

Фильтр `HttpCache` управляет заголовками `Last-Modified` и `ETag`, позволяя браузеру и прокси-серверам кэшировать ответы.

```php
use yii\filters\HttpCache;

public function behaviors()
{
    return [
        'httpCache' => [
            'class' => HttpCache::class,
            'only' => ['view'],
            'lastModified' => function ($action, $params) {
                $id = Yii::$app->request->get('id');
                return Post::find()
                    ->where(['id' => $id])
                    ->max('updated_at');
            },
            'etagSeed' => function ($action, $params) {
                $post = Post::findOne(Yii::$app->request->get('id'));
                return serialize([$post->id, $post->updated_at]);
            },
        ],
    ];
}
```

### Практические советы

Используйте `TagDependency` для инвалидации связанных кэшей — это гибче, чем `DbDependency`, и не требует лишних запросов к БД. Включайте кэширование схемы БД на проде: `'enableSchemaCache' => true, 'schemaCacheDuration' => 86400`. Для высоконагруженных проектов используйте Redis — он быстрее FileCache и поддерживает атомарные операции. Не кэшируйте персонализированные данные без `variations` — иначе один пользователь увидит данные другого. Начинайте с кэша данных, затем добавляйте фрагментный — это дает наибольший эффект при минимальных усилиях.

## Примеры
1. `getOrSet('popular-posts', fn() => Post::find()->all(), 3600)`.
2. Фрагментное кэширование боковой колонки через `beginCache()`.
3. HTTP‑кэш: `HttpCache` с `Last-Modified` по `updated_at`.

## Доп. теория
1. Вариации кэша обязательны для персонализированных страниц.
2. `TagDependency` удобен для массовой инвалидации связанных ключей.
