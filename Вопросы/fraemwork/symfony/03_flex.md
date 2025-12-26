## Вопрос: Symfony Flex
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
Flex управляет зависимостями и "рецептами" — автогенерирует конфиги, папки, пакеты при установке через Composer.

## Ответ
```bash
composer create-project symfony/skeleton myapp
composer require orm
composer require symfony/twig-pack
```
