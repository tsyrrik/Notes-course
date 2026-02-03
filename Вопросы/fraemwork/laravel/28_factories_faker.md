## Вопрос: Factory и Faker в тестах
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Factory+Faker генерирует тестовые данные и связанные модели для сценариев.

## Ответ

Factories и Faker -- это инструменты Laravel для генерации реалистичных тестовых данных. Factory определяет "шаблон" для создания экземпляров моделей с заполненными полями, а Faker генерирует правдоподобные случайные значения (имена, email, адреса, тексты, даты). Вместе они позволяют создавать тестовые сценарии с минимальным количеством кода, заменяя ручное описание каждого поля.

Factories располагаются в `database/factories/` и следуют конвенции именования `ModelFactory`. В Laravel 11 фабрики основаны на классах (class-based), что даёт type-hints, автодополнение и возможность наследования.

### Создание Factory

```bash
php artisan make:factory UserFactory --model=User
php artisan make:factory PostFactory --model=Post
```

### Определение Factory

```php
namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'name'              => fake()->name(),        // "John Smith"
            'email'             => fake()->unique()->safeEmail(),  // "john@example.org"
            'email_verified_at' => now(),
            'password'          => bcrypt('password'),     // Один пароль для удобства тестов
            'phone'             => fake()->phoneNumber(),  // "+1-555-123-4567"
            'avatar'            => fake()->imageUrl(200, 200, 'people'),
            'bio'               => fake()->paragraph(3),
            'birthday'          => fake()->dateTimeBetween('-60 years', '-18 years'),
            'remember_token'    => Str::random(10),
        ];
    }

    // States -- модификации дефолтного набора полей
    public function admin(): static
    {
        return $this->state(fn(array $attributes) => [
            'role'     => 'admin',
            'is_admin' => true,
        ]);
    }

    public function unverified(): static
    {
        return $this->state(fn(array $attributes) => [
            'email_verified_at' => null,
        ]);
    }

    public function banned(): static
    {
        return $this->state(fn(array $attributes) => [
            'banned_at' => now(),
            'ban_reason' => fake()->sentence(),
        ]);
    }
}
```

### Factory для модели со связями

```php
class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id'      => User::factory(),  // Автоматически создаст User
            'title'        => fake()->sentence(6),
            'slug'         => fake()->unique()->slug(),
            'body'         => fake()->paragraphs(5, true),
            'is_published' => fake()->boolean(80),  // 80% вероятность true
            'views_count'  => fake()->numberBetween(0, 10000),
            'published_at' => fake()->dateTimeBetween('-1 year', 'now'),
        ];
    }

    public function draft(): static
    {
        return $this->state(fn() => [
            'is_published' => false,
            'published_at' => null,
        ]);
    }

    // Callback после создания
    public function configure(): static
    {
        return $this->afterCreating(function (Post $post) {
            // Добавить теги после создания
            $post->tags()->attach(
                Tag::factory(3)->create()
            );
        });
    }
}
```

### Использование в тестах

```php
// Создать одного пользователя (сохраняет в БД)
$user = User::factory()->create();

// Создать без сохранения (только модель в памяти)
$user = User::factory()->make();

// Массовое создание
$users = User::factory(10)->create();

// С переопределением полей
$user = User::factory()->create([
    'email' => 'specific@test.com',
    'name'  => 'Test User',
]);

// Комбинирование states
$user = User::factory()->admin()->unverified()->create();

// Связи: has (hasMany)
$user = User::factory()
    ->has(Post::factory(5)->draft(), 'posts')
    ->create();

// Сокращённый синтаксис (magic method: hasPosts)
$user = User::factory()->hasPosts(3)->create();

// Связи: for (belongsTo)
$user = User::factory()->create();
$posts = Post::factory(5)->for($user)->create();

// Сокращённый синтаксис
$posts = Post::factory(3)->forUser(['name' => 'John'])->create();

// Связи: hasAttached (many-to-many)
$user = User::factory()
    ->hasAttached(
        Role::factory(2),
        ['assigned_at' => now()]  // pivot-данные
    )
    ->create();
```

### Популярные методы Faker

```php
// Персональные данные
fake()->name();                    // "Dr. John Smith"
fake()->firstName();               // "John"
fake()->lastName();                // "Smith"
fake()->email();                   // "john@gmail.com"
fake()->unique()->safeEmail();     // "john@example.org" (уникальный)
fake()->phoneNumber();             // "+1-555-123-4567"

// Текст
fake()->sentence();                // "Lorem ipsum dolor sit amet."
fake()->paragraph();               // Абзац текста
fake()->text(200);                 // Текст до 200 символов
fake()->word();                    // "quia"
fake()->words(3, true);           // "quia voluptas sed"

// Числа и булевы
fake()->numberBetween(1, 100);     // 42
fake()->randomFloat(2, 0, 1000);   // 523.47
fake()->boolean(70);               // true (70% вероятность)
fake()->randomElement(['S', 'M', 'L', 'XL']);  // "M"

// Даты
fake()->dateTimeBetween('-1 year', 'now');   // DateTime
fake()->date('Y-m-d');                      // "2024-03-15"

// Адреса
fake()->address();                 // Полный адрес
fake()->city();                    // "New York"
fake()->country();                 // "Germany"
fake()->postcode();                // "12345"

// Интернет
fake()->url();                     // "https://example.com"
fake()->ipv4();                    // "192.168.1.1"
fake()->slug();                    // "lorem-ipsum-dolor"
fake()->uuid();                    // UUID v4

// Изображения / файлы
fake()->imageUrl(640, 480);        // URL placeholder-изображения
fake()->filePath();                // "/path/to/file.txt"
```

### Использование в Seeders

```php
class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Создать админа
        $admin = User::factory()->admin()->create([
            'email' => 'admin@app.com',
        ]);

        // Создать 50 пользователей, у каждого 1-5 постов
        User::factory(50)
            ->has(Post::factory(rand(1, 5)))
            ->create();
    }
}
```

### Практические советы

Называйте states осмысленно -- `->admin()`, `->draft()`, `->banned()` -- это делает тесты читаемыми. Используйте `fake()->unique()` для полей с уникальным индексом, чтобы избежать коллизий. Метод `make()` быстрее `create()`, так как не обращается к БД -- используйте его, когда запись в БД не нужна. Не создавайте больше данных, чем нужно для теста -- это замедляет тесты. Для русской локали Faker используйте `fake('ru_RU')->name()`. В `configure()` можно добавить afterCreating/afterMaking callbacks для сложной логики (например, добавить связи many-to-many).

## Примеры

1. `User::factory()->count(10)->create()`.
2. Состояния: `User::factory()->admin()->create()`.
3. `faker->email` в factory.

## Доп. теория

1. Фабрики используют `HasFactory` на моделях.
2. State/sequence помогают моделировать сценарии.
