## Вопрос: Валидация и сценарии
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Правила задаются в `rules()` и применяются через `validate()`.

## Ответ

### Как описываются правила валидации?

Валидация в Yii2 — это механизм проверки данных модели, описанный в методе `rules()`. Каждое правило — массив вида `[атрибуты, валидатор, ...параметры]`. Yii2 предоставляет более 20 встроенных валидаторов: `required`, `string`, `number`, `email`, `url`, `date`, `boolean`, `in`, `match` (regex), `unique`, `exist`, `compare`, `file`, `image`, `ip` и другие. При вызове `validate()` Yii2 применяет все правила, соответствующие текущему сценарию, и заполняет массив ошибок `$model->errors`.

```php
class User extends \yii\db\ActiveRecord
{
    public function rules()
    {
        return [
            // Обязательные поля
            [['username', 'email', 'password_hash'], 'required'],

            // Строковая валидация с ограничением длины
            ['username', 'string', 'min' => 3, 'max' => 32],
            ['password_hash', 'string', 'min' => 6],

            // Email
            ['email', 'email'],

            // Уникальность в таблице
            ['email', 'unique', 'message' => 'Этот email уже зарегистрирован'],
            ['username', 'unique'],

            // Значение по умолчанию
            ['status', 'default', 'value' => self::STATUS_INACTIVE],

            // Допустимые значения
            ['status', 'in', 'range' => [self::STATUS_INACTIVE, self::STATUS_ACTIVE, self::STATUS_BANNED]],

            // Регулярное выражение
            ['username', 'match', 'pattern' => '/^[a-zA-Z0-9_]+$/'],

            // Числовая валидация
            ['age', 'integer', 'min' => 18, 'max' => 120],

            // Фильтр (преобразование значения)
            ['email', 'filter', 'filter' => 'trim'],
            ['email', 'filter', 'filter' => 'strtolower'],

            // Безопасные атрибуты (доступны для массового присваивания)
            [['bio', 'avatar_url'], 'safe'],
        ];
    }
}
```

### Сценарии (Scenarios)

Сценарии позволяют применять разные наборы правил для разных контекстов. Например, при регистрации нужен пароль, а при обновлении профиля — нет. Сценарий задается через `$model->scenario`. В `rules()` сценарий указывается через параметр `on`, а в `scenarios()` — перечень «активных» атрибутов для каждого сценария.

```php
class User extends \yii\db\ActiveRecord
{
    const SCENARIO_REGISTER = 'register';
    const SCENARIO_UPDATE_PROFILE = 'update_profile';
    const SCENARIO_ADMIN_UPDATE = 'admin_update';

    public function rules()
    {
        return [
            // Применяется во всех сценариях
            [['username', 'email'], 'required'],
            ['email', 'email'],
            ['username', 'string', 'max' => 32],

            // Только при регистрации
            ['password', 'required', 'on' => self::SCENARIO_REGISTER],
            ['password', 'string', 'min' => 6, 'on' => self::SCENARIO_REGISTER],
            ['password_repeat', 'compare', 'compareAttribute' => 'password',
                'on' => self::SCENARIO_REGISTER],

            // Только для админа
            ['role', 'in', 'range' => ['user', 'moderator', 'admin'],
                'on' => self::SCENARIO_ADMIN_UPDATE],
            ['status', 'integer', 'on' => self::SCENARIO_ADMIN_UPDATE],
        ];
    }

    public function scenarios()
    {
        $scenarios = parent::scenarios();
        // Какие атрибуты доступны для массового присваивания в каждом сценарии
        $scenarios[self::SCENARIO_REGISTER] = ['username', 'email', 'password', 'password_repeat'];
        $scenarios[self::SCENARIO_UPDATE_PROFILE] = ['username', 'email', 'bio', 'avatar_url'];
        $scenarios[self::SCENARIO_ADMIN_UPDATE] = ['username', 'email', 'role', 'status'];
        return $scenarios;
    }
}

// Использование сценариев
// Регистрация
$user = new User(['scenario' => User::SCENARIO_REGISTER]);
$user->load(Yii::$app->request->post());
$user->validate(); // проверит username, email, password, password_repeat

// Обновление профиля
$user = User::findOne($id);
$user->scenario = User::SCENARIO_UPDATE_PROFILE;
$user->load(Yii::$app->request->post()); // загрузит только username, email, bio, avatar_url
$user->save();
```

### Пользовательские валидаторы

Yii2 позволяет создавать собственные валидаторы двумя способами: inline-валидатор (метод в модели) и standalone-валидатор (отдельный класс). Inline подходит для простых проверок, standalone — для переиспользования.

```php
class Order extends \yii\db\ActiveRecord
{
    public function rules()
    {
        return [
            ['total', 'required'],
            // Inline валидатор — метод этого же класса
            ['delivery_date', 'validateDeliveryDate'],
            // Standalone валидатор
            ['phone', 'app\validators\PhoneValidator'],
        ];
    }

    // Inline валидатор
    public function validateDeliveryDate($attribute, $params)
    {
        $date = strtotime($this->$attribute);
        if ($date && $date < strtotime('+1 day')) {
            $this->addError($attribute, 'Дата доставки должна быть минимум завтра');
        }
    }
}

// Standalone валидатор — app/validators/PhoneValidator.php
namespace app\validators;

use yii\validators\Validator;

class PhoneValidator extends Validator
{
    public $pattern = '/^\+7\d{10}$/';

    public function validateAttribute($model, $attribute)
    {
        if (!preg_match($this->pattern, $model->$attribute)) {
            $this->addError($model, $attribute,
                'Номер телефона должен быть в формате +7XXXXXXXXXX');
        }
    }

    // Для клиентской валидации (JavaScript)
    public function clientValidateAttribute($model, $attribute, $view)
    {
        $pattern = json_encode($this->pattern);
        return <<<JS
        if (!new RegExp($pattern).test(value)) {
            messages.push('Номер телефона должен быть в формате +7XXXXXXXXXX');
        }
        JS;
    }
}
```

### Условная валидация

Правила могут применяться по условию через параметры `when` (серверная проверка) и `whenClient` (клиентская). Это полезно, когда валидация одного поля зависит от значения другого.

```php
public function rules()
{
    return [
        ['delivery_type', 'required'],
        // Адрес обязателен только при доставке курьером
        ['address', 'required',
            'when' => function ($model) {
                return $model->delivery_type === 'courier';
            },
            'whenClient' => "function (attribute, value) {
                return $('#order-delivery_type').val() === 'courier';
            }",
        ],
        // Скидка не может превышать 50% для обычных пользователей
        ['discount', 'number', 'max' => 50,
            'when' => function ($model) {
                return !Yii::$app->user->can('admin');
            },
        ],
    ];
}
```

### Стандартный цикл формы

Типичный паттерн работы с формой в Yii2: создание/загрузка модели, `load()` для массового присваивания из POST-данных, `validate()` для проверки, `save()` для сохранения (в случае ActiveRecord).

```php
// Контроллер
public function actionUpdate($id)
{
    $model = Post::findOne($id);
    if ($model === null) {
        throw new NotFoundHttpException();
    }

    if ($model->load(Yii::$app->request->post())) {
        if ($model->validate()) {
            $model->save(false); // false — пропустить повторную валидацию
            Yii::$app->session->setFlash('success', 'Данные сохранены');
            return $this->redirect(['view', 'id' => $model->id]);
        }
        // Ошибки валидации доступны через $model->errors
    }

    return $this->render('update', ['model' => $model]);
}

// Получение ошибок
$model->validate();
$errors = $model->getErrors();          // все ошибки
$error = $model->getFirstError('title'); // первая ошибка поля title
$hasError = $model->hasErrors('title');  // есть ли ошибки для поля
```

### Практические советы

Используйте `filter` валидатор для trim и нормализации данных перед основной валидацией — ставьте его в начало списка правил. Для сложной бизнес-логики создавайте standalone валидаторы — они переиспользуемы и тестируемы. Не забывайте про клиентскую валидацию (`clientValidateAttribute`) — она улучшает UX, давая мгновенную обратную связь. Помните, что `save()` по умолчанию вызывает `validate()` — передавайте `false`, если валидация уже выполнена. Используйте `scenarios()` для контроля массового присваивания, а не только для условной валидации.

## Примеры
1. Сценарий регистрации требует `password`, сценарий профиля — нет.
2. Inline‑валидатор запрещает доставку «сегодня» для поля `delivery_date`.
3. Условная валидация адреса при `delivery_type = courier`.

## Доп. теория
1. `scenarios()` определяет safe‑атрибуты для массового присваивания.
2. `save()` вызывает `validate()`, поэтому `save(false)` используйте только после ручной проверки.
