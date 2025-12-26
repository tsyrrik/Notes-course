## Вопрос: Миграции
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
PHP-классы для изменения схемы БД с версионированием.

## Ответ
## Вопрос: Что такое миграции в Yii2?
Ответ: PHP-классы для изменения схемы БД с версионированием.

## Вопрос: Основные команды
Ответ:
- `php yii migrate/create create_post_table`
- `php yii migrate`

## Вопрос: Пример миграции
Ответ:
```php
public function safeUp() {
    $this->createTable('post', [
        'id' => $this->primaryKey(),
        'title' => $this->string()->notNull(),
    ]);
}
```
