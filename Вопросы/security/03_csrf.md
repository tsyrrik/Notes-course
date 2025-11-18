# CSRF (Cross-Site Request Forgery)
- Заставляет авторизованного пользователя выполнить запрос без его ведома.
## Защита
- CSRF-токен в формах/заголовках, проверка токена на сервере.
- SameSite/Lax/Strict cookies, `Content-Type` проверки, double-submit cookie.
- Для критичных действий — подтверждение паролем/2FA.
```html
<form method="post">
  <input type="hidden" name="_token" value="<?= $token ?>">
</form>
```
