## Вопрос: Фильтры и behaviors
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Механизм, который выполняется до или после action (например, проверка доступа или HTTP-методов).

## Ответ

Фильтры выполняются до/после action и решают задачи доступа, методов, форматов ответа.

Какие фильтры используются чаще всего:
- `AccessControl` — доступ по ролям/правам.
- `VerbFilter` — ограничение методов (GET/POST).
- `ContentNegotiator` — форматы ответа.

Как подключить фильтр:
```php
public function behaviors() {
    return [
        'access' => [
            'class' => yii\\filters\\AccessControl::class,
            'rules' => [
                ['allow' => true, 'roles' => ['@']],
            ],
        ],
    ];
}
```

## Примеры

1. `VerbFilter` запрещает `DELETE` по GET.
2. `AccessControl` пускает только роли `admin`.
3. `ContentNegotiator` отвечает JSON для API.

## Доп. теория

1. Фильтры подключаются через `behaviors()` и применяются к выбранным actions.
2. Порядок фильтров важен, можно ограничивать через `only/except`.
