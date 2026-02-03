## Вопрос: Clickjacking

## Простой ответ
- Запрещаем встраивать наш сайт в чужие iframes с помощью `X-Frame-Options`/`frame-ancestors`.

## Ответ
Простыми словами: жертве подсовывают невидимый фрейм с вашим сайтом и заставляют кликнуть по кнопке под видом другого интерфейса.

## Защита
- Заголовки `X-Frame-Options: DENY|SAMEORIGIN` или `Content-Security-Policy: frame-ancestors 'none'|'self' <origins>`.
- Проверка `Origin/Referer` на чувствительных эндпоинтах.
- UI-защита: подтверждения, double-submit, CAPTCHA на опасных действиях (дополнительно).

## Примеры

1. Невидимый iframe с кнопкой «Удалить аккаунт».
2. Защита: `CSP frame-ancestors 'none'`.
3. Защита: `X-Frame-Options: DENY`.

## Доп. теория

1. Clickjacking — UI‑атака, а не про XHR/fetch.
2. `frame-ancestors` предпочтительнее `X-Frame-Options`.
