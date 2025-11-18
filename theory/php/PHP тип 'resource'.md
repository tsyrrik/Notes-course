
## Коротко

`resource` — специальный внутренний тип PHP, который указывает на «внешний» объект из расширения/С-мира (файл, сокет, процесс и т.д.). Это **не PHP-объект** и **не скаляр**. Его нельзя объявить в сигнатуре параметра/возврата, нельзя сериализовать, и он живёт своей жизнью до явного закрытия или конца запроса/процесса.

С PHP 8.x большинство исторических ресурсов мигрировали в нормальные классы. Но «потоки» и пара родственных сущностей всё ещё возвращаются как `resource`.

---

## Что под капотом

- В zval это особый тег «ресурс» (внутри — `zend_resource` с числовым дескриптором).
    
- `(int)$res` даёт **ID ресурса**. Идентификаторы могут **переиспользоваться** после `fclose()` и т.п.
    
- Есть **счётчик ссылок**; при обнулении — вызывается деструктор ресурса и он закрывается.
    
- В классических PHP-FPM/CLI ресурсы авто-закрываются на завершение запроса/процесса. В **долго живущих воркерах** (RoadRunner, Swoole, ReactPHP, воркеры Messenger) обязан закрывать сам, иначе утечёшь.
    

Полезные проверки:

```php
is_resource($res);               // true/false
get_resource_type($res);         // например: "stream", "curl", "process", "stream-context"
```

Сериализация:

```php
serialize($res); // не имеет смысла: ресурс не сериализуем, на выходе будет null/пустышка + предупреждение
```

---

## Типизация: чего нельзя и что делать вместо

**Нельзя** так:

```php
function read(resource $r): string {}           // Ошибка: `resource` нельзя в сигнатуре
function open(): resource {}                    // Тоже нельзя
```

### Варианты

1. **Принимать `mixed` + рантайм-проверки** (нелюбимо, но работает):
    

```php
function read($stream): string {
    if (!is_resource($stream) || get_resource_type($stream) !== 'stream') {
        throw new InvalidArgumentException('Expected stream resource');
    }
    return stream_get_contents($stream) ?: '';
}
```

2. **Документировать через PHPDoc**:
    

```php
/**
 * @param resource $stream
 */
function drain($stream): void { /* ... */ }
```

3. **Поддержка «объектов вместо ресурсов»** в новых PHP:
    

```php
/**
 * @param resource|\CurlHandle $ch
 */
function setCurlTimeout($ch, int $sec): void {
    // для PHP < 8 это ресурс, для 8+ — объект CurlHandle
    curl_setopt($ch, CURLOPT_TIMEOUT, $sec);
}
```

4. **Обёртка-класс** вокруг ресурса, чтобы иметь нормальный type-hint:
    

```php
final class StreamHandle {
    /** @var resource */
    public $stream;
    public function __construct($stream) {
        if (!is_resource($stream) || get_resource_type($stream) !== 'stream') {
            throw new InvalidArgumentException('Not a stream resource');
        }
        $this->stream = $stream;
    }
}
function readStream(StreamHandle $h): string { return stream_get_contents($h->stream) ?: ''; }
```

---

## Что в PHP 8.x всё ещё возвращает `resource`

Типы и примеры, которые на 8.2/8.3 остаются ресурсами (не объектами):

|Категория|Функция|Возвращает|Тип ресурса|
|---|---|---|---|
|Потоки/файлы|`fopen()`, `tmpfile()`, `popen()`|resource|`stream`|
|Сокеты через stream-обёртки|`fsockopen()`, `pfsockopen()`|resource|`stream`|
|Универсальные сокеты (stream API)|`stream_socket_client()`, `stream_socket_server()`|resource|`stream`|
|Контексты потоков|`stream_context_create()`, `stream_context_get_options()`|resource|`stream-context`|
|Фильтры потоков|`stream_filter_append()`, `stream_filter_prepend()`|resource|`stream-filter`|
|Директории (через `opendir`)|`opendir()`|resource|`Directory`|
|Процессы|`proc_open()`|resource|`process`|
|Архивация через zlib|`gzopen()`|resource|`stream`|

> Примечания  
> • `dir()` возвращает объект `Directory`, но `opendir()` — **ресурс**.  
> • Все стандартные обёртки потоков (`file://`, `php://memory`, `php://temp`, `compress.zlib://`, `zip://`) тоже дают `stream`-ресурс через `fopen()`.

---

## Что **перестало** быть ресурсом в 8.x и стало объектами

Добро пожаловать в цивилизацию:

- cURL → **`CurlHandle`**, **`CurlMultiHandle`**, **`CurlShareHandle`** (с PHP 8.0)
    
- GD image → **`GdImage`** (8.0)
    
- OpenSSL → **`OpenSSLAsymmetricKey`**, **`OpenSSLCertificate`**, **`OpenSSLCertificateSigningRequest`** (8.0)
    
- Sockets (ext-sockets) → **`Socket`** (8.0). Если юзаешь `socket_create()` из sockets-API, получишь объект; а `fsockopen()` даёт stream-ресурс.
    
- PostgreSQL (ext-pgsql) → **`PgSql\Connection`**, **`PgSql\Result`**, **`PgSql\Lob`** (8.1)
    
- FTP → **`FTP\Connection`** (8.1)
    
- Hash → **`HashContext`** (8.0)
    
- XML parser (ext-xml) → **`XMLParser`** (8.0)
    
- GMP, SQLite3, PDO, mysqli, Imagick — уже давно объекты
    

---

## Примеры (актуально на PHP 8.3)

### Поток файла

```php
$fp = fopen(__DIR__ . '/data.log', 'ab');   // resource(stream)|false
if ($fp === false) {
    throw new RuntimeException('Open failed');
}

try {
    fwrite($fp, "hello\n");
    fflush($fp);
    // что это за ресурс:
    // var_dump(get_resource_type($fp)); // "stream"
    // var_dump((int)$fp);                // целочисленный id
} finally {
    fclose($fp); // закрыть обязательно в длинноживущих процессах
}
```

### Процесс через `proc_open`

```php
$descriptors = [
    0 => ['pipe', 'r'],  // stdin
    1 => ['pipe', 'w'],  // stdout
    2 => ['pipe', 'w'],  // stderr
];
$proc = proc_open(['php', '-v'], $descriptors, $pipes); // resource(process)|false
if (!is_resource($proc)) {
    throw new RuntimeException('proc_open failed');
}
fclose($pipes[0]);
$stdout = stream_get_contents($pipes[1]); fclose($pipes[1]);
$stderr = stream_get_contents($pipes[2]); fclose($pipes[2]);
$status = proc_close($proc);
// ... используем $stdout/$stderr, проверяем $status
```

---

## Практические советы

- Всегда проверяй на **`false`**: многие функции возвращают `resource|false`.
    
- Закрывай: `fclose()`, `pclose()`, `proc_close()`, `stream_socket_shutdown()` — особенно важно в **воркерах**.
    
- Логируй ID ресурса: `(int)$res` помогает отлавливать утечки и двойные закрытия.
    
- Если нужен **type-hint**, делай **обёртку** или принимай объект в новых PHP и ресурс в старых (см. union в PHPDoc).
    
- Не пытайся класть ресурс в кэш/очередь/сессию: это **не сериализуется** и бессмысленно вне текущего процесса.
