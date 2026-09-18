# 37 — Ilg'or Eloquent va unumdorlik

[← Oldingi: Ko'p tillilik](36-koptillilik-va-mintaqa.md) · [Mundarija](README.md) · [Keyingi: API pro darajasi →](38-api-pro.md)

---

## 1. Custom cast klasslar

Tayyor cast'lar yetmasa (`array`, `datetime`, `encrypted`, `hashed`...), o'zingiznikini yozasiz:

```shell
php artisan make:cast AsMoney
```

```php
namespace App\Casts;

use App\ValueObjects\Money;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

/** @implements CastsAttributes<Money, Money> */
class AsMoney implements CastsAttributes
{
    public function get(Model $model, string $key, mixed $value, array $attributes): ?Money
    {
        return $value === null ? null : new Money((int) $value, $attributes['currency'] ?? 'UZS');
    }

    /** @return array<string, mixed> */
    public function set(Model $model, string $key, mixed $value, array $attributes): array
    {
        if (! $value instanceof Money) {
            throw new \InvalidArgumentException('Money kutilgan edi.');
        }

        return [$key => $value->amountInTiyin, 'currency' => $value->currency];
    }
}
```

```php
protected function casts(): array
{
    return ['price' => AsMoney::class];
}
```

```php
$order->price = new Money(1_500_000_00, 'UZS');
$order->save();
$order->price->format();     // "1 500 000,00 so'm"
```

**Nega qiymat ob'ekti (value object)?** Chunki `$order->price` endi shunchaki son emas — u valyutani ham biladi, formatlashni ham biladi va noto'g'ri turdagi qiymatni qabul qilmaydi. Pul, telefon raqami, koordinata, IP — bularning hammasi qiymat ob'ektiga nomzod.

Parametrli cast:

```php
'secret' => AsHash::class.':sha256',
```

Ob'ekt keshini o'chirish (har murojaatda qaytadan yaratilsin):

```php
class AsAddress implements CastsAttributes
{
    public bool $withoutObjectCaching = true;
}
```

Model JSON'ga aylanganda ishlashi uchun qiymat ob'ektiga `Arrayable` va `JsonSerializable` ni implement qiling.

---

## 2. JSON ustunlar bilan ishlash

```php
$table->json('meta');
$table->json('settings')->default(new Expression("('{}')"));
```

So'rovlar:

```php
Post::where('meta->source', 'telegram')->get();
Post::whereJsonContains('meta->tags', 'laravel')->get();
Post::whereJsonLength('meta->tags', '>', 2)->get();
Post::orderBy('meta->priority')->get();

// yangilash — butun ustunni qayta yozmasdan
Post::where('id', 1)->update(['meta->views' => 10]);
```

> **Qachon JSON, qachon alohida jadval?** JSON — sxemasi barqaror bo'lmagan qo'shimcha ma'lumot uchun (integratsiya javobi, sozlamalar). Agar bo'yicha tez-tez filtr, `join` yoki agregat qilsangiz — **oddiy ustun** qiling. JSON ustunlarni indekslash cheklangan va so'rovlar sekin bo'ladi.

PostgreSQL'da `jsonb` + GIN indeks, MySQL'da generated column + indeks ishlatiladi:

```php
$table->string('source')->virtualAs('meta->>"$.source"')->index();   // MySQL
```

---

## 3. Bir vaqtda o'zgartirish muammosi (concurrency)

Ikki so'rov bir vaqtda bitta balansni o'zgartirsa, biri ikkinchisini "yeb qo'yadi" (lost update).

### Pessimistik qulf

```php
DB::transaction(function () use ($userId, $sum) {
    $wallet = Wallet::where('user_id', $userId)->lockForUpdate()->first();

    if ($wallet->balance < $sum) {
        throw new InsufficientBalanceException($sum, $wallet->balance);
    }

    $wallet->decrement('balance', $sum);
});
```

`lockForUpdate()` — tranzaksiya tugaguncha boshqa so'rov shu qatorni o'qiy olmaydi. `sharedLock()` — o'qishga ruxsat, yozishga yo'q.

### Atomik amallar

```php
Wallet::where('user_id', $userId)->where('balance', '>=', $sum)->decrement('balance', $sum);
```

Bu bitta SQL — qulf kerak emas. `->decrement()` natijasi 0 bo'lsa, mablag' yetmagan.

### Optimistik qulf (versiya ustuni)

```php
$updated = Post::where('id', $post->id)
    ->where('version', $post->version)
    ->update(['title' => $title, 'version' => $post->version + 1]);

if ($updated === 0) {
    abort(409, 'Yozuv boshqa foydalanuvchi tomonidan o\'zgartirilgan.');
}
```

API'da bu **409 Conflict** bilan qaytariladi va mijozga "qayta yuklang" deyiladi ([38-bob](38-api-pro.md)).

### Kesh qulfi (bir nechta server)

```php
Cache::lock("import:{$fileId}", 300)->get(function () {
    // faqat bitta jarayon
});
```

---

## 4. Katta hajm bilan ishlash

```php
// o'qish
Post::where('status', 'draft')->chunkById(500, fn ($posts) => $posts->each->archive());
foreach (Post::lazyById(500) as $post) { /* ... */ }
foreach (Post::cursor() as $post) { /* ... */ }

// yozish — bitta so'rovda ko'p qator
Post::insert($rows);                         // model hodisalari ISHLAMAYDI
Post::upsert($rows, uniqueBy: ['slug'], update: ['title', 'body']);

// o'chirish — bo'laklab
Post::where('created_at', '<', now()->subYear())->chunkById(1000, fn ($p) => $p->each->delete());
```

> **`chunk` va `chunkById` farqi:** agar aylanish ichida shart bo'yicha yozuvlarni o'zgartirsangiz, `chunk` sahifalarni `offset` bilan oladi va **ba'zi yozuvlarni o'tkazib yuboradi**. `chunkById` esa oxirgi ID dan davom etadi — shuning uchun yozish operatsiyalarida doim `chunkById`.

Eski yozuvlarni avtomatik tozalash:

```php
use Illuminate\Database\Eloquent\Prunable;

class Log extends Model
{
    use Prunable;

    public function prunable(): Builder
    {
        return static::where('created_at', '<', now()->subMonth());
    }

    protected function pruning(): void
    {
        // masalan faylni ham o'chirish
    }
}
```

```php
Schedule::command('model:prune')->daily();
```

---

## 5. So'rovlarni tezlashtirish

### Indekslar — eng katta ta'sir

```php
$table->index('status');                       // WHERE status = ?
$table->index(['user_id', 'created_at']);      // WHERE user_id = ? ORDER BY created_at
$table->unique(['post_id', 'user_id']);
```

Qoidalar:

1. `WHERE`, `JOIN`, `ORDER BY` da qatnashadigan ustunlar indekslanadi.
2. Kompozit indeksda **tartib muhim**: `['user_id', 'created_at']` indeksi `user_id` bo'yicha filtrda ishlaydi, faqat `created_at` bo'yicha — yo'q.
3. Ustun ustida funksiya ishlatilsa indeks ishlamaydi: `WHERE lower(email) = ?` — buning uchun alohida ustun yoki funksional indeks kerak.
4. Ortiqcha indeks ham yomon: har `INSERT`/`UPDATE` sekinlashadi.

Tekshirish:

```php
Post::where('status', 'published')->explain()->dd();
```

### So'rovni yengillashtirish

```php
Post::select(['id', 'title', 'user_id'])->with('author:id,name')->paginate(15);
Post::withCount('comments')->get();             // COUNT(*) subquery
Post::withExists('comments')->get();
Post::whereRelation('comments', 'approved', true)->get();

// N ta emas, bitta so'rov bilan "oxirgi izoh"
Post::addSelect(['last_comment' => Comment::select('body')
    ->whereColumn('post_id', 'posts.id')
    ->latest()
    ->limit(1),
])->get();
```

`paginate()` o'rniga `simplePaginate()` yoki `cursorPaginate()` — `COUNT(*)` so'rovi katta jadvalda qimmat ([20-bob](20-api-resurslar.md)).

### Nazorat

```php
// AppServiceProvider::boot()
Model::shouldBeStrict(! app()->isProduction());          // lazy loading taqiqi

DB::whenQueryingForLongerThan(500, fn ($connection) =>
    logger()->warning('Sekin so\'rov', ['db' => $connection->getName()]));

DB::listen(fn ($q) => logger()->debug($q->sql, ['ms' => $q->time]));
```

Har bir so'rovdagi SQL sonini test bilan qotirish:

```php
it('index sahifasi 5 tadan kam so\'rov qiladi', function () {
    Post::factory()->count(20)->published()->create();

    DB::enableQueryLog();
    getJson('/api/posts')->assertOk();

    expect(count(DB::getQueryLog()))->toBeLessThan(5);
});
```

---

## 6. Kesh strategiyalari

| Naqsh | Qachon | Misol |
| --- | --- | --- |
| **Cache-aside** (`remember`) | Umumiy holat | `Cache::remember('stats', 300, fn () => ...)` |
| **Write-through** | Yozuv o'zgarganda darhol yangilash | Observer'da `Cache::put` |
| **Tag bilan tozalash** | Bog'liq kalitlar guruhi (Redis) | `Cache::tags(['posts'])->flush()` |
| **Versiyali kalit** | Tozalash murakkab bo'lsa | `"posts:v{$version}"` |

```php
// Observer — yozuv o'zgarganda keshni bekor qilish
public function saved(Post $post): void
{
    Cache::forget("post:{$post->id}");
    Cache::forget('posts:latest');
}
```

**Cache stampede** (kesh muddati tugagan lahzada yuzlab so'rov bir vaqtda bazaga urilishi) muammosiga qarshi:

```php
Cache::flexible('stats', [60, 300], fn () => $heavy());   // 60s "yangi", 300s gacha "eski, fonda yangilanadi"
Cache::lock('stats:rebuild')->get(fn () => /* ... */);
```

---

## 7. Multi-tenancy (ko'p ijarachi) — qisqacha

Bir ilova, ko'p mijoz. Uch yondashuv:

| Model | Qanday | Qachon |
| --- | --- | --- |
| **Ustun bo'yicha** (`tenant_id`) | Har jadvalga `tenant_id` + global scope | Eng ko'p ishlatiladi |
| **Sxema/baza bo'yicha** | Har ijarachiga alohida baza | Qat'iy izolyatsiya talab qilinsa |
| **Alohida ilova** | Har mijozga alohida deploy | Juda kam |

Ustun yondashuvi:

```php
// app/Models/Scopes/TenantScope.php
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if ($tenantId = app('tenant')?->id) {
            $builder->where($model->getTable().'.tenant_id', $tenantId);
        }
    }
}
```

```php
#[ScopedBy([TenantScope::class])]
class Project extends Model
{
    protected static function booted(): void
    {
        static::creating(fn (Project $p) => $p->tenant_id ??= app('tenant')?->id);
    }
}
```

> **Xavfsizlik:** global scope'ni **unutish** — ma'lumot sizib chiqishining eng keng tarqalgan sababi. Har bir ijarachiga tegishli modelda test yozing: boshqa ijarachi yozuvi ko'rinmasligini tekshiring. `spatie/laravel-multitenancy` yoki `stancl/tenancy` paketlari shu ishni yengillashtiradi.

---

## 8. Octane bilan ishlaganda

Doimiy ishlaydigan serverda (`laravel/octane`) ilova **so'rovlar orasida xotirada qoladi**. Shuning uchun:

- Statik xossalarda so'rovga oid ma'lumot saqlamang.
- `singleton` ichida `Request` yoki foydalanuvchini ushlab turmang — `scoped` ishlating ([05-bob](05-service-container.md)).
- Global o'zgaruvchilarni "tozalash" kerak bo'lsa `Octane::tick()` / `flush` hodisalaridan foydalaning.
- Xotira sizib chiqmasligi uchun worker'larni davriy qayta ishga tushiring (`--max-requests`).

---

## Amaliyot

1. `AsMoney` cast'ini va `Money` qiymat ob'ektini yozing, `price` ustunida sinang.
2. `lockForUpdate()` bilan hamyondan pul yechishni yozing; ikki terminalda bir vaqtda ishga tushirib, balans manfiy bo'lmasligini tekshiring.
3. `chunk` va `chunkById` farqini amalda ko'ring: `chunk` ichida yozuv holatini o'zgartiring va nechta yozuv o'tkazib yuborilganini sanang.
4. 100 000 qator yarating, `paginate()` va `cursorPaginate()` so'rov vaqtini solishtiring.
5. "So'rovlar soni" testini yozing (`DB::getQueryLog()`), keyin `with()` ni olib tashlab, test qulaganini ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/eloquent-mutators#custom-casts>
- <https://laravel.com/docs/13.x/queries#pessimistic-locking>
- <https://laravel.com/docs/13.x/eloquent#pruning-models>
- <https://laravel.com/docs/13.x/cache>
- <https://laravel.com/docs/13.x/octane>

---

[← Oldingi: Ko'p tillilik](36-koptillilik-va-mintaqa.md) · [Mundarija](README.md) · [Keyingi: API pro darajasi →](38-api-pro.md)
