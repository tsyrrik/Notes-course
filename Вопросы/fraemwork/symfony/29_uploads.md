## Вопрос: Работа с файлами (загрузка)
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
