## Вопрос: Безопасность (firewalls, auth, access control)
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
Security компонент настраивает правила входа, firewalls, user providers, access control. Поддерживает сессии, JWT (через пакеты), remember-me, роли/векселя.

## Ответ
```yaml
# config/packages/security.yaml (пример)
security:
  password_hashers:
    Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'

  providers:
    app_user_provider:
      entity:
        class: App\Entity\User
        property: email

  firewalls:
    main:
      provider: app_user_provider
      form_login: ~
      logout: ~
      anonymous: true

  access_control:
    - { path: ^/admin, roles: ROLE_ADMIN }
```
