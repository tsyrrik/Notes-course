## Вопрос: Связи в Doctrine
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
описываются как свойства с атрибутами/аннотациями, Doctrine управляет join'ами и связями.

## Ответ

### Типы связей в Doctrine

Doctrine поддерживает четыре типа связей между сущностями, аналогичных связям в реляционных базах данных. Каждая связь имеет **owning side** (сторону-владельца, которая хранит внешний ключ) и **inverse side** (обратную сторону). Doctrine отслеживает изменения только на owning side -- это критически важный момент, который часто упускают.

### ManyToOne / OneToMany

Самая распространённая связь. Например, у поста есть один автор, а у автора много постов. `ManyToOne` -- это owning side (таблица `post` содержит `author_id`):

```php
#[ORM\Entity]
class Post
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private string $title;

    // Owning side -- здесь хранится FK
    #[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'posts')]
    #[ORM\JoinColumn(nullable: false)]
    private User $author;
}

#[ORM\Entity]
class User
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    private ?int $id = null;

    // Inverse side -- "зеркало" для удобного доступа
    #[ORM\OneToMany(mappedBy: 'author', targetEntity: Post::class, cascade: ['persist', 'remove'])]
    private Collection $posts;

    public function __construct()
    {
        $this->posts = new ArrayCollection();
    }

    public function addPost(Post $post): self
    {
        if (!$this->posts->contains($post)) {
            $this->posts->add($post);
            $post->setAuthor($this); // Синхронизация owning side!
        }
        return $this;
    }

    public function removePost(Post $post): self
    {
        $this->posts->removeElement($post);
        return $this;
    }
}
```

### OneToOne

Связь один-к-одному. Например, пользователь и его профиль:

```php
#[ORM\Entity]
class User
{
    #[ORM\OneToOne(mappedBy: 'user', cascade: ['persist', 'remove'])]
    private ?Profile $profile = null;
}

#[ORM\Entity]
class Profile
{
    // Owning side
    #[ORM\OneToOne(inversedBy: 'profile')]
    #[ORM\JoinColumn(nullable: false)]
    private User $user;

    #[ORM\Column(type: 'text', nullable: true)]
    private ?string $bio = null;
}
```

### ManyToMany

Связь многие-ко-многим. Doctrine автоматически создаёт промежуточную таблицу:

```php
#[ORM\Entity]
class Article
{
    #[ORM\ManyToMany(targetEntity: Tag::class, inversedBy: 'articles')]
    #[ORM\JoinTable(name: 'article_tag')]
    private Collection $tags;

    public function __construct()
    {
        $this->tags = new ArrayCollection();
    }

    public function addTag(Tag $tag): self
    {
        if (!$this->tags->contains($tag)) {
            $this->tags->add($tag);
        }
        return $this;
    }
}

#[ORM\Entity]
class Tag
{
    #[ORM\ManyToMany(targetEntity: Article::class, mappedBy: 'tags')]
    private Collection $articles;
}
```

### Стратегии загрузки (Fetch)

- **LAZY** (по умолчанию) -- связанные объекты загружаются при первом обращении (через proxy-объекты). Это эффективно, если вы не всегда обращаетесь к связанным данным.
- **EAGER** -- связанные объекты загружаются сразу вместе с основной сущностью. Используется редко, так как всегда делает JOIN.
- **EXTRA_LAZY** -- для коллекций; методы `count()`, `contains()`, `slice()` выполняют оптимизированные SQL-запросы без загрузки всей коллекции.

```php
#[ORM\OneToMany(mappedBy: 'author', targetEntity: Post::class, fetch: 'EXTRA_LAZY')]
private Collection $posts;

// $user->getPosts()->count() — выполнит SELECT COUNT(*), не загружая все посты
```

### Cascade-операции

`cascade` определяет, какие операции EntityManager применяются каскадно к связанным объектам:

```php
#[ORM\OneToMany(mappedBy: 'author', targetEntity: Post::class,
    cascade: ['persist', 'remove'],   // При persist/remove User — то же для Post
    orphanRemoval: true               // Удалять пост, если он больше не связан с User
)]
private Collection $posts;
```

- `cascade: ['persist']` -- при `$em->persist($user)` все новые посты тоже сохранятся
- `cascade: ['remove']` -- при удалении User удалятся все его посты
- `orphanRemoval: true` -- если `$user->removePost($post)`, пост будет удалён из БД при flush

### Проблема N+1 и решение

Lazy loading приводит к проблеме N+1: один запрос для основных объектов + N запросов для загрузки связей. Решение -- использовать fetch join в QueryBuilder:

```php
// N+1 проблема:
$users = $userRepo->findAll(); // 1 запрос
foreach ($users as $user) {
    $user->getPosts();          // N запросов!
}

// Решение — fetch join:
$users = $userRepo->createQueryBuilder('u')
    ->leftJoin('u.posts', 'p')
    ->addSelect('p')            // addSelect решает N+1
    ->getQuery()
    ->getResult();              // 1 запрос с JOIN
```

### Практические советы

- Всегда синхронизируйте обе стороны связи через хелпер-методы `addPost()`/`removePost()`
- Используйте `orphanRemoval: true` для зависимых сущностей (комментарии, элементы заказа)
- Для ManyToMany с дополнительными полями в промежуточной таблице используйте две связи ManyToOne через промежуточную сущность
- Избегайте `fetch: 'EAGER'` -- вместо этого делайте осознанные fetch join в репозитории
- Генерируйте связи через `php bin/console make:entity` -- MakerBundle создаст все нужные методы

## Примеры

1. `Post` → `User` через `ManyToOne` и `author_id`.
2. `Article` ↔ `Tag` через `ManyToMany` с `JoinTable`.
3. Fetch join для устранения N+1: `addSelect('p')`.

## Доп. теория

1. Doctrine отслеживает изменения только на owning side.
2. `orphanRemoval` полезен для зависимых сущностей.
