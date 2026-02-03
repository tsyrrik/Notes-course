## Вопрос: Filesystem / Storage
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Storage — единый API для локальных дисков и облачных (S3/FTP).

## Ответ
Абстракция над файловыми системами (Flysystem). Диски задаются в `config/filesystems.php`.
```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('reports/report.txt', 'Example');
$content = Storage::disk('local')->get('reports/report.txt');

if (Storage::disk('s3')->exists('images/photo.jpg')) { /* ... */ }
$path = $request->file('avatar')->store('avatars', 'public');
```
Типичные диски: local, public, s3, ftp и др.

## Примеры

1. `Storage::disk('s3')->put($path, $content)`.
2. `php artisan storage:link`.
3. Публичные/приватные диски через `visibility`.

## Доп. теория

1. Файлы на S3 требуют настроек ключей и регионов.
2. Публичные файлы — через `storage:link`.
