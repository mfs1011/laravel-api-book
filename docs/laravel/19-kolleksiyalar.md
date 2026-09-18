# 19 — Kolleksiyalar (Collections)

[← Oldingi: Factory va Seeder](18-factory-va-seeder.md) · [Mundarija](README.md) · [Keyingi: API Resurslar →](20-api-resurslar.md)

---

## Nima uchun kerak

Eloquent so'rovlari massiv emas, **`Illuminate\Support\Collection`** qaytaradi. Bu — massiv ustidagi "o'ralgan" ob'ekt bo'lib, zanjir shaklida ishlash imkonini beradi:

```php
$emails = User::all()
    ->filter(fn (User $u) => $u->isActive())
    ->sortBy('name')
    ->pluck('email')
    ->take(10)
    ->values()
    ->all();
```

Xuddi shu narsani oddiy massivda `array_filter` + `usort` + `array_column` + `array_slice` bilan yozish uzunroq va o'qish qiyinroq.

Har qanday massivni kolleksiyaga aylantirish:

```php
collect([1, 2, 3])->sum();          // 6
collect(['a' => 1])->toArray();
```

---

## Eng ko'p ishlatiladigan metodlar

### Filtrlash va tanlash

```php
$c->filter(fn ($item) => $item->active);
$c->reject(fn ($item) => $item->banned);
$c->where('status', 'published');
$c->whereIn('status', ['draft', 'published']);
$c->whereNotNull('published_at');
$c->first();
$c->first(fn ($p) => $p->views > 100);
$c->firstWhere('slug', 'salom');
$c->last();
$c->take(5);
$c->skip(5);
$c->unique('email');
$c->only(['id', 'title']);          // massiv kalitlari bo'yicha
```

### O'zgartirish

```php
$c->map(fn ($p) => $p->title);
$c->mapWithKeys(fn ($p) => [$p->id => $p->title]);
$c->flatMap(fn ($p) => $p->tags);
$c->pluck('title');
$c->pluck('title', 'id');           // ['1' => 'Salom', ...]
$c->keyBy('slug');
$c->groupBy('status');
$c->sortBy('created_at');
$c->sortByDesc('views');
$c->sortBy([['status', 'asc'], ['views', 'desc']]);
$c->values();                        // kalitlarni qayta raqamlash
$c->flatten();
$c->chunk(3);
$c->collapse();
```

### Hisoblash

```php
$c->count();
$c->sum('total');
$c->avg('price');
$c->max('views');
$c->min('price');
$c->countBy('status');               // ['draft' => 4, 'published' => 10]
$c->reduce(fn ($carry, $item) => $carry + $item->total, 0);
```

### Tekshirish

```php
$c->isEmpty();
$c->isNotEmpty();
$c->contains('slug', 'salom');
$c->every(fn ($p) => $p->published);
$c->some(fn ($p) => $p->featured);
```

### Boshqa foydali metodlar

```php
$c->each(fn ($p) => $p->publish());
$c->tap(fn ($c) => logger($c->count()));
$c->when($sorted, fn ($c) => $c->sortBy('title'));
$c->partition(fn ($p) => $p->published);   // [ha, yo'q] — ikki kolleksiya
$c->dd();                                   // chiqarib to'xtatish
$c->dump();
```

Eloquent kolleksiyasida qo'shimcha:

```php
$posts->load('author');
$posts->modelKeys();          // [1, 2, 3]
$posts->find(3);
$posts->each->publish();      // "higher order" — har biriga publish() chaqiradi
$posts->toQuery()->update(['status' => 'archived']);
```

---

## ⚠️ Eng muhim tuzoq: kolleksiyada emas, bazada filtrlang

```php
// ❌ YOMON: 100 000 qator PHP xotirasiga yuklanadi
Post::all()->where('status', 'published')->take(10);

// ✅ YAXSHI: filtrlash SQL'da bo'ladi
Post::where('status', 'published')->take(10)->get();
```

Farqi: birinchi variantda `->where()` **kolleksiya** metodi (PHP'da ishlaydi), ikkinchisida **query builder** metodi (SQL'ga aylanadi). Sintaksis o'xshash, natija ham o'xshash, lekin ish unumdorligi yer bilan osmoncha.

**Qoida:** `get()` / `all()` chaqirilgunga qadar — bu so'rov; keyin — kolleksiya.

---

## Lazy collections (katta hajm uchun)

```php
use Illuminate\Support\LazyCollection;

Post::cursor()                       // LazyCollection
    ->filter(fn ($p) => $p->views > 100)
    ->each(fn ($p) => $p->archive());

LazyCollection::make(function () {
    $handle = fopen('katta.log', 'r');
    while (($line = fgets($handle)) !== false) {
        yield $line;
    }
})->chunk(1000)->each(fn ($lines) => /* ... */);
```

Lazy collection generator ustida ishlaydi: barcha ma'lumot bir vaqtda xotirada bo'lmaydi. Tashqi API'ni sekinlashtirib chaqirish uchun:

```php
User::where('vip', true)->cursor()->throttle(seconds: 1)->each(fn ($u) => $api->sync($u));
```

---

## Maxsus kolleksiya klassi

```shell
php artisan make:class Collections/PostCollection
```

```php
use Illuminate\Database\Eloquent\Collection;

class PostCollection extends Collection
{
    public function published(): static
    {
        return $this->filter(fn ($post) => $post->isPublished());
    }
}
```

```php
use Illuminate\Database\Eloquent\Attributes\CollectedBy;

#[CollectedBy(PostCollection::class)]
class Post extends Model {}
```

---

## Amaliyot

1. Tinker'da:

```php
collect([1,2,3,4,5])->filter(fn ($n) => $n % 2)->map(fn ($n) => $n * 10)->values()->all();
```

2. `Post::all()->groupBy('status')->map->count()` natijasini ko'ring.
3. Bir xil natijani ikki usulda oling va `DB::listen()` orqali SQL sonini solishtiring:

```php
Post::all()->where('status', 'published')->count();
Post::where('status', 'published')->count();
```

4. `Post::cursor()` bilan katta ro'yxatni aylanib chiqing va xotira farqini `memory_get_peak_usage()` bilan o'lchang.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/collections>
- <https://laravel.com/docs/13.x/eloquent-collections>

---

[← Oldingi: Factory va Seeder](18-factory-va-seeder.md) · [Mundarija](README.md) · [Keyingi: API Resurslar →](20-api-resurslar.md)
