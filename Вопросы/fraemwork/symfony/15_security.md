## Вопрос: Безопасность (firewalls, auth, access control)
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Security компонент настраивает правила входа, firewalls, user providers, access control. Поддерживает сессии, JWT (через пакеты), remember-me, роли/векселя.

## Ответ

### Архитектура Security в Symfony

Компонент Security в Symfony отвечает за две ключевые задачи: **аутентификация** (кто вы?) и **авторизация** (что вам разрешено?). Система построена на трёх основных концепциях: firewalls (определяют как аутентифицировать), user providers (откуда загружать пользователей) и access control (правила доступа). В Symfony 7 используется система аутентификаторов (Authenticator-based security), введённая в Symfony 5.3.

### Конфигурация security.yaml

```yaml
# config/packages/security.yaml
security:
    # Алгоритм хэширования паролей
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
        # 'auto' выберет лучший алгоритм (bcrypt/argon2i/argon2id)

    # Источники пользователей
    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email   # По какому полю искать пользователя

    # Файрволы -- правила аутентификации
    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false    # Отключить Security для dev-маршрутов

        main:
            lazy: true          # Загружать пользователя только при обращении
            provider: app_user_provider

            # Form Login
            form_login:
                login_path: app_login
                check_path: app_login
                default_target_path: app_dashboard
                enable_csrf: true

            # Logout
            logout:
                path: app_logout
                target: app_home

            # Remember Me
            remember_me:
                secret: '%kernel.secret%'
                lifetime: 604800    # 7 дней
                path: /

            # Или Custom Authenticator
            custom_authenticators:
                - App\Security\ApiTokenAuthenticator

    # Правила доступа (проверяются сверху вниз, первое совпавшее применяется)
    access_control:
        - { path: ^/login, roles: PUBLIC_ACCESS }
        - { path: ^/api/login, roles: PUBLIC_ACCESS }
        - { path: ^/admin, roles: ROLE_ADMIN }
        - { path: ^/profile, roles: ROLE_USER }
        - { path: ^/api, roles: IS_AUTHENTICATED_FULLY }

    # Иерархия ролей
    role_hierarchy:
        ROLE_MODERATOR: ROLE_USER
        ROLE_ADMIN: [ROLE_MODERATOR, ROLE_ALLOWED_TO_SWITCH]
        ROLE_SUPER_ADMIN: ROLE_ADMIN
```

### Сущность User

```php
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;

#[ORM\Entity(repositoryClass: UserRepository::class)]
#[UniqueEntity(fields: ['email'], message: 'Этот email уже зарегистрирован')]
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 180, unique: true)]
    private string $email;

    #[ORM\Column(type: 'json')]
    private array $roles = [];

    #[ORM\Column]
    private string $password;

    public function getUserIdentifier(): string
    {
        return $this->email;
    }

    public function getRoles(): array
    {
        $roles = $this->roles;
        $roles[] = 'ROLE_USER';  // Гарантируем базовую роль
        return array_unique($roles);
    }

    public function getPassword(): string
    {
        return $this->password;
    }

    public function eraseCredentials(): void
    {
        // Очистить временные чувствительные данные (plainPassword и т.п.)
    }
}
```

### Custom Authenticator

Для кастомной логики аутентификации (API-токены, JWT, OAuth) создаётся свой authenticator:

```php
use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;

class ApiTokenAuthenticator extends AbstractAuthenticator
{
    public function supports(Request $request): ?bool
    {
        return $request->headers->has('X-API-TOKEN');
    }

    public function authenticate(Request $request): Passport
    {
        $token = $request->headers->get('X-API-TOKEN');
        if (!$token) {
            throw new CustomUserMessageAuthenticationException('Токен не предоставлен');
        }

        return new SelfValidatingPassport(
            new UserBadge($token, function (string $token) {
                return $this->userRepository->findOneBy(['apiToken' => $token]);
            })
        );
    }

    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
    {
        return null; // Продолжить обработку запроса
    }

    public function onAuthenticationFailure(Request $request, AuthenticationException $exception): ?Response
    {
        return new JsonResponse(['error' => $exception->getMessage()], 401);
    }
}
```

### Voters -- гранулярная авторизация

Для сложной логики авторизации (например, "пользователь может редактировать только свои посты") используются Voters:

```php
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class PostVoter extends Voter
{
    public const EDIT = 'POST_EDIT';
    public const DELETE = 'POST_DELETE';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::EDIT, self::DELETE]) && $subject instanceof Post;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        if (!$user instanceof User) return false;

        /** @var Post $post */
        $post = $subject;

        return match ($attribute) {
            self::EDIT => $post->getAuthor() === $user,
            self::DELETE => $post->getAuthor() === $user || in_array('ROLE_ADMIN', $user->getRoles()),
            default => false,
        };
    }
}

// Использование в контроллере
#[Route('/posts/{id}/edit', name: 'post_edit')]
public function edit(Post $post): Response
{
    $this->denyAccessUnlessGranted('POST_EDIT', $post);
    // ...
}
```

```twig
{# В Twig #}
{% if is_granted('POST_EDIT', post) %}
    <a href="{{ path('post_edit', {id: post.id}) }}">Редактировать</a>
{% endif %}
```

### Атрибут #[IsGranted] и #[Security]

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_ADMIN')]  // На весь контроллер
class AdminController extends AbstractController
{
    #[Route('/admin/users/{id}/ban')]
    #[IsGranted('ROLE_SUPER_ADMIN')]  // Дополнительная проверка на метод
    public function banUser(User $user): Response { /* ... */ }
}
```

### Практические советы

- Используйте `password_hashers: 'auto'` -- Symfony выберет лучший алгоритм для вашей платформы
- `role_hierarchy` позволяет избежать дублирования ролей -- ADMIN автоматически имеет все права USER
- Voters -- предпочтительный механизм авторизации для бизнес-логики (вместо проверок в контроллере)
- Для JWT-аутентификации используйте `lexik/jwt-authentication-bundle`
- Для OAuth/Social login используйте `knpuniversity/oauth2-client-bundle`
- `lazy: true` в firewall -- важная оптимизация, пользователь из сессии загружается только при первом вызове `getUser()`
- В Symfony 7 `anonymous: true` удалён, вместо этого используйте `PUBLIC_ACCESS` в access_control

## Примеры

1. `access_control` для `/admin` → `ROLE_ADMIN`.
2. Custom authenticator по заголовку `X-API-TOKEN`.
3. Voter для проверки прав на редактирование поста.

## Доп. теория

1. `PUBLIC_ACCESS` заменяет старую `anonymous: true`.
2. Авторизацию бизнес‑логики лучше держать в Voter‑ах.
