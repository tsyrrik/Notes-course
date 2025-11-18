# Eloquent ORM: что это

Простыми словами: Eloquent — ORM ActiveRecord, таблицы становятся классами, строки — объектами, связи описываются методами.
- ActiveRecord ORM: каждая таблица представлена моделью.
- Инкапсулирует доступ к данным, связи, события, аксессоры/мутаторы, касты, soft deletes.
```php
class User extends Model {
    protected $table = 'users';
    protected $fillable = ['name','email','password'];
    protected $hidden = ['password'];
    protected $casts = ['created_at' => 'datetime'];
}

$user = User::create(['name'=>'John','email'=>'john@example.com']);
$users = User::where('active',1)->orderBy('name')->get();
```

## Типы связей
- hasOne / hasMany, belongsTo, belongsToMany, hasManyThrough, morphOne/many (полиморфные).
```php
class User extends Model {
    public function profile() { return $this->hasOne(Profile::class); }
    public function posts() { return $this->hasMany(Post::class); }
    public function roles() { return $this->belongsToMany(Role::class); }
}
```
