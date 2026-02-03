## Вопрос: Безопасность и RBAC
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Хелперы для хеширования паролей, CSRF, фильтры доступа и RBAC для ролей/прав.

## Ответ

### Что дает Yii2 для безопасности?

Yii2 предоставляет комплексный набор инструментов безопасности: хеширование паролей через `bcrypt`, защита от CSRF-атак, фильтры доступа (`AccessControl`), защита от XSS через автоматическое экранирование в `Html::encode()`, защита от SQL-инъекций через параметризованные запросы в ActiveRecord и Query Builder, и полноценная система управления доступом RBAC (Role-Based Access Control). Компонент `Yii::$app->security` предоставляет криптографические утилиты для генерации случайных строк, хешей и токенов.

### Хеширование паролей

Yii2 использует `password_hash()` с алгоритмом bcrypt по умолчанию. Никогда не храните пароли в открытом виде — всегда используйте `generatePasswordHash()`.

```php
// Хеширование пароля
$hash = Yii::$app->security->generatePasswordHash('mypassword');
// Результат: $2y$13$...

// Проверка пароля
$isValid = Yii::$app->security->validatePassword('mypassword', $hash);

// Генерация случайного токена (для email-подтверждения, сброса пароля)
$token = Yii::$app->security->generateRandomString(32);

// Генерация auth key (для cookie-авторизации "Remember me")
$authKey = Yii::$app->security->generateRandomString();

// В модели User
class User extends ActiveRecord implements \yii\web\IdentityInterface
{
    public function setPassword($password)
    {
        $this->password_hash = Yii::$app->security->generatePasswordHash($password);
    }

    public function validatePassword($password)
    {
        return Yii::$app->security->validatePassword($password, $this->password_hash);
    }

    public function generateAuthKey()
    {
        $this->auth_key = Yii::$app->security->generateRandomString();
    }

    public function generatePasswordResetToken()
    {
        $this->password_reset_token = Yii::$app->security->generateRandomString() . '_' . time();
    }

    // IdentityInterface
    public static function findIdentity($id) { return static::findOne($id); }
    public static function findIdentityByAccessToken($token, $type = null) { /* ... */ }
    public function getId() { return $this->id; }
    public function getAuthKey() { return $this->auth_key; }
    public function validateAuthKey($authKey) { return $this->auth_key === $authKey; }
}
```

### Защита от CSRF

Yii2 автоматически генерирует CSRF-токен и проверяет его при каждом POST-запросе. `ActiveForm` автоматически добавляет скрытое поле с токеном. Для AJAX-запросов токен нужно передавать вручную.

```php
// CSRF-токен автоматически в ActiveForm
// Для ручных форм:
<form method="post">
    <input type="hidden" name="<?= Yii::$app->request->csrfParam ?>"
           value="<?= Yii::$app->request->csrfToken ?>">
</form>

// Для AJAX (jQuery)
$.ajax({
    url: '/post/delete',
    type: 'POST',
    data: {
        id: 42,
        _csrf: yii.getCsrfToken()
    }
});

// Отключение CSRF для определенных actions (например, для API)
public function beforeAction($action)
{
    if ($action->id === 'webhook') {
        $this->enableCsrfValidation = false;
    }
    return parent::beforeAction($action);
}
```

### AccessControl — фильтр доступа

`AccessControl` — это фильтр, проверяющий права доступа к actions контроллера. Правила проверяются сверху вниз, первое совпавшее применяется. Символ `@` означает авторизованного пользователя, `?` — гостя.

```php
use yii\filters\AccessControl;

public function behaviors()
{
    return [
        'access' => [
            'class' => AccessControl::class,
            'rules' => [
                // Разрешить всем доступ к index и view
                [
                    'actions' => ['index', 'view'],
                    'allow' => true,
                    'roles' => ['?', '@'], // все
                ],
                // Только авторизованные могут создавать
                [
                    'actions' => ['create'],
                    'allow' => true,
                    'roles' => ['@'],
                ],
                // Только админы могут удалять
                [
                    'actions' => ['delete', 'update'],
                    'allow' => true,
                    'roles' => ['admin'], // RBAC роль
                ],
                // Ограничение по IP
                [
                    'actions' => ['admin'],
                    'allow' => true,
                    'ips' => ['192.168.1.*'],
                ],
                // Ограничение по HTTP-методу
                [
                    'actions' => ['delete'],
                    'allow' => true,
                    'verbs' => ['POST'],
                    'roles' => ['admin'],
                ],
                // Правило с callback
                [
                    'actions' => ['update'],
                    'allow' => true,
                    'roles' => ['@'],
                    'matchCallback' => function ($rule, $action) {
                        $postId = Yii::$app->request->get('id');
                        $post = Post::findOne($postId);
                        return $post && $post->author_id === Yii::$app->user->id;
                    },
                ],
            ],
            // Действие при отказе в доступе
            'denyCallback' => function ($rule, $action) {
                throw new \yii\web\ForbiddenHttpException('Доступ запрещен');
            },
        ],
    ];
}
```

### RBAC — система ролей и разрешений

RBAC в Yii2 — это иерархическая система управления доступом. Она оперирует **разрешениями** (permissions), **ролями** (roles) и **правилами** (rules). Роль может содержать другие роли и разрешения. Правило — это PHP-класс, который динамически проверяет условие (например, является ли пользователь автором записи).

```php
// Настройка AuthManager в конфиге
'components' => [
    'authManager' => [
        'class' => 'yii\rbac\DbManager', // или PhpManager
        'defaultRoles' => ['guest'],
    ],
],

// Создание таблиц RBAC
// php yii migrate --migrationPath=@yii/rbac/migrations

// Инициализация ролей и разрешений (console command или миграция)
$auth = Yii::$app->authManager;

// Создание разрешений
$createPost = $auth->createPermission('createPost');
$createPost->description = 'Создание поста';
$auth->add($createPost);

$updatePost = $auth->createPermission('updatePost');
$updatePost->description = 'Редактирование поста';
$auth->add($updatePost);

$deletePost = $auth->createPermission('deletePost');
$deletePost->description = 'Удаление поста';
$auth->add($deletePost);

$manageUsers = $auth->createPermission('manageUsers');
$manageUsers->description = 'Управление пользователями';
$auth->add($manageUsers);

// Создание ролей
$author = $auth->createRole('author');
$auth->add($author);
$auth->addChild($author, $createPost);

$editor = $auth->createRole('editor');
$auth->add($editor);
$auth->addChild($editor, $author);    // наследует все от author
$auth->addChild($editor, $updatePost);

$admin = $auth->createRole('admin');
$auth->add($admin);
$auth->addChild($admin, $editor);     // наследует все от editor
$auth->addChild($admin, $deletePost);
$auth->addChild($admin, $manageUsers);

// Назначение роли пользователю
$auth->assign($author, $userId);
```

### RBAC Rules — динамические правила

Rules позволяют добавлять программную логику к разрешениям. Например, разрешить редактирование поста только его автору.

```php
namespace app\rbac;

use yii\rbac\Rule;
use app\models\Post;

class AuthorRule extends Rule
{
    public $name = 'isAuthor';

    public function execute($user, $item, $params)
    {
        if (!isset($params['post'])) {
            return false;
        }
        return $params['post']->author_id == $user;
    }
}

// Применение правила
$rule = new AuthorRule();
$auth->add($rule);

$updateOwnPost = $auth->createPermission('updateOwnPost');
$updateOwnPost->description = 'Редактирование своего поста';
$updateOwnPost->ruleName = $rule->name;
$auth->add($updateOwnPost);
$auth->addChild($updateOwnPost, $updatePost);
$auth->addChild($author, $updateOwnPost);

// Проверка в контроллере
$post = Post::findOne($id);
if (Yii::$app->user->can('updateOwnPost', ['post' => $post])) {
    // разрешено
}

// Или через AccessControl
'matchCallback' => function ($rule, $action) {
    $post = Post::findOne(Yii::$app->request->get('id'));
    return Yii::$app->user->can('updateOwnPost', ['post' => $post]);
},
```

### Практические советы

Используйте `DbManager` для RBAC в production — он хранит данные в БД и поддерживает динамическое управление ролями через админку. `PhpManager` подходит только для простых проектов с фиксированным набором ролей. Всегда кэшируйте RBAC-данные: `'cache' => 'cache'` в настройках `authManager`. Используйте `VerbFilter` совместно с `AccessControl` — запрещайте GET для delete/update. Для API используйте token-based авторизацию через `QueryParamAuth` или `HttpBearerAuth`. Никогда не полагайтесь только на клиентскую валидацию — всегда проверяйте права на сервере.

## Примеры
1. RBAC: роль `author` может `createPost`, роль `editor` наследует `author`.
2. `AccessControl` ограничивает `delete` только для `admin`.
3. `Yii::$app->security->generatePasswordHash()` для хранения паролей.

## Доп. теория
1. `DbManager` подходит для динамических ролей, `PhpManager` — для статичных.
2. RBAC — это слой авторизации; аутентификация решается отдельно.
