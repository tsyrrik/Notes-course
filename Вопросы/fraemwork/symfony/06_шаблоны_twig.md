## Вопрос: Twig шаблоны
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
Twig — шаблонизатор с экранированием по умолчанию, наследованием и фильтрами.

## Ответ
```twig
{% extends 'base.html.twig' %}
{% block body %}
  <h1>{{ title }}</h1>
  {% for user in users %}
    <p>{{ user.name }}</p>
  {% else %}
    <p>Нет пользователей</p>
  {% endfor %}
{% endblock %}
```
