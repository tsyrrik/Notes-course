## Вопрос: Связи в Doctrine
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
описываются как свойства с атрибутами/аннотациями, Doctrine управляет join'ами и связями.

## Ответ
- OneToOne, OneToMany/ManyToOne, ManyToMany.
- fetch: LAZY (по требованию), EAGER (сразу).

```php
#[ORM\Entity]
class Post {
    #[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'posts')]
    private User $author;
}

#[ORM\Entity]
class User {
    #[ORM\OneToMany(mappedBy: 'author', targetEntity: Post::class)]
    private Collection $posts;
}
```
