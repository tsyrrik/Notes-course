## Вопрос: Модели и ActiveRecord
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
`yii\base\Model` — валидация и формы, `yii\db\ActiveRecord` — работа с БД по Active Record.

## Ответ

### Чем отличаются Model и ActiveRecord?

В Yii2 существует четкая иерархия: `yii\base\Model` — базовый класс для любых моделей, предоставляющий валидацию, массовое присваивание атрибутов, сценарии и метки. `yii\db\ActiveRecord` наследует `Model` и добавляет работу с базой данных: CRUD-операции, связи (relations), Query Builder. Модели без ActiveRecord используются для форм (например, `LoginForm`, `ContactForm`, `SearchForm`), которым не нужна привязка к таблице в БД.

### Базовая модель (yii\base\Model)

`Model` работает с атрибутами, которые определяются через метод `attributes()` (или просто как public свойства класса). Модель обеспечивает валидацию через `rules()`, метки полей через `attributeLabels()`, массовое присваивание через `load()` и `setAttributes()`.

```php
namespace app\models;

use yii\base\Model;

class ContactForm extends Model
{
    public $name;
    public $email;
    public $subject;
    public $body;

    public function rules()
    {
        return [
            [['name', 'email', 'subject', 'body'], 'required'],
            ['email', 'email'],
            ['subject', 'string', 'max' => 128],
        ];
    }

    public function attributeLabels()
    {
        return [
            'name' => 'Имя',
            'email' => 'Email',
            'subject' => 'Тема',
            'body' => 'Сообщение',
        ];
    }

    public function sendEmail()
    {
        return Yii::$app->mailer->compose()
            ->setTo(Yii::$app->params['adminEmail'])
            ->setFrom([$this->email => $this->name])
            ->setSubject($this->subject)
            ->setTextBody($this->body)
            ->send();
    }
}
```

### ActiveRecord — работа с БД

ActiveRecord представляет строку таблицы как объект. Атрибуты модели соответствуют колонкам таблицы. Yii2 автоматически определяет схему таблицы и создает атрибуты. Метод `tableName()` указывает имя таблицы (по умолчанию — имя класса в snake_case).

```php
namespace app\models;

use yii\db\ActiveRecord;

class Post extends ActiveRecord
{
    const STATUS_DRAFT = 0;
    const STATUS_ACTIVE = 1;

    public static function tableName()
    {
        return '{{%post}}'; // с поддержкой префикса таблиц
    }

    public function rules()
    {
        return [
            [['title', 'body'], 'required'],
            ['title', 'string', 'max' => 255],
            ['status', 'default', 'value' => self::STATUS_DRAFT],
            ['status', 'in', 'range' => [self::STATUS_DRAFT, self::STATUS_ACTIVE]],
            [['created_at', 'updated_at'], 'integer'],
        ];
    }

    public function attributeLabels()
    {
        return [
            'id' => 'ID',
            'title' => 'Заголовок',
            'body' => 'Содержание',
            'status' => 'Статус',
            'created_at' => 'Дата создания',
        ];
    }
}
```

### CRUD-операции

ActiveRecord предоставляет полный набор CRUD-операций. Метод `save()` автоматически определяет, нужно ли выполнить INSERT или UPDATE на основе `isNewRecord`. Перед сохранением вызывается `validate()`, и если валидация не пройдена, `save()` вернет `false`.

```php
// CREATE
$post = new Post();
$post->title = 'Новая статья';
$post->body = 'Содержание статьи';
$post->save(); // INSERT INTO post ...

// READ
$post = Post::findOne(1);                            // по ID
$post = Post::findOne(['slug' => 'my-article']);      // по условию
$posts = Post::find()->where(['status' => 1])->all(); // все активные
$count = Post::find()->where(['status' => 1])->count();

// UPDATE
$post = Post::findOne(1);
$post->title = 'Обновленный заголовок';
$post->save(); // UPDATE post SET title = ...

// Массовое обновление (без загрузки моделей)
Post::updateAll(['status' => 0], ['<', 'created_at', time() - 86400 * 30]);

// DELETE
$post = Post::findOne(1);
$post->delete(); // DELETE FROM post WHERE id = 1

// Массовое удаление
Post::deleteAll(['status' => Post::STATUS_DRAFT]);
```

### Связи (Relations)

ActiveRecord поддерживает все типы связей: `hasOne`, `hasMany`, `many-to-many` (через промежуточную таблицу с `viaTable` или `via`). Связи объявляются как методы, возвращающие `ActiveQuery`. Для доступа к связанным данным используется свойство (без `get`-префикса) или метод (с `get`-префиксом).

```php
class Post extends ActiveRecord
{
    // One-to-one: у поста один автор
    public function getAuthor()
    {
        return $this->hasOne(User::class, ['id' => 'author_id']);
    }

    // One-to-many: у поста много комментариев
    public function getComments()
    {
        return $this->hasMany(Comment::class, ['post_id' => 'id'])
            ->orderBy('created_at DESC');
    }

    // Many-to-many: теги через промежуточную таблицу
    public function getTags()
    {
        return $this->hasMany(Tag::class, ['id' => 'tag_id'])
            ->viaTable('post_tag', ['post_id' => 'id']);
    }

    // Связь с дополнительным условием
    public function getActiveComments()
    {
        return $this->hasMany(Comment::class, ['post_id' => 'id'])
            ->where(['status' => Comment::STATUS_APPROVED])
            ->orderBy('created_at DESC');
    }
}

// Использование связей
$post = Post::findOne(1);
echo $post->author->name;              // lazy loading (1 доп. запрос)
$comments = $post->comments;           // lazy loading
$tags = $post->tags;                   // через промежуточную таблицу

// Eager loading — решение проблемы N+1
$posts = Post::find()
    ->with(['author', 'comments', 'tags'])
    ->where(['status' => 1])
    ->all();
// Всего 4 запроса вместо 1 + 3*N
```

### Массовое присваивание и безопасность

Метод `load()` заполняет только «безопасные» атрибуты — те, которые упомянуты в `rules()` или явно перечислены в `scenarios()`. Атрибуты, не указанные в правилах, не будут присвоены через `load()`, что защищает от массового присваивания (mass assignment vulnerability).

```php
// В контроллере
$model = new Post();
// load() берет данные из POST в формате Post[title], Post[body]
if ($model->load(Yii::$app->request->post()) && $model->save()) {
    return $this->redirect(['view', 'id' => $model->id]);
}

// Прямое присваивание — обходит проверку safe-атрибутов
$model->author_id = Yii::$app->user->id; // безопасно, т.к. контролируемо

// Массовое присваивание без load()
$model->setAttributes(['title' => 'Test', 'body' => 'Content']);
```

### События жизненного цикла ActiveRecord

ActiveRecord предоставляет события, которые вызываются при различных операциях: `beforeValidate`, `afterValidate`, `beforeSave`, `afterSave`, `beforeDelete`, `afterDelete`, `afterFind`. Их можно использовать через переопределение методов или через `behaviors()`.

```php
class Post extends ActiveRecord
{
    public function beforeSave($insert)
    {
        if (!parent::beforeSave($insert)) {
            return false;
        }

        if ($insert) {
            $this->created_at = time();
            $this->author_id = Yii::$app->user->id;
        }
        $this->updated_at = time();

        // Генерация slug из title
        $this->slug = \yii\helpers\Inflector::slug($this->title);

        return true;
    }

    public function afterDelete()
    {
        parent::afterDelete();
        // Удаление связанных файлов
        @unlink(Yii::getAlias('@webroot/uploads/' . $this->image));
    }
}
```

### Практические советы

Используйте eager loading (`with()`) для предотвращения проблемы N+1 запросов. Для сложных выборок с агрегациями используйте Query Builder вместо ActiveRecord — это быстрее и потребляет меньше памяти. Не загружайте все записи через `all()`, если их тысячи — используйте `batch()` или `each()` для итерации большими блоками. Для форм, не привязанных к таблице, используйте `yii\base\Model`, а не ActiveRecord. Выносите бизнес-логику из `beforeSave`/`afterSave` в сервисные классы для улучшения тестируемости.

## Примеры
1. Форма без БД: `LoginForm extends Model` с `rules()` и `load()`.
2. ActiveRecord: `Post::find()->with('author')->all()` для загрузки связей.
3. Массовое обновление: `Post::updateAll(['status' => 0], ['<', 'created_at', $ts]);`.

## Доп. теория
1. `load()` заполняет только safe‑атрибуты из `rules()`/`scenarios()`.
2. ActiveRecord удобен, но для тяжёлой аналитики лучше Query Builder/SQL.
