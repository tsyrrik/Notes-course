## Вопрос: Doctrine ORM (основы)
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Doctrine — ORM Data Mapper; сущности — обычные классы, маппинг через атрибуты/аннотации/YAML, запросы через репозитории и QueryBuilder.

## Ответ

### Что такое Doctrine ORM

Doctrine ORM -- это реализация паттерна **Data Mapper** для PHP. В отличие от Active Record (как в Eloquent/Laravel), сущности Doctrine -- это обычные PHP-классы (POPO -- Plain Old PHP Objects), которые ничего не знают о базе данных. Маппинг между объектами и таблицами описывается через метаданные (атрибуты PHP 8, XML или YAML). EntityManager отвечает за синхронизацию объектов с БД.

### Сущности (Entity)

Сущность -- это PHP-класс, который Doctrine маппит на строку таблицы. В Symfony 7 маппинг описывается через PHP-атрибуты:

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: UserRepository::class)]
#[ORM\Table(name: 'users')]
#[ORM\HasLifecycleCallbacks]
class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(length: 180, unique: true)]
    private string $email;

    #[ORM\Column(length: 255)]
    private string $name;

    #[ORM\Column(type: 'boolean', options: ['default' => true])]
    private bool $active = true;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\PrePersist]
    public function setCreatedAtValue(): void
    {
        $this->createdAt = new \DateTimeImmutable();
    }

    // Геттеры и сеттеры...
}
```

### EntityManager и Unit of Work

Doctrine использует паттерн **Unit of Work** -- EntityManager собирает все изменения в объектах и при вызове `flush()` генерирует оптимальный набор SQL-запросов. Это означает, что можно изменить несколько объектов, и все изменения будут записаны в одной транзакции:

```php
use Doctrine\ORM\EntityManagerInterface;

class UserService
{
    public function __construct(private EntityManagerInterface $em) {}

    public function createUser(string $email, string $name): User
    {
        $user = new User();
        $user->setEmail($email);
        $user->setName($name);

        $this->em->persist($user);  // Запланировать INSERT
        $this->em->flush();         // Выполнить все ожидающие SQL

        return $user;
    }

    public function updateUser(User $user, string $newName): void
    {
        $user->setName($newName);   // Просто изменяем объект
        $this->em->flush();         // Doctrine сама обнаружит изменение (dirty checking)
    }

    public function deleteUser(User $user): void
    {
        $this->em->remove($user);   // Запланировать DELETE
        $this->em->flush();
    }
}
```

### Репозитории

Репозиторий -- класс для инкапсуляции запросов к БД. Стандартный `ServiceEntityRepository` предоставляет базовые методы `find()`, `findBy()`, `findOneBy()`, `findAll()`:

```php
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

class UserRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, User::class);
    }

    /**
     * @return User[]
     */
    public function findActiveUsers(): array
    {
        return $this->createQueryBuilder('u')
            ->andWhere('u.active = :active')
            ->setParameter('active', true)
            ->orderBy('u.createdAt', 'DESC')
            ->getQuery()
            ->getResult();
    }

    public function findByEmailDomain(string $domain): array
    {
        return $this->createQueryBuilder('u')
            ->andWhere('u.email LIKE :domain')
            ->setParameter('domain', '%@' . $domain)
            ->getQuery()
            ->getResult();
    }
}
```

### Типы колонок и специальные маппинги

```php
#[ORM\Column(type: 'string', length: 255)]              // VARCHAR(255)
#[ORM\Column(type: 'text')]                               // TEXT
#[ORM\Column(type: 'integer')]                             // INT
#[ORM\Column(type: 'bigint')]                              // BIGINT
#[ORM\Column(type: 'float')]                               // FLOAT
#[ORM\Column(type: 'decimal', precision: 10, scale: 2)]   // DECIMAL(10,2)
#[ORM\Column(type: 'boolean')]                             // BOOLEAN
#[ORM\Column(type: 'datetime_immutable')]                  // DATETIME
#[ORM\Column(type: 'date_immutable')]                      // DATE
#[ORM\Column(type: 'json')]                                // JSON
#[ORM\Column(type: 'simple_array')]                        // VARCHAR, хранит через запятую
#[ORM\Column(enumType: UserStatus::class)]                 // PHP enum (Doctrine 2.14+/3.x)
```

### Lifecycle Callbacks и Events

Doctrine поддерживает хуки жизненного цикла сущности:

```php
#[ORM\HasLifecycleCallbacks]
class User
{
    #[ORM\PrePersist]   // Перед INSERT
    public function onPrePersist(): void { /* ... */ }

    #[ORM\PostPersist]  // После INSERT
    #[ORM\PreUpdate]    // Перед UPDATE
    #[ORM\PostUpdate]   // После UPDATE
    #[ORM\PreRemove]    // Перед DELETE
    #[ORM\PostLoad]     // После загрузки из БД
}
```

### Практические советы

- Используйте `DateTimeImmutable` вместо `DateTime` в сущностях -- это предотвращает случайные мутации дат
- Не вызывайте `flush()` в репозиториях -- это ответственность сервисного слоя
- Для массовых операций используйте `$em->getConnection()->executeStatement()` (raw SQL) или batch processing с периодическим `flush()` + `clear()`
- Генерируйте сущности через MakerBundle: `php bin/console make:entity User`
- Используйте `doctrine:schema:validate` для проверки корректности маппинга
- В Symfony 7 рекомендуется Doctrine ORM 3.x с PHP-атрибутами, типизацией и enum-маппингом

## Примеры

1. Создание сущности: `php bin/console make:entity User`.
2. Сохранение: `$em->persist($user); $em->flush();`.
3. Запрос через репозиторий: `$repo->findBy(['active' => true]);`.

## Доп. теория

1. Doctrine работает по паттерну Unit of Work и откладывает SQL до `flush()`.
2. Для массовых операций лучше использовать DQL/SQL, а не тысячи сущностей.
