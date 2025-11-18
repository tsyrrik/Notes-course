# Twig шаблоны

Простыми словами: Twig — шаблонизатор с экранированием по умолчанию, наследованием и фильтрами.

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
