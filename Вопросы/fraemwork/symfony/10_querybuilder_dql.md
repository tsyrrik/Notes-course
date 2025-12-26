## Вопрос: QueryBuilder и DQL
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
QueryBuilder строит DQL (SQL-подобный язык Doctrine) методами; DQL затем переводится в SQL. Удобно для сложных фильтров без сырого SQL.

## Ответ
```php
$qb = $em->createQueryBuilder()
    ->select('u')
    ->from(User::class, 'u')
    ->where('u.active = :active')
    ->setParameter('active', true)
    ->orderBy('u.createdAt', 'DESC');
$users = $qb->getQuery()->getResult();
```
