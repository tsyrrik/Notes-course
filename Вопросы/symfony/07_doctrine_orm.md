# Doctrine ORM (основы)

Простыми словами: Doctrine — ORM Data Mapper; сущности — обычные классы, маппинг через атрибуты/аннотации/YAML, запросы через репозитории и QueryBuilder.

```php
#[ORM\Entity]
class User {
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type:'integer')]
    private int $id;
    #[ORM\Column(length:255)] private string $email;
}
```

Репозиторий:
```php
$user = $userRepo->find($id);
$active = $userRepo->findBy(['active'=>true], ['createdAt'=>'DESC']);
```
