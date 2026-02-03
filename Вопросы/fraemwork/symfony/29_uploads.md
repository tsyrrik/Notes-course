## Вопрос: Работа с файлами (загрузка)
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
используем `UploadedFile` из запроса, валидируем, генерируем имя, перемещаем в нужную директорию.

## Ответ
```php
public function upload(Request $request, string $uploadDir): JsonResponse {
    /** @var UploadedFile|null $file */
    $file = $request->files->get('upload');
    if (!$file) { return $this->json(['error' => 'no file'], 400); }

    $name = md5(uniqid()).'Notes'.$file->guessExtension();
    $file->move($uploadDir, $name);

    return $this->json(['filename' => $name]);
}
```

- Валидация: Constraint `File/Image` (размер, mime).  
- Настройка путей через параметры/`services.yaml` или `config/services.yaml` параметр `%upload_directory%`.

## Примеры

1. `UploadedFile` + `File` constraint (maxSize, mimeTypes).
2. Генерация безопасного имени через `uniqid()` или `Uuid`.
3. Сохранение пути в сущности и отдача через `BinaryFileResponse`.

## Доп. теория

1. Никогда не доверяйте исходному имени файла.
2. Храните файл вне `public/` если он приватный.
