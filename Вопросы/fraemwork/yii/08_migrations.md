## Вопрос: Миграции
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
PHP-классы для изменения схемы БД с версионированием.

## Ответ

### Что такое миграции в Yii2?

Миграции — это механизм версионирования схемы базы данных. Каждая миграция представляет собой PHP-класс с методами `up()` (или `safeUp()`) и `down()` (или `safeDown()`), описывающими прямое и обратное изменение БД. Миграции хранятся в директории `migrations/` (или `console/migrations/` в advanced-шаблоне) и применяются последовательно. Yii2 отслеживает примененные миграции в таблице `migration` в БД, что позволяет накатывать и откатывать изменения.

### Основные команды

Yii2 предоставляет набор консольных команд для работы с миграциями. Имя файла миграции содержит временную метку, что гарантирует уникальность и правильный порядок выполнения.

```bash
# Создание миграции
php yii migrate/create create_post_table
# Создаст файл: migrations/m231015_120000_create_post_table.php

# Применить все новые миграции
php yii migrate

# Применить N миграций
php yii migrate 3

# Откатить последнюю миграцию
php yii migrate/down

# Откатить N миграций
php yii migrate/down 3

# Повторить последнюю миграцию (down + up)
php yii migrate/redo

# Показать статус миграций
php yii migrate/history     # примененные
php yii migrate/new         # непримененные

# Пометить миграцию как примененную без выполнения
php yii migrate/mark m231015_120000_create_post_table
```

### Создание таблицы

Метод `safeUp()` выполняется в транзакции (если СУБД поддерживает DDL-транзакции). Yii2 предоставляет fluent API для определения столбцов — `$this->primaryKey()`, `$this->string()`, `$this->integer()`, `$this->text()` и т.д.

```php
use yii\db\Migration;

class m231015_120000_create_post_table extends Migration
{
    public function safeUp()
    {
        $this->createTable('{{%post}}', [
            'id' => $this->primaryKey(),
            'author_id' => $this->integer()->notNull(),
            'category_id' => $this->integer(),
            'title' => $this->string(255)->notNull(),
            'slug' => $this->string(255)->notNull()->unique(),
            'body' => $this->text()->notNull(),
            'status' => $this->smallInteger()->notNull()->defaultValue(0),
            'views_count' => $this->integer()->notNull()->defaultValue(0),
            'created_at' => $this->integer()->notNull(),
            'updated_at' => $this->integer()->notNull(),
        ]);

        // Индексы
        $this->createIndex('idx-post-author_id', '{{%post}}', 'author_id');
        $this->createIndex('idx-post-status', '{{%post}}', 'status');
        $this->createIndex('idx-post-created_at', '{{%post}}', 'created_at');

        // Внешние ключи
        $this->addForeignKey(
            'fk-post-author_id',       // имя FK
            '{{%post}}',                // таблица
            'author_id',                // столбец
            '{{%user}}',                // связанная таблица
            'id',                       // связанный столбец
            'CASCADE',                  // ON DELETE
            'CASCADE'                   // ON UPDATE
        );
    }

    public function safeDown()
    {
        $this->dropForeignKey('fk-post-author_id', '{{%post}}');
        $this->dropTable('{{%post}}');
    }
}
```

### Модификация существующей таблицы

Для изменения схемы существующей таблицы используются методы `addColumn()`, `dropColumn()`, `alterColumn()`, `renameColumn()`, `addPrimaryKey()`, `dropPrimaryKey()`.

```php
class m231020_100000_add_post_columns extends Migration
{
    public function safeUp()
    {
        // Добавление столбца
        $this->addColumn('{{%post}}', 'published_at', $this->integer()->after('status'));

        // Изменение типа столбца
        $this->alterColumn('{{%post}}', 'title', $this->string(500)->notNull());

        // Переименование столбца
        $this->renameColumn('{{%post}}', 'body', 'content');

        // Добавление составного индекса
        $this->createIndex('idx-post-status-published', '{{%post}}', ['status', 'published_at']);
    }

    public function safeDown()
    {
        $this->dropIndex('idx-post-status-published', '{{%post}}');
        $this->renameColumn('{{%post}}', 'content', 'body');
        $this->alterColumn('{{%post}}', 'title', $this->string(255)->notNull());
        $this->dropColumn('{{%post}}', 'published_at');
    }
}
```

### Миграции данных

Миграции можно использовать не только для изменения схемы, но и для модификации данных. Это полезно при рефакторинге: например, при разделении поля `name` на `first_name` и `last_name`.

```php
class m231025_150000_split_user_name extends Migration
{
    public function safeUp()
    {
        $this->addColumn('{{%user}}', 'first_name', $this->string(100));
        $this->addColumn('{{%user}}', 'last_name', $this->string(100));

        // Миграция данных
        $users = (new \yii\db\Query())
            ->select(['id', 'name'])
            ->from('{{%user}}')
            ->all();

        foreach ($users as $user) {
            $parts = explode(' ', $user['name'], 2);
            $this->update('{{%user}}', [
                'first_name' => $parts[0],
                'last_name' => $parts[1] ?? '',
            ], ['id' => $user['id']]);
        }

        $this->dropColumn('{{%user}}', 'name');
    }

    public function safeDown()
    {
        $this->addColumn('{{%user}}', 'name', $this->string(200));

        $users = (new \yii\db\Query())
            ->select(['id', 'first_name', 'last_name'])
            ->from('{{%user}}')
            ->all();

        foreach ($users as $user) {
            $this->update('{{%user}}', [
                'name' => trim($user['first_name'] . ' ' . $user['last_name']),
            ], ['id' => $user['id']]);
        }

        $this->dropColumn('{{%user}}', 'last_name');
        $this->dropColumn('{{%user}}', 'first_name');
    }
}
```

### Миграции для нескольких БД и пространства имен

Yii2 поддерживает миграции из разных директорий и пространств имен, что удобно для модульных приложений и расширений.

```bash
# Миграции из другой директории
php yii migrate --migrationPath=@app/modules/blog/migrations

# Миграции из нескольких источников (в конфиге)
```

```php
// config/console.php
'controllerMap' => [
    'migrate' => [
        'class' => 'yii\console\controllers\MigrateController',
        'migrationPath' => null, // отключаем default path
        'migrationNamespaces' => [
            'app\migrations',
            'app\modules\blog\migrations',
            'yii\queue\db\migrations', // миграции расширения yii2-queue
        ],
    ],
],
```

### Практические советы

Всегда пишите метод `safeDown()` — он нужен для отката при проблемах на деплое. Используйте `safeUp()`/`safeDown()` вместо `up()`/`down()` — они оборачивают операции в транзакцию. Используйте префикс таблиц `{{%table}}` — это позволяет разным приложениям использовать одну БД. Не используйте модели ActiveRecord в миграциях — модель может измениться в будущем, а миграция должна работать всегда; используйте `Query Builder` и `$this->insert()` / `$this->update()`. Именуйте индексы и внешние ключи по конвенции `idx-table-column` и `fk-table-column` для единообразия.

## Примеры
1. `php yii migrate/create add_status_to_post` и `addColumn()` в `safeUp()`.
2. Добавить индекс: `$this->createIndex('idx-post-status', '{{%post}}', 'status');`.
3. Миграция данных: разбить поле `name` на `first_name`/`last_name`.

## Доп. теория
1. Миграции должны быть идемпотентны и обратимы (где возможно).
2. Не используйте ActiveRecord в миграциях — модель может измениться.
