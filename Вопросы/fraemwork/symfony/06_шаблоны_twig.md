## Вопрос: Twig шаблоны
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Twig — шаблонизатор с экранированием по умолчанию, наследованием и фильтрами.

## Ответ

### Основы Twig

Twig -- это современный шаблонизатор для PHP, созданный автором Symfony. Он компилирует шаблоны в оптимизированный PHP-код, обеспечивает **автоматическое экранирование** (защита от XSS по умолчанию) и предоставляет чистый, лаконичный синтаксис. Twig использует три типа разделителей:

- `{{ ... }}` -- вывод переменной или выражения
- `{% ... %}` -- управляющие конструкции (циклы, условия, блоки)
- `{# ... #}` -- комментарии

### Наследование шаблонов

Ключевая возможность Twig -- **наследование шаблонов** через `extends` и `block`. Базовый шаблон определяет «скелет» страницы с блоками, которые дочерние шаблоны могут переопределять:

```twig
{# templates/base.html.twig #}
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}Мой сайт{% endblock %}</title>
    {% block stylesheets %}{% endblock %}
</head>
<body>
    <nav>{% block navbar %}{% endblock %}</nav>
    <main>{% block body %}{% endblock %}</main>
    <footer>{% block footer %}&copy; 2025{% endblock %}</footer>
    {% block javascripts %}{% endblock %}
</body>
</html>
```

```twig
{# templates/users/index.html.twig #}
{% extends 'base.html.twig' %}

{% block title %}Пользователи | {{ parent() }}{% endblock %}

{% block body %}
    <h1>{{ title }}</h1>
    {% for user in users %}
        <div class="user-card">
            <p>{{ user.name }}</p>
            <small>{{ user.email }}</small>
        </div>
    {% else %}
        <p>Нет пользователей</p>
    {% endfor %}
{% endblock %}
```

### Включение и переиспользование (include, embed, macro)

```twig
{# Включение другого шаблона #}
{% include 'components/_alert.html.twig' with {type: 'success', message: 'Сохранено!'} %}

{# Embed -- как include, но можно переопределить блоки #}
{% embed 'components/_card.html.twig' %}
    {% block card_body %}Содержимое карточки{% endblock %}
{% endembed %}

{# Macro -- переиспользуемые функции в шаблонах #}
{% macro input(name, value, type) %}
    <input type="{{ type|default('text') }}" name="{{ name }}" value="{{ value }}">
{% endmacro %}

{{ _self.input('email', '', 'email') }}
```

### Фильтры и функции

Twig предоставляет богатый набор встроенных фильтров и функций:

```twig
{# Фильтры #}
{{ user.name|upper }}                    {# ИВАН #}
{{ user.name|lower }}                    {# иван #}
{{ user.name|title }}                    {# Иван #}
{{ user.bio|striptags|slice(0, 100) }}   {# Первые 100 символов без HTML #}
{{ price|number_format(2, '.', ' ') }}   {# 1 234.56 #}
{{ user.createdAt|date('d.m.Y H:i') }}   {# 15.03.2025 14:30 #}
{{ items|length }}                        {# Количество элементов #}
{{ text|nl2br }}                          {# \n → <br> #}
{{ rawHtml|raw }}                         {# Без экранирования (осторожно!) #}
{{ list|join(', ') }}                     {# Объединение массива #}

{# Функции Symfony #}
{{ path('user_show', {id: user.id}) }}   {# Генерация URL #}
{{ url('user_show', {id: user.id}) }}    {# Абсолютный URL #}
{{ asset('images/logo.png') }}            {# Путь к статике #}
{{ csrf_token('delete-user') }}           {# CSRF-токен #}
{{ is_granted('ROLE_ADMIN') }}            {# Проверка роли #}
```

### Условия и циклы

```twig
{# Условия #}
{% if users|length > 0 %}
    <p>Найдено {{ users|length }} пользователей</p>
{% elseif is_granted('ROLE_ADMIN') %}
    <p>Пользователей нет, но вы можете создать</p>
{% else %}
    <p>Нет данных</p>
{% endif %}

{# Тернарный оператор #}
{{ user.active ? 'Активен' : 'Неактивен' }}
{{ user.nickname ?? user.name ?? 'Аноним' }}

{# Циклы с loop-переменной #}
{% for user in users %}
    {{ loop.index }}. {{ user.name }}
    {% if loop.first %}(первый){% endif %}
    {% if loop.last %}(последний){% endif %}
    {# loop.index0, loop.revindex, loop.length также доступны #}
{% endfor %}
```

### Расширения Twig в Symfony

Вы можете создавать собственные фильтры и функции через Twig Extension:

```php
use Twig\Extension\AbstractExtension;
use Twig\TwigFilter;
use Twig\TwigFunction;

class AppExtension extends AbstractExtension
{
    public function getFilters(): array
    {
        return [
            new TwigFilter('price', [$this, 'formatPrice']),
        ];
    }

    public function getFunctions(): array
    {
        return [
            new TwigFunction('user_avatar', [$this, 'getUserAvatar']),
        ];
    }

    public function formatPrice(float $amount, string $currency = 'RUB'): string
    {
        return number_format($amount, 2, '.', ' ') . ' ' . $currency;
    }
}
```

```twig
{{ product.price|price }}          {# 1 234.56 RUB #}
{{ product.price|price('USD') }}   {# 1 234.56 USD #}
```

### Практические советы

- Всегда используйте `{{ ... }}` для вывода переменных -- автоматическое экранирование защитит от XSS
- Используйте `|raw` только для доверенного контента (например, из WYSIWYG-редактора, предварительно очищенного HTMLPurifier)
- Выносите повторяющиеся элементы в `include` или `macro`
- В Symfony 7 Twig интегрирован с Symfony UX для работы с Turbo и Stimulus (реактивные компоненты без полного фронтенд-фреймворка)
- Профайлер Symfony показывает время рендеринга каждого шаблона и количество вызовов

## Примеры

1. `{{ path('user_show', {id: user.id}) }}` — генерация URL.
2. `{% extends 'base.html.twig' %}` и `{% block body %}` — наследование.
3. `{{ user.bio|striptags|slice(0, 100) }}` — фильтры.

## Доп. теория

1. Экранирование включено по умолчанию, `|raw` использовать осторожно.
2. Макросы и include помогают уменьшать дублирование.
