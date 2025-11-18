# API Resources

Простыми словами: API Resource превращает модель в аккуратный JSON с нужными полями/условиями.
Трансформеры для JSON-ответов.
```bash
php artisan make:resource UserResource
```
```php
class UserResource extends JsonResource {
    public function toArray($request) {
        return [
            'id'=>$this->id,
            'name'=>$this->name,
            'email'=>$this->email,
            'created_at'=>$this->created_at->toDateTimeString(),
            'posts_count'=>$this->whenLoaded('posts', fn() => $this->posts->count()),
            'is_admin'=>$this->when($this->isAdmin(), true),
        ];
    }
}
```
Коллекции: `php artisan make:resource UserCollection` + пагинация в `toArray`.
