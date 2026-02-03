## Вопрос: Events и Behaviors
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Механизм подписки на события жизненного цикла (например, перед вставкой в БД).

## Ответ

### Что такое events в Yii2?

Events (события) — это механизм, позволяющий подписываться на определенные действия объекта и выполнять произвольный код при их наступлении. Любой класс, наследующий `yii\base\Component`, поддерживает события. ActiveRecord, Controller, Application — все они предоставляют набор событий жизненного цикла. Событие состоит из имени (строковая константа) и обработчиков (callable), которые вызываются при его срабатывании. Это реализация паттерна Observer, который позволяет расширять поведение объекта без модификации его кода.

### Подписка на события

Подписаться на событие можно тремя способами: через метод `on()`, через конфигурацию (ключ `on eventName`) и через глобальные события на уровне класса.

```php
// 1. Через метод on()
$post = new Post();
$post->on(Post::EVENT_BEFORE_INSERT, function ($event) {
    $event->sender->created_at = time();
    $event->sender->author_id = Yii::$app->user->id;
});

// 2. Через конфигурацию компонента
'components' => [
    'db' => [
        'class' => 'yii\db\Connection',
        'dsn' => '...',
        'on afterOpen' => function ($event) {
            $event->sender->createCommand("SET time_zone = '+03:00'")->execute();
        },
    ],
],

// 3. Глобальные события на уровне класса — срабатывают для ВСЕХ экземпляров
use yii\base\Event;

Event::on(Post::class, Post::EVENT_AFTER_INSERT, function ($event) {
    // Отправить уведомление при создании любого поста
    Yii::$app->mailer->compose()
        ->setTo(Yii::$app->params['adminEmail'])
        ->setSubject('Новый пост: ' . $event->sender->title)
        ->send();
});

// Глобальные события для всех ActiveRecord
Event::on(ActiveRecord::class, ActiveRecord::EVENT_AFTER_INSERT, function ($event) {
    Yii::info('Создана запись в ' . get_class($event->sender), 'db');
});
```

### Создание собственных событий

Вы можете определять собственные события в любом компоненте. Событие — это просто строковая константа и вызов `$this->trigger()`. Для передачи данных в обработчик используется объект события (наследник `yii\base\Event`).

```php
namespace app\models;

use yii\base\Event;
use yii\db\ActiveRecord;

// Пользовательский класс события
class OrderEvent extends Event
{
    public $order;
    public $previousStatus;
}

class Order extends ActiveRecord
{
    const EVENT_STATUS_CHANGED = 'statusChanged';
    const EVENT_PAYMENT_RECEIVED = 'paymentReceived';

    private $_oldStatus;

    public function afterFind()
    {
        parent::afterFind();
        $this->_oldStatus = $this->status;
    }

    public function afterSave($insert, $changedAttributes)
    {
        parent::afterSave($insert, $changedAttributes);

        if (isset($changedAttributes['status'])) {
            $event = new OrderEvent();
            $event->order = $this;
            $event->previousStatus = $this->_oldStatus;
            $this->trigger(self::EVENT_STATUS_CHANGED, $event);
        }
    }

    public function markAsPaid()
    {
        $this->status = self::STATUS_PAID;
        $this->paid_at = time();
        $this->save();
        $this->trigger(self::EVENT_PAYMENT_RECEIVED);
    }
}

// Подписка на пользовательское событие
$order->on(Order::EVENT_STATUS_CHANGED, function (OrderEvent $event) {
    Yii::info("Заказ #{$event->order->id}: статус изменен с {$event->previousStatus} на {$event->order->status}");
});
```

### Что такое behaviors?

Behaviors (поведения) — это механизм, позволяющий добавлять функциональность к компоненту без наследования. Behavior может подписываться на события владельца, добавлять свойства и методы. Это реализация паттерна Mixin. Behaviors подключаются через метод `behaviors()` или динамически через `attachBehavior()`.

```php
use yii\behaviors\TimestampBehavior;
use yii\behaviors\BlameableBehavior;
use yii\behaviors\SluggableBehavior;
use yii\behaviors\AttributesBehavior;

class Post extends ActiveRecord
{
    public function behaviors()
    {
        return [
            // Автоматическое заполнение created_at и updated_at
            'timestamp' => [
                'class' => TimestampBehavior::class,
                'createdAtAttribute' => 'created_at',
                'updatedAtAttribute' => 'updated_at',
                'value' => time(), // или new \yii\db\Expression('NOW()')
            ],

            // Автоматическое заполнение author_id
            'blameable' => [
                'class' => BlameableBehavior::class,
                'createdByAttribute' => 'author_id',
                'updatedByAttribute' => 'updated_by',
            ],

            // Генерация slug из title
            'sluggable' => [
                'class' => SluggableBehavior::class,
                'attribute' => 'title',
                'slugAttribute' => 'slug',
                'ensureUnique' => true,
                'immutable' => true, // не менять slug при обновлении
            ],
        ];
    }
}

// Теперь при сохранении автоматически заполняются
// created_at, updated_at, author_id, slug
$post = new Post(['title' => 'Мой пост']);
$post->body = 'Текст';
$post->save();
// $post->created_at = 1697400000
// $post->author_id = 1
// $post->slug = 'moj-post'
```

### Создание собственного behavior

Для создания пользовательского behavior нужно наследовать `yii\base\Behavior` и переопределить метод `events()` для подписки на события владельца.

```php
namespace app\behaviors;

use yii\base\Behavior;
use yii\db\ActiveRecord;

class SoftDeleteBehavior extends Behavior
{
    public $deletedAtAttribute = 'deleted_at';
    public $deletedByAttribute = 'deleted_by';

    public function events()
    {
        return [
            ActiveRecord::EVENT_BEFORE_DELETE => 'softDelete',
        ];
    }

    public function softDelete($event)
    {
        $owner = $this->owner;

        // Вместо реального удаления — помечаем запись
        $owner->updateAttributes([
            $this->deletedAtAttribute => time(),
            $this->deletedByAttribute => \Yii::$app->user->id,
        ]);

        // Отменяем реальное удаление
        $event->isValid = false;
    }

    // Метод, доступный через владельца: $model->restore()
    public function restore()
    {
        return $this->owner->updateAttributes([
            $this->deletedAtAttribute => null,
            $this->deletedByAttribute => null,
        ]);
    }

    // Scope для запросов
    public function getIsDeleted()
    {
        return $this->owner->{$this->deletedAtAttribute} !== null;
    }
}

// Использование
class Post extends ActiveRecord
{
    public function behaviors()
    {
        return [
            'softDelete' => [
                'class' => 'app\behaviors\SoftDeleteBehavior',
            ],
        ];
    }
}

$post->delete();      // пометит как удаленный, но не удалит из БД
$post->restore();     // восстановит запись
$post->isDeleted;     // true/false
```

### Практические советы

Используйте глобальные события (`Event::on`) для cross-cutting concerns: аудит, логирование, отправка уведомлений. Behaviors идеальны для переиспользуемой логики: timestamp, slug, soft delete, versioning. Не злоупотребляйте событиями — слишком длинные цепочки обработчиков усложняют отладку. Для отмены операции из обработчика события установите `$event->isValid = false` (работает для событий `beforeXxx`). Помните о порядке обработчиков — они вызываются в порядке подключения, behaviors обрабатываются первыми.

## Примеры
1. `TimestampBehavior` автоматически заполняет `created_at`/`updated_at`.
2. `Event::on(ActiveRecord::class, EVENT_AFTER_INSERT, ...)` для аудита.
3. `SoftDeleteBehavior` отменяет реальное удаление через `$event->isValid = false`.

## Доп. теория
1. События работают на уровне экземпляра и класса, порядок обработчиков важен.
2. Behavior может добавлять свойства/методы и слушать события владельца.
