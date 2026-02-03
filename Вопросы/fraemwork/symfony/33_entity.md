## Вопрос: Entity (Doctrine)
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Entity — класс домена, который Doctrine мапит на строку таблицы в БД.

## Ответ
- Обычный PHP-класс с атрибутами/аннотациями: id, поля, связи, геттеры/сеттеры.
- Управление жизненным циклом: Doctrine следит за изменениями и сохраняет через EntityManager.
- Связи: OneToOne, ManyToOne/OneToMany, ManyToMany.

```php
#[ORM\Entity(repositoryClass: UserRepository::class)]
class User {
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type:'integer')]
    private int $id;
    #[ORM\Column(length:255)]
    private string $email;
}
```

## Примеры

1. `#[ORM\Entity]` + `#[ORM\Column]` на свойствах.
2. Связь `ManyToOne` к `User`.
3. `make:entity` для генерации полей.

## Доп. теория

1. Entity — POPO, бизнес‑логика допустима, но без зависимостей от инфраструктуры.
2. Не храните в Entity сервисы контейнера.
