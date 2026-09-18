# 27 — Kesh, fayllar va HTTP client

[← Oldingi: Hodisalar](26-hodisalar-va-observerlar.md) · [Mundarija](README.md) · [Keyingi: Mail va bildirishnomalar →](28-mail-va-bildirishnoma.md)

---

## 1. Kesh

Drayverlar: `database` (bu loyihada standart), `redis`, `memcached`, `file`, `array` (testlar uchun), `null`.

```ini
CACHE_STORE=database
```

### Asosiy amallar

```php
use Illuminate\Support\Facades\Cache;

Cache::put('key', 'value', now()->addMinutes(10));
Cache::put('key', 'value');                    // muddatsiz
Cache::add('key', 'value', 60);                // faqat yo'q bo'lsa qo'yadi
Cache::forever('key', 'value');

Cache::get('key');
Cache::get('key', 'default');
Cache::get('key', fn () => $expensive());
Cache::has('key');
Cache::pull('key');                            // o'qib, o'chiradi
Cache::forget('key');
Cache::flush();                                // hammasini (ehtiyot bo'ling)

Cache::increment('views');
Cache::decrement('stock', 3);
```

### Eng ko'p ishlatiladigan naqsh: `remember`

```php
$stats = Cache::remember('dashboard:stats', now()->addMinutes(15), function () {
    return [
        'users' => User::count(),
        'posts' => Post::published()->count(),
    ];
});

$config = Cache::rememberForever('settings', fn () => Setting::pluck('value', 'key'));
```

Ma'nosi: "keshda bo'lsa oladi, bo'lmasa closure'ni bajarib, natijani keshga yozadi".

### Keshni to'g'ri bekor qilish

Eng keng tarqalgan xato — eskirgan ma'lumotni ko'rsatib qolish. Yechim: yozuv o'zgarganda keshni tozalash.

```php
// PostObserver
public function saved(Post $post): void
{
    Cache::forget('posts:latest');
    Cache::forget("post:{$post->id}");
}
```

Yoki kalitga versiya qo'shish:

```php
$key = "post:{$post->id}:".$post->updated_at->timestamp;
```

### Teglar (faqat `redis` / `memcached`)

```php
Cache::tags(['posts', "user:{$id}"])->put('key', $value, 600);
Cache::tags(['posts'])->flush();
```

`database` va `file` drayverlari teglarni qo'llab-quvvatlamaydi.

### Atomik qulflar (lock)

```php
$lock = Cache::lock('import-products', 120);

if ($lock->get()) {
    try {
        // faqat bitta jarayon bu yerga kiradi
    } finally {
        $lock->release();
    }
}

// yoki kutib turish bilan
Cache::lock('report')->block(5, function () {
    // ...
});
```

**Nega kerak?** Bir nechta worker yoki server bir vaqtda bir xil ishni bajarmasligi uchun (masalan ikki marta hisob-faktura chiqarmaslik).

### Kesh nima uchun ishlatilmasligi kerak

Kesh — **hisoblash natijasini** saqlash uchun. Sessiya, navbat yoki muhim ma'lumotni faqat keshda saqlamang: `Cache::flush()` yoki Redis qayta ishga tushishi bilan ular yo'qoladi.

---

## 2. Fayllar (Storage)

`config/filesystems.php` da disklar:

| Disk | Joylashuvi | Ko'rinishi |
| --- | --- | --- |
| `local` | `storage/app/private` | Yopiq |
| `public` | `storage/app/public` | `public/storage` symlink orqali ochiq |
| `s3` | Amazon S3 | Sozlamaga bog'liq |

```shell
php artisan storage:link
```

### Amallar

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('public')->put('avatars/1.jpg', $contents);
Storage::disk('public')->putFile('avatars', $request->file('avatar'));
Storage::disk('public')->putFileAs('avatars', $file, "user-{$id}.jpg");

Storage::get('file.txt');
Storage::exists('file.txt');
Storage::missing('file.txt');
Storage::delete('file.txt');
Storage::copy('a.txt', 'b.txt');
Storage::move('a.txt', 'c.txt');
Storage::size('file.txt');
Storage::lastModified('file.txt');
Storage::files('avatars');
Storage::allFiles('avatars');
Storage::directories('avatars');

Storage::disk('public')->url('avatars/1.jpg');        // to'liq URL
Storage::download('hisobot.pdf');
```

So'rovdagi fayl bilan ([12-bob](12-sorov-va-javob.md)):

```php
$path = $request->file('avatar')->store('avatars', 'public');
$user->update(['avatar_path' => $path]);
```

### Vaqtinchalik (imzolangan) URL

Yopiq fayllarni cheklangan vaqtga ochish:

```php
$url = Storage::disk('s3')->temporaryUrl('invoices/1.pdf', now()->addMinutes(10));
```

Lokal diskda esa imzolangan route ishlating:

```php
Route::get('/files/{file}', function (string $file) {
    abort_unless(request()->hasValidSignature(), 401);

    return Storage::download($file);
})->name('files.show');

URL::temporarySignedRoute('files.show', now()->addMinutes(10), ['file' => 'a.pdf']);
```

### Testlarda

```php
Storage::fake('public');

// ...

Storage::disk('public')->assertExists('avatars/1.jpg');
Storage::disk('public')->assertMissing('avatars/2.jpg');
```

`Storage::fake()` haqiqiy fayl tizimiga tegmaydi — test tugagach hammasi tozalanadi.

---

## 3. HTTP client (tashqi API'lar)

Guzzle ustidagi qulay qatlam:

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('https://api.example.uz/posts', ['page' => 2]);
$response = Http::post('https://api.example.uz/posts', ['title' => 'Salom']);

$response->json();            // massiv
$response->json('data.0.id');
$response->body();
$response->status();
$response->successful();      // 2xx
$response->failed();
$response->clientError();     // 4xx
$response->serverError();     // 5xx
$response->header('X-Rate-Limit');
```

Sozlamalar zanjiri:

```php
Http::withToken($token)
    ->withHeaders(['X-Client' => 'mobile'])
    ->acceptJson()
    ->timeout(10)
    ->connectTimeout(3)
    ->retry(3, 100, throw: false)        // 3 marta, 100ms oraliq bilan
    ->baseUrl('https://api.example.uz')
    ->post('/orders', $payload);
```

Xatolarda istisno tashlash:

```php
$response = Http::get($url)->throw();          // 4xx/5xx bo'lsa RequestException
$response = Http::get($url)->throwIf($condition);
```

Parallel so'rovlar:

```php
$responses = Http::pool(fn ($pool) => [
    $pool->get('https://api.uz/a'),
    $pool->get('https://api.uz/b'),
]);
```

Barcha so'rovlar uchun umumiy sozlama:

```php
// AppServiceProvider::boot()
Http::globalOptions(['timeout' => 15]);

Http::macro('eskiz', fn () => Http::baseUrl(config('services.eskiz.url'))
    ->withToken(config('services.eskiz.token')));
```

```php
Http::eskiz()->post('/message/sms/send', [...]);
```

### Testlarda

```php
Http::fake([
    'api.example.uz/*' => Http::response(['ok' => true], 200),
    '*' => Http::response('', 404),
]);

// ...

Http::assertSent(fn ($request) => $request->url() === 'https://api.example.uz/posts');
Http::assertNothingSent();
```

`Http::preventStrayRequests()` — testda soxtalashtirilmagan haqiqiy so'rov ketsa, xato beradi. Buni testlarda doim yoqib qo'yish tavsiya etiladi.

---

## Amaliyot

1. `Cache::remember()` bilan statistikani 1 daqiqaga keshlang va ikkinchi so'rovda SQL ketmaganini `DB::listen()` orqali tekshiring.
2. `Cache::lock()` bilan bitta buyruqni ikki terminalda bir vaqtda ishga tushirib ko'ring.
3. Avatar yuklash endpointini yozing (`store('avatars', 'public')`) va `Storage::url()` bilan havolani qaytaring.
4. `Http::fake()` bilan tashqi API'ni soxtalashtirib, servis klassingizni sinang.
5. `Http::retry(3, 100)` ni sinab ko'ring: soxta javobda 500 qaytarib, nechta urinish bo'lganini tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/cache>
- <https://laravel.com/docs/13.x/filesystem>
- <https://laravel.com/docs/13.x/http-client>

---

[← Oldingi: Hodisalar](26-hodisalar-va-observerlar.md) · [Mundarija](README.md) · [Keyingi: Mail va bildirishnomalar →](28-mail-va-bildirishnoma.md)
