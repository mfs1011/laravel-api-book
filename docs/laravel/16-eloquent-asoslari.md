# 16 — Eloquent asoslari

[← Oldingi: Migratsiyalar](15-migratsiyalar.md) · [Mundarija](README.md) · [Keyingi: Eloquent aloqalari →](17-eloquent-aloqalar.md)

---

## Active Record — Doctrine'dan asosiy farq

| Doctrine (Data Mapper) | Eloquent (Active Record) |
| --- | --- |
| Entity — oddiy ob'ekt, bazani bilmaydi | Model bazani biladi va so'rov yozadi |
| `$em->persist($p); $em->flush();` | `$post->save();` |
| Repository klassi majburiy | `Post::where(...)` to'g'ridan-to'g'ri |
| Xususiyatlar tipli va aniq | Xususiyatlar bazadagi ustunlardan dinamik keladi |
| `UnitOfWork`, identity map | Har `save()` — darhol SQL |

**Nega Laravel Active Record'ni tanlagan?** Tipik CRUD ilovada kod 2-3 barobar qisqaradi va o'qish oson bo'ladi. Murakkab domen mantiqi kerak bo'lganda siz baribir modeldan tashqarida "Action"/"Service" klasslari yozasiz — Eloquent bunga xalaqit bermaydi.

---

## Model yaratish

```shell
php artisan make:model Post
php artisan make:model Post -mf          # + migratsiya + factory
php artisan make:model Post -a           # + migratsiya, factory, seeder, policy, resource, controller, form requests
```

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['title', 'slug', 'body', 'status', 'published_at'])]
class Post extends Model
{
    use HasFactory;

    protected function casts(): array
    {
        return [
            'published_at' => 'datetime',
            'meta' => 'array',
        ];
    }
}
```

---

## Konvensiyalar (va ularni bekor qilish)

| Konvensiya | Standart | O'zgartirish |
| --- | --- | --- |
| Jadval nomi | klass nomining ko'plik, snake_case shakli (`Post` → `posts`) | `protected $table = 'maqolalar';` yoki `#[Table('maqolalar')]` |
| Birlamchi kalit | `id`, auto-increment, int | `protected $primaryKey = 'uuid';` |
| Vaqt ustunlari | `created_at`, `updated_at` | `public $timestamps = false;` yoki `#[WithoutTimestamps]` |
| Ulanish | standart | `protected $connection = 'reporting';` yoki `#[Connection('reporting')]` |
| Route kaliti | `id` | `getRouteKeyName()` yoki `#[RouteKey('slug')]` |

Laravel 13 da ko'p sozlamani **atribut** bilan yozish mumkin (xossalar ham ishlaydi — ikkalasi ham to'g'ri):

```php
use Illuminate\Database\Eloquent\Attributes\{Table, Fillable, Hidden, Appends, RouteKey, UsePolicy, ObservedBy};

#[Table('maqolalar')]
#[Fillable(['title', 'body'])]
#[Hidden(['internal_note'])]
#[Appends('excerpt')]
#[RouteKey('slug')]
#[UsePolicy(PostPolicy::class)]
#[ObservedBy(PostObserver::class)]
class Post extends Model {}
```

---

## Mass assignment: `$fillable` va `$guarded`

```php
#[Fillable(['title', 'body'])]      // yoki: protected $fillable = ['title', 'body'];
```

`Post::create($request->validated())` faqat ro'yxatdagi maydonlarni to'ldiradi. Ro'yxatda yo'q maydon **jimgina tashlab yuboriladi**.

**Nega bu kerak?** Foydalanuvchi so'rovga `"is_admin": true` qo'shib yuborishi mumkin. Agar hamma maydon ochiq bo'lsa, u o'zini admin qilib qo'yadi. Bu — mass assignment zaifligi.

Teskarisi (hammasi ochiq, ba'zisi yopiq) — ehtiyot bo'ling:

```php
protected $guarded = ['id', 'is_admin'];
```

Ishlab chiqish paytida "jim tashlab yuborish" o'rniga xato olish uchun:

```php
// AppServiceProvider::boot()
Model::preventSilentlyDiscardingAttributes(! app()->isProduction());
```

---

## CRUD

```php
// O'qish
$posts = Post::all();                              // hammasi (kichik jadvallarda)
$posts = Post::where('status', 'published')->get();
$post  = Post::find(1);                            // topilmasa null
$post  = Post::findOrFail(1);                      // topilmasa 404
$post  = Post::where('slug', $slug)->firstOrFail();
$count = Post::where('status', 'draft')->count();
$exists = Post::where('slug', $slug)->exists();

// Yaratish
$post = Post::create(['title' => 'Salom', 'body' => '...']);

$post = new Post;
$post->title = 'Salom';
$post->save();

// Yangilash
$post->update(['title' => 'Yangi sarlavha']);
$post->title = 'Yangi';
$post->save();

Post::where('status', 'draft')->update(['status' => 'archived']);   // ommaviy

// O'chirish
$post->delete();
Post::destroy([1, 2, 3]);
Post::where('views', 0)->delete();
```

Foydali yordamchilar:

```php
Post::firstOrCreate(['slug' => $slug], ['title' => $title]);   // topadi yoki yaratadi
Post::updateOrCreate(['slug' => $slug], ['title' => $title]);  // topadi va yangilaydi yoki yaratadi
Post::firstOrNew(['slug' => $slug]);                            // saqlamaydi

$post->refresh();     // bazadan qayta o'qish
$post->replicate();   // nusxa (saqlanmagan)
$post->is($other);    // bir xil modelmi
$post->wasChanged('title');
$post->isDirty();     // saqlanmagan o'zgarish bormi
$post->getOriginal('title');
```

---

## Cast'lar — ustunni PHP tipiga aylantirish

```php
protected function casts(): array
{
    return [
        'published_at' => 'datetime',
        'is_featured' => 'boolean',
        'price' => 'decimal:2',
        'meta' => 'array',            // JSON ustun ↔ PHP massiv
        'options' => AsCollection::class,
        'status' => PostStatus::class, // PHP enum
        'secret' => 'encrypted',       // bazada shifrlangan holda
        'password' => 'hashed',        // saqlashda avtomatik hash
    ];
}
```

Enum bilan:

```php
enum PostStatus: string
{
    case Draft = 'draft';
    case Published = 'published';
}

$post->status === PostStatus::Published;   // to'g'ridan-to'g'ri solishtirish
```

**Nega cast kerak?** Bazada `published_at` — satr. Cast'siz `$post->published_at->format(...)` ishlamaydi. Cast bilan u `Carbon` ob'ekti bo'ladi.

---

## Accessor va Mutator

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function title(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => ucfirst($value),
        set: fn (string $value) => trim($value),
    );
}

// hisoblanadigan (bazada yo'q) xossa
protected function excerpt(): Attribute
{
    return Attribute::make(
        get: fn () => str($this->body)->limit(100)->value(),
    );
}
```

Hisoblangan xossani JSON'ga qo'shish:

```php
#[Appends('excerpt')]      // yoki: protected $appends = ['excerpt'];
```

---

## Scope — takrorlanuvchi shartlar

Laravel 13 uslubi — `#[Scope]` atributi:

```php
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;

#[Scope]
protected function published(Builder $query): void
{
    $query->where('status', 'published')->whereNotNull('published_at');
}

#[Scope]
protected function ofAuthor(Builder $query, User $author): void
{
    $query->where('user_id', $author->id);
}
```

```php
Post::published()->ofAuthor($user)->latest()->paginate();
```

> Eski uslub — metod nomini `scopePublished()` deb yozish — hali ham ishlaydi, lekin 13 da atribut tavsiya etiladi.

**Global scope** — barcha so'rovlarga avtomatik qo'shiladi:

```php
php artisan make:scope PublishedScope
```

```php
#[ScopedBy([PublishedScope::class])]
class Post extends Model {}

// vaqtincha o'chirish
Post::withoutGlobalScope(PublishedScope::class)->get();
```

---

## Soft delete — "yumshoq o'chirish"

Migratsiyada `$table->softDeletes();`, modelda:

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;
}
```

```php
$post->delete();                  // deleted_at to'ldiriladi, qator qoladi
Post::all();                      // o'chirilganlar KO'RINMAYDI
Post::withTrashed()->get();       // hammasi
Post::onlyTrashed()->get();       // faqat o'chirilganlar
$post->restore();                 // tiklash
$post->forceDelete();             // haqiqiy o'chirish
```

Eskilarini tozalash:

```php
// routes/console.php
Schedule::command('model:prune')->daily();
```

---

## Model hodisalari

`creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restored`, `replicating`.

```php
protected static function booted(): void
{
    static::creating(function (Post $post) {
        $post->slug ??= str($post->title)->slug()->value();
    });
}
```

Ko'p mantiq bo'lsa — Observer ishlating ([26-bob](26-hodisalar-va-observerlar.md)).

---

## Qat'iy rejim (strict mode) — albatta yoqing

```php
// AppServiceProvider::boot()
use Illuminate\Database\Eloquent\Model;

Model::shouldBeStrict(! app()->isProduction());
```

Bu uchta himoyani yoqadi:

1. **Lazy loading taqiqlanadi** → N+1 muammosi darhol istisno bilan ko'rinadi ([17-bob](17-eloquent-aloqalar.md)).
2. Mavjud bo'lmagan xossaga murojaat — istisno (typo'lar ushlanadi).
3. `fillable` ga kirmagan maydon jimgina tashlanmaydi — istisno.

---

## Amaliyot

1. `Post` modelini yarating, `#[Fillable]` va `casts()` ni to'ldiring.
2. Tinker'da:

```php
$post = Post::create(['title' => 'Birinchi post', 'body' => str_repeat('matn ', 50), 'status' => 'draft']);
$post->update(['status' => 'published']);
Post::published()->count();
```

3. `#[Scope] published()` ni yozing va uni sinang.
4. `excerpt` accessor'ini qo'shing, `#[Appends('excerpt')]` bilan JSON'da chiqishini tekshiring.
5. `Model::shouldBeStrict()` ni yoqing va nima o'zgarishini ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/eloquent>
- <https://laravel.com/docs/13.x/eloquent-mutators>

---

[← Oldingi: Migratsiyalar](15-migratsiyalar.md) · [Mundarija](README.md) · [Keyingi: Eloquent aloqalari →](17-eloquent-aloqalar.md)
