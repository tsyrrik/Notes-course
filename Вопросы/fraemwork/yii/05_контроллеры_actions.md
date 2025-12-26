## Вопрос: Контроллеры и actions
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Контроллер принимает запрос и возвращает ответ (HTML, JSON, redirect). Каждое действие — метод `actionName`.

## Ответ
## Вопрос: Как работает контроллер?
Ответ: Контроллер принимает запрос и возвращает ответ (HTML, JSON, redirect). Каждое действие — метод `actionName`.

## Вопрос: Пример контроллера
Ответ:
```php
class PostController extends yii\\web\\Controller {
    public function actionView($id) {
        $model = Post::findOne($id);
        return $this->render('view', ['model' => $model]);
    }
}
```

## Вопрос: Какие есть хуки и фильтры?
Ответ:
- `behaviors()` для фильтров (access, verb).
- `beforeAction/afterAction` — хуки жизненного цикла.
- External actions через `actions()`.
