# GET vs POST
| Характеристика | GET | POST |
| --- | --- | --- |
| Данные в URL | Да | Нет |
| Идемпотентность | Да | Нет |
| Кэширование | Можно | Нет |
| Макс. длина | Ограничена (~2048) | Не ограничена |
| Использование | Получение данных | Отправка данных |
- Семантика важнее: можно передавать тело в GET и query в POST, но это некорректно. POST не кешируется и не идемпотентен.

```bash
curl -X GET "https://api.example.com/users?limit=10"
curl -X POST "https://api.example.com/users" -H "Content-Type: application/json" -d '{"email":"a@b.c"}'
```
