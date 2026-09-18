# 17 — Eloquent aloqalari (Relationships)

[← Oldingi: Eloquent asoslari](16-eloquent-asoslari.md) · [Mundarija](README.md) · [Keyingi: Factory va Seeder →](18-factory-va-seeder.md)

---

## Aloqa turlari

| Turi | Misol | Metod |
| --- | --- | --- |
| Bir-ga-bir | `User` → `Profile` | `hasOne` / `belongsTo` |
| Bir-ga-ko'p | `Post` → `Comment` | `hasMany` / `belongsTo` |
| Ko'p-ga-ko'p | `Post` ↔ `Tag` | `belongsToMany` |
| Uzoq orqali | `Country` → `Post` (`User` orqali) | `hasManyThrough` |
| Polimorf | `Comment` → `Post` yoki `Video` | `morphTo` / `morphMany` |

---

## `belongsTo` va `hasMany`

```php
class Post extends Model
{
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');   // posts.user_id → users.id
    }

    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);             // comments.post_id → posts.id
    }
}

class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}
```

Tashqi kalit nomi konvensiyadan kelib chiqadi: `belongsTo(User::class)` → `user_id`; `hasMany(Comment::class)` → `post_id`. Boshqacha bo'lsa ikkinchi argumentda ko'rsatasiz.

Ishlatish:

```php
$post->author->name;                 // bog'langan model (ob'ekt)
$post->comments;                     // Collection
$post->comments()->count();          // so'rov (Collection yuklamasdan)
$post->comments()->where('approved', true)->get();

// Yaratish
$post->comments()->create(['body' => 'Zo\'r!', 'user_id' => $user->id]);
$user->posts()->createMany([[...], [...]]);

// Bog'lash
$post->author()->associate($user)->save();
$post->author()->dissociate()->save();
```

---

## `belongsToMany` (ko'p-ga-ko'p)

Oraliq jadval kerak: `post_tag` (alifbo tartibida, birlikda — konvensiya).

```php
// migratsiya
Schema::create('post_tag', function (Blueprint $table) {
    $table->foreignId('post_id')->constrained()->cascadeOnDelete();
    $table->foreignId('tag_id')->constrained()->cascadeOnDelete();
    $table->primary(['post_id', 'tag_id']);
});
```

```php
class Post extends Model
{
    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class)
            ->withTimestamps()
            ->withPivot('sort_order');
    }
}
```

```php
$post->tags;                                   // Collection<Tag>
$post->tags()->attach($tagId);                 // qo'shish
$post->tags()->attach([$id1, $id2], ['sort_order' => 1]);
$post->tags()->detach($tagId);                 // olib tashlash
$post->tags()->sync([1, 2, 3]);                // aynan shu ro'yxat qolsin
$post->tags()->syncWithoutDetaching([4]);
$post->tags()->toggle([1, 5]);

foreach ($post->tags as $tag) {
    $tag->pivot->sort_order;                   // oraliq jadval ustunlari
}
```

---

## Polimorf aloqalar

Bitta `comments` jadvali ham post'ga, ham video'ga izoh saqlasin:

```php
// migratsiya
$table->morphs('commentable');   // commentable_id + commentable_type
```

```php
class Comment extends Model
{
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

`commentable_type` da to'liq klass nomi saqlanadi. Buni qisqartirish tavsiya etiladi:

```php
// AppServiceProvider::boot()
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => Post::class,
    'video' => Video::class,
]);
```

**Nega?** Klass nomini o'zgartirsangiz (namespace ko'chirish) bazadagi eski qiymatlar buziladi. Qisqa nom esa barqaror bo'ladi.

---

## ⚠️ N+1 muammosi — eng muhim mavzu

```php
$posts = Post::all();                  // 1 ta so'rov

foreach ($posts as $post) {
    echo $post->author->name;          // HAR BIR post uchun alohida so'rov!
}
// 100 ta post → 101 ta SQL so'rov
```

Yechim — **eager loading**:

```php
$posts = Post::with('author')->get();           // 2 ta so'rov
$posts = Post::with(['author', 'comments'])->get();
$posts = Post::with('comments.user')->get();    // ichma-ich
$posts = Post::with(['comments' => fn ($q) => $q->latest()->limit(3)])->get();
$posts = Post::with('author:id,name')->get();   // faqat kerakli ustunlar
```

Keyinroq yuklash:

```php
$posts->load('author');
$post->loadMissing('comments');
```

### N+1 ni avtomatik ushlash

```php
// AppServiceProvider::boot()
Model::preventLazyLoading(! app()->isProduction());
// yoki: Model::shouldBeStrict(! app()->isProduction());
```

Endi `with()` siz aloqaga murojaat qilsangiz, **istisno** chiqadi. Bu — productionga sekin kod chiqib ketishining oldini oluvchi eng samarali usul.

> **Symfony bilan solishtirish:** Doctrine'da bu `fetch="EAGER"` yoki DQL'dagi `JOIN FETCH`. Muammo bir xil, yechim ham.

---

## Sanash va mavjudlik

```php
$posts = Post::withCount('comments')->get();
$posts[0]->comments_count;

Post::withCount(['comments as approved_count' => fn ($q) => $q->where('approved', true)])->get();

Post::has('comments')->get();                        // izohi borlar
Post::has('comments', '>=', 5)->get();
Post::doesntHave('comments')->get();
Post::whereHas('comments', fn ($q) => $q->where('approved', true))->get();
Post::whereRelation('comments', 'approved', true)->get();   // qisqa shakl

Post::withSum('orders', 'total')->get();
Post::withExists('comments')->get();
```

Agregatni bitta so'rovda olish:

```php
$post->loadCount('comments');
```

---

## Aloqalarga qo'shimcha shartlar

```php
public function publishedComments(): HasMany
{
    return $this->comments()->where('approved', true);
}

// Laravel 13: aloqa orqali yaratilganda ham atribut qo'yilsin
public function featuredPosts(): HasMany
{
    return $this->posts()->withAttributes(['featured' => true]);
}
```

```php
$post = $user->featuredPosts()->create(['title' => '...']);
$post->featured;   // true
```

---

## `hasManyThrough` va `hasOneThrough`

```php
class Country extends Model
{
    // countries → users → posts
    public function posts(): HasManyThrough
    {
        return $this->hasManyThrough(Post::class, User::class);
    }
}
```

---

## Amaliyot

1. `Post`, `Comment`, `Tag` modellarini va ular orasidagi aloqalarni yarating.
2. 10 ta post va har biriga 3 tadan izoh yarating (factory bilan — [18-bob](18-factory-va-seeder.md)).
3. `DB::listen()` ni yoqib, quyidagi ikki variantni solishtiring:

```php
Post::all()->each(fn ($p) => $p->author->name);     // nechta so'rov?
Post::with('author')->get()->each(fn ($p) => $p->author->name);
```

4. `Model::preventLazyLoading()` ni yoqing va birinchi variant istisno berishini ko'ring.
5. `Post::withCount('comments')` bilan ro'yxat chiqaring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/eloquent-relationships>
- <https://laravel.com/docs/13.x/eloquent-relationships#eager-loading>

---

[← Oldingi: Eloquent asoslari](16-eloquent-asoslari.md) · [Mundarija](README.md) · [Keyingi: Factory va Seeder →](18-factory-va-seeder.md)
