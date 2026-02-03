## Вопрос: QueryBuilder и DQL
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
QueryBuilder строит DQL (SQL-подобный язык Doctrine) методами; DQL затем переводится в SQL. Удобно для сложных фильтров без сырого SQL.

## Ответ

### Что такое DQL и QueryBuilder

**DQL** (Doctrine Query Language) -- это объектно-ориентированный язык запросов Doctrine, похожий на SQL, но оперирующий не таблицами и колонками, а сущностями и их свойствами. DQL компилируется в SQL, адаптированный под конкретную СУБД. **QueryBuilder** -- это fluent API для программного построения DQL-запросов, что удобнее при динамических фильтрах и условиях.

### Базовый QueryBuilder

```php
// В репозитории (наследник ServiceEntityRepository)
public function findActiveUsers(): array
{
    return $this->createQueryBuilder('u')
        ->andWhere('u.active = :active')
        ->setParameter('active', true)
        ->orderBy('u.createdAt', 'DESC')
        ->setMaxResults(50)
        ->getQuery()
        ->getResult();
}

// Через EntityManager
$qb = $em->createQueryBuilder()
    ->select('u')
    ->from(User::class, 'u')
    ->where('u.active = :active')
    ->setParameter('active', true)
    ->orderBy('u.createdAt', 'DESC');

$users = $qb->getQuery()->getResult();
```

### DQL напрямую

```php
// Простой DQL-запрос
$query = $em->createQuery(
    'SELECT u FROM App\Entity\User u WHERE u.email LIKE :domain ORDER BY u.name ASC'
);
$query->setParameter('domain', '%@gmail.com');
$users = $query->getResult();

// Получить один результат
$user = $em->createQuery(
    'SELECT u FROM App\Entity\User u WHERE u.email = :email'
)->setParameter('email', 'admin@example.com')
 ->getOneOrNullResult();
```

### JOIN-запросы

```php
// LEFT JOIN с выборкой (fetch join -- решает N+1)
$qb = $this->createQueryBuilder('u')
    ->leftJoin('u.posts', 'p')
    ->addSelect('p')                // addSelect загружает посты в одном запросе
    ->where('u.active = true')
    ->getQuery()
    ->getResult();

// INNER JOIN с условием
$qb = $this->createQueryBuilder('p')
    ->innerJoin('p.author', 'a')
    ->addSelect('a')
    ->where('a.active = :active')
    ->andWhere('p.publishedAt IS NOT NULL')
    ->setParameter('active', true);

// JOIN с дополнительным условием
$qb = $this->createQueryBuilder('p')
    ->leftJoin('p.comments', 'c', 'WITH', 'c.approved = true')
    ->addSelect('c');
```

### Динамические фильтры

QueryBuilder удобен для построения запросов с динамическими условиями:

```php
public function search(?string $name, ?string $email, ?bool $active): array
{
    $qb = $this->createQueryBuilder('u');

    if ($name !== null) {
        $qb->andWhere('u.name LIKE :name')
           ->setParameter('name', '%' . $name . '%');
    }

    if ($email !== null) {
        $qb->andWhere('u.email = :email')
           ->setParameter('email', $email);
    }

    if ($active !== null) {
        $qb->andWhere('u.active = :active')
           ->setParameter('active', $active);
    }

    return $qb->orderBy('u.name', 'ASC')
              ->getQuery()
              ->getResult();
}
```

### Агрегация и группировка

```php
// COUNT
$count = $this->createQueryBuilder('u')
    ->select('COUNT(u.id)')
    ->where('u.active = true')
    ->getQuery()
    ->getSingleScalarResult();

// GROUP BY с агрегацией
$stats = $this->createQueryBuilder('o')
    ->select('o.status, COUNT(o.id) as cnt, SUM(o.total) as totalSum')
    ->groupBy('o.status')
    ->having('COUNT(o.id) > :min')
    ->setParameter('min', 5)
    ->getQuery()
    ->getResult();

// Подзапрос
$subQuery = $em->createQueryBuilder()
    ->select('IDENTITY(p.author)')
    ->from(Post::class, 'p')
    ->where('p.publishedAt IS NOT NULL');

$activeAuthors = $this->createQueryBuilder('u')
    ->where($qb->expr()->in('u.id', $subQuery->getDQL()))
    ->getQuery()
    ->getResult();
```

### Пагинация

```php
use Doctrine\ORM\Tools\Pagination\Paginator;

public function findPaginated(int $page, int $limit = 20): Paginator
{
    $query = $this->createQueryBuilder('u')
        ->orderBy('u.createdAt', 'DESC')
        ->setFirstResult(($page - 1) * $limit)
        ->setMaxResults($limit)
        ->getQuery();

    return new Paginator($query);
    // $paginator->count() — общее количество записей
    // foreach ($paginator as $user) — итерация по странице
}
```

### Partial-выборки и DTO

```php
// Выборка только нужных полей (возвращает массивы)
$data = $this->createQueryBuilder('u')
    ->select('u.id, u.name, u.email')
    ->getQuery()
    ->getArrayResult();

// Маппинг на DTO через NEW
$dtos = $em->createQuery(
    'SELECT NEW App\DTO\UserListItem(u.id, u.name, u.email) FROM App\Entity\User u'
)->getResult();
```

### Практические советы

- Используйте `andWhere()` вместо `where()` при добавлении условий динамически -- `where()` перезаписывает предыдущие условия
- Всегда используйте параметры (`:param` + `setParameter()`) -- это защита от SQL-инъекций
- `getResult()` возвращает массив объектов, `getArrayResult()` -- массив ассоциативных массивов (быстрее, без tracking), `getSingleScalarResult()` -- одно скалярное значение
- Для отладки DQL/SQL используйте Symfony Profiler или `$query->getSQL()` и `$query->getParameters()`
- Если запрос сложный, не стесняйтесь использовать нативный SQL через `$connection->executeQuery()` -- Doctrine это позволяет

## Примеры

1. DQL: `SELECT u FROM App\\Entity\\User u WHERE u.active = true`.
2. QueryBuilder с динамическими фильтрами `andWhere()` + `setParameter()`.
3. Fetch join: `leftJoin('u.posts', 'p')->addSelect('p')`.

## Доп. теория

1. `getArrayResult()` быстрее, но не возвращает сущности.
2. `where()` перезаписывает условия, используйте `andWhere()` для дополнения.
