# QueryBuilder и DQL

Простыми словами: QueryBuilder строит DQL (SQL-подобный язык Doctrine) методами; DQL затем переводится в SQL. Удобно для сложных фильтров без сырого SQL.

```php
$qb = $em->createQueryBuilder()
    ->select('u')
    ->from(User::class, 'u')
    ->where('u.active = :active')
    ->setParameter('active', true)
    ->orderBy('u.createdAt', 'DESC');
$users = $qb->getQuery()->getResult();
```
