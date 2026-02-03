## Вопрос: Контроллеры и actions
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Контроллер принимает запрос и возвращает ответ (HTML, JSON, redirect). Каждое действие — метод `actionName`.

## Ответ

### Как работает контроллер?

Контроллер в Yii2 — это класс, наследующий `yii\web\Controller` (или `yii\console\Controller` для консоли). Он является связующим звеном между запросом пользователя и моделями/views. Каждый публичный метод, начинающийся с `action`, является действием (action) и доступен по URL. Имя контроллера определяет часть маршрута: `PostController` соответствует маршруту `post`, `SiteController` — маршруту `site`. Контроллер по умолчанию — `SiteController`, действие по умолчанию — `actionIndex`.

### Жизненный цикл action

При вызове action Yii2 выполняет строгую последовательность: создание контроллера, вызов `behaviors()` для регистрации фильтров, выполнение `beforeAction()` на уровне приложения, модуля и контроллера, выполнение самого action, затем `afterAction()` в обратном порядке. Если любой из `beforeAction()` вернет `false`, выполнение прекращается.

```php
namespace app\controllers;

use yii\web\Controller;
use yii\web\NotFoundHttpException;
use app\models\Post;

class PostController extends Controller
{
    // Действие по умолчанию (GET /post)
    public function actionIndex()
    {
        $posts = Post::find()
            ->where(['status' => Post::STATUS_ACTIVE])
            ->orderBy('created_at DESC')
            ->all();

        return $this->render('index', ['posts' => $posts]);
    }

    // GET /post/view?id=42 или /post/42 (с правилом маршрутизации)
    public function actionView($id)
    {
        $model = $this->findModel($id);
        return $this->render('view', ['model' => $model]);
    }

    // POST /post/create
    public function actionCreate()
    {
        $model = new Post();

        if ($model->load(Yii::$app->request->post()) && $model->save()) {
            Yii::$app->session->setFlash('success', 'Статья создана');
            return $this->redirect(['view', 'id' => $model->id]);
        }

        return $this->render('create', ['model' => $model]);
    }

    // Вспомогательный метод (не action — без префикса action)
    protected function findModel($id)
    {
        if (($model = Post::findOne($id)) !== null) {
            return $model;
        }
        throw new NotFoundHttpException('Страница не найдена');
    }
}
```

### Фильтры через behaviors()

Метод `behaviors()` позволяет подключить фильтры, которые выполняются до и/или после каждого action. Наиболее используемые фильтры: `AccessControl` (проверка прав доступа), `VerbFilter` (ограничение HTTP-методов), `ContentNegotiator` (формат ответа), `HttpCache` (HTTP-кэширование), `Cors` (CORS-заголовки для API).

```php
use yii\filters\AccessControl;
use yii\filters\VerbFilter;
use yii\filters\ContentNegotiator;
use yii\web\Response;

public function behaviors()
{
    return [
        // Контроль доступа
        'access' => [
            'class' => AccessControl::class,
            'only' => ['create', 'update', 'delete'],
            'rules' => [
                [
                    'allow' => true,
                    'roles' => ['@'], // только авторизованные
                ],
                [
                    'allow' => true,
                    'actions' => ['delete'],
                    'roles' => ['admin'], // только админы
                ],
            ],
        ],
        // Ограничение HTTP-методов
        'verbs' => [
            'class' => VerbFilter::class,
            'actions' => [
                'delete' => ['POST'],
                'create' => ['GET', 'POST'],
                'index'  => ['GET'],
            ],
        ],
        // Формат ответа для API
        'contentNegotiator' => [
            'class' => ContentNegotiator::class,
            'only' => ['api-data'],
            'formats' => [
                'application/json' => Response::FORMAT_JSON,
                'application/xml' => Response::FORMAT_XML,
            ],
        ],
    ];
}
```

### Standalone Actions (внешние действия)

Для переиспользования логики можно выносить actions в отдельные классы. Метод `actions()` контроллера возвращает карту `id => class`. Yii2 сам использует этот механизм: `ErrorAction`, `CaptchaAction`, `ViewAction` — все это standalone actions.

```php
// В контроллере
public function actions()
{
    return [
        // Стандартный обработчик ошибок
        'error' => [
            'class' => 'yii\web\ErrorAction',
        ],
        // Капча
        'captcha' => [
            'class' => 'yii\captcha\CaptchaAction',
            'fixedVerifyCode' => YII_ENV_TEST ? 'testme' : null,
        ],
        // Свой standalone action
        'upload' => [
            'class' => 'app\actions\UploadAction',
            'directory' => '@webroot/uploads',
            'maxSize' => 5 * 1024 * 1024,
        ],
    ];
}

// app/actions/UploadAction.php
namespace app\actions;

use yii\base\Action;
use yii\web\UploadedFile;

class UploadAction extends Action
{
    public $directory;
    public $maxSize;

    public function run()
    {
        $file = UploadedFile::getInstanceByName('file');
        if ($file && $file->size <= $this->maxSize) {
            $path = \Yii::getAlias($this->directory) . '/' . $file->name;
            $file->saveAs($path);
            return ['success' => true, 'path' => $path];
        }
        return ['success' => false];
    }
}
```

### Inline Actions с параметрами

Параметры action-метода автоматически заполняются из GET-параметров запроса. Yii2 выполняет приведение типов и выбрасывает `BadRequestHttpException`, если обязательный параметр отсутствует. Можно указывать значения по умолчанию для необязательных параметров.

```php
// GET /post/search?q=yii&page=2&sort=date
public function actionSearch($q, $page = 1, $sort = 'relevance')
{
    // $q = 'yii' (обязательный — без него будет 400 ошибка)
    // $page = 2 (из запроса)
    // $sort = 'date' (из запроса)

    $query = Post::find()->where(['like', 'title', $q]);

    $dataProvider = new \yii\data\ActiveDataProvider([
        'query' => $query,
        'pagination' => ['pageSize' => 20, 'page' => $page - 1],
        'sort' => ['defaultOrder' => [$sort => SORT_DESC]],
    ]);

    return $this->render('search', [
        'dataProvider' => $dataProvider,
        'q' => $q,
    ]);
}
```

### beforeAction и afterAction

Хуки `beforeAction()` и `afterAction()` позволяют выполнять логику до и после каждого action. Это полезно для логирования, общей подготовки данных или обработки результатов. Важно помнить, что `beforeAction()` вызывается после применения фильтров из `behaviors()`.

```php
public function beforeAction($action)
{
    if (!parent::beforeAction($action)) {
        return false;
    }

    // Логирование каждого запроса
    Yii::info("Action {$action->id} запущен пользователем " .
        Yii::$app->user->id, 'controllers');

    // Установка layout для определенных actions
    if (in_array($action->id, ['login', 'register'])) {
        $this->layout = 'auth';
    }

    return true;
}

public function afterAction($action, $result)
{
    $result = parent::afterAction($action, $result);
    // Можно модифицировать результат
    return $result;
}
```

### Практические советы

Контроллеры должны быть «тонкими» — минимум бизнес-логики, основная задача: принять данные, вызвать сервис/модель, вернуть ответ. Выносите повторяющуюся логику в standalone actions и behaviors. Всегда используйте `VerbFilter` для ограничения HTTP-методов на действиях, изменяющих данные (create, update, delete). Для AJAX-действий возвращайте JSON через `Yii::$app->response->format = Response::FORMAT_JSON`. Используйте `findModel()` паттерн для единообразной обработки несуществующих записей.

## Примеры
1. `actionView($id)` загружает модель и рендерит `view`.
2. `VerbFilter` запрещает `DELETE` без POST/DELETE.
3. `actions()` подключает `CaptchaAction` и `ErrorAction`.

## Доп. теория
1. Публичные методы без префикса `action` не являются actions.
2. Чем больше логики в контроллере, тем сложнее тестировать — лучше выносить в сервисы.
