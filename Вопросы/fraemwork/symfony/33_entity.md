## Вопрос: Entity (Doctrine)
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
