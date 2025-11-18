# Filesystem / Storage

Простыми словами: Storage — единый API для локальных дисков и облачных (S3/FTP).
Абстракция над файловыми системами (Flysystem). Диски задаются в `config/filesystems.php`.
```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('reports/report.txt', 'Example');
$content = Storage::disk('local')->get('reports/report.txt');

if (Storage::disk('s3')->exists('images/photo.jpg')) { /* ... */ }
$path = $request->file('avatar')->store('avatars', 'public');
```
Типичные диски: local, public, s3, ftp и др.
