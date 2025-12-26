## Вопрос: Doctrine ORM (основы)
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
Doctrine — ORM Data Mapper; сущности — обычные классы, маппинг через атрибуты/аннотации/YAML, запросы через репозитории и QueryBuilder.

## Ответ
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
