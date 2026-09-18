# 14 — Ma'lumotlar bazasi va Query Builder

[← Oldingi: Validatsiya](13-validatsiya.md) · [Mundarija](README.md) · [Keyingi: Migratsiyalar →](15-migratsiyalar.md)

---

## Ulanishlar

`config/database.php` da bir nechta ulanish ta'riflanadi, `.env` da qaysi biri standart ekani ko'rsatiladi:

```ini
DB_CONNECTION=sqlite         # mysql | pgsql | sqlsrv | sqlite | mariadb
```

Qo'llab-quvvatlanadigan bazalar: MySQL 8+, MariaDB 10+, PostgreSQL 12+, SQLite 3.26+, SQL Server 2019+.

Bir nechta ulanish bilan ishlash:

```php
DB::connection('reporting')->table('orders')->count();
```

Tekshirish:

```shell
php artisan db:show
php artisan db:table users
php artisan db:monitor
php artisan db                 # baza CLI klientini ochadi
```

> **SQLite haqida:** o'rganish va testlar uchun ideal — server kerak emas, fayl. Lekin productionda odatda MySQL/PostgreSQL ishlatiladi.

---

## Query Builder

Eloquent'ning ostida shu turadi. To'g'ridan-to'g'ri ishlatish kerak bo'lgan holatlar: hisobotlar, katta `JOIN` lar, modelga hojat yo'q joylar.

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();                      // Collection<stdClass>
$user = DB::table('users')->find(3);
$email = DB::table('users')->where('id', 3)->value('email');
$names = DB::table('users')->pluck('name', 'id');
```

### Shartlar

```php
DB::table('posts')
    ->where('status', 'published')
    ->where('views', '>', 100)
    ->orWhere(fn ($q) => $q->where('featured', true)->where('views', '>', 50))
    ->whereIn('category_id', [1, 2, 3])
    ->whereNotNull('published_at')
    ->whereBetween('created_at', [$from, $to])
    ->whereDate('created_at', today())
    ->when($search, fn ($q, $search) => $q->where('title', 'like', "%{$search}%"))
    ->orderByDesc('created_at')
    ->limit(10)
    ->get();
```

`when()` — juda foydali: shart bajarilsagina qism qo'shiladi, ya'ni `if` yozish shart emas.

### Agregatlar va guruhlash

```php
DB::table('orders')->count();
DB::table('orders')->sum('total');
DB::table('orders')->avg('total');
DB::table('orders')->max('total');

DB::table('orders')
    ->select('user_id', DB::raw('COUNT(*) as orders_count'), DB::raw('SUM(total) as revenue'))
    ->groupBy('user_id')
    ->having('revenue', '>', 1_000_000)
    ->get();
```

### JOIN

```php
DB::table('posts')
    ->join('users', 'users.id', '=', 'posts.user_id')
    ->leftJoin('categories', 'categories.id', '=', 'posts.category_id')
    ->select('posts.*', 'users.name as author', 'categories.title as category')
    ->get();
```

### Yozish

```php
DB::table('users')->insert(['name' => 'Ali', 'email' => 'ali@example.com']);
$id = DB::table('users')->insertGetId([...]);

DB::table('users')->where('id', 1)->update(['votes' => 10]);
DB::table('users')->where('id', 1)->increment('votes');
DB::table('users')->where('id', 1)->decrement('balance', 500);

DB::table('users')->where('votes', '<', 10)->delete();
DB::table('users')->truncate();

// bor bo'lsa yangilash, yo'q bo'lsa yaratish
DB::table('settings')->upsert(
    [['key' => 'theme', 'value' => 'dark']],
    uniqueBy: ['key'],
    update: ['value'],
);
```

---

## Tranzaksiyalar

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () use ($order) {
    $order->save();
    $order->items()->createMany($items);
    Inventory::decrement($order->product_id, $order->qty);
});
```

Closure ichida istisno chiqsa — hamma narsa avtomatik `rollback` bo'ladi. Deadlock bo'lganda qayta urinish:

```php
DB::transaction(fn () => /* ... */, attempts: 3);
```

Qo'lda boshqarish:

```php
DB::beginTransaction();

try {
    // ...
    DB::commit();
} catch (\Throwable $e) {
    DB::rollBack();
    throw $e;
}
```

Tranzaksiya muvaffaqiyatli yakunlangandan **keyin** biror ish qilish (masalan navbatga job qo'yish):

```php
DB::afterCommit(fn () => ProcessOrder::dispatch($order));
```

> **Nega bu muhim?** Agar job'ni tranzaksiya ichida navbatga qo'ysangiz, worker uni `commit` dan oldin olib qolishi va bazada hali yo'q yozuvni qidirishi mumkin. `afterCommit` shu poyga holatini (race condition) yo'qotadi. Job klasslari uchun `public $afterCommit = true;` ham bor ([25-bob](25-navbatlar.md)).

---

## Katta hajmdagi ma'lumot

```php
// 200 tadan bo'lib o'qish (xotira tejaydi)
DB::table('users')->orderBy('id')->chunk(200, function ($users) {
    foreach ($users as $user) { /* ... */ }
});

// yozuvlarni o'zgartirayotganda — chunkById xavfsizroq
Post::where('status', 'draft')->chunkById(200, fn ($posts) => $posts->each->publish());

// bittalab, generator orqali
foreach (DB::table('users')->lazy() as $user) { /* ... */ }
foreach (Post::cursor() as $post) { /* ... */ }
```

> **Nega `all()` yomon?** 1 million qatorni PHP xotirasiga yuklash — "Allowed memory size exhausted". `chunk`/`lazy`/`cursor` esa xotirani bir xil darajada ushlab turadi.

---

## Xom SQL (raw)

```php
$users = DB::select('select * from users where votes > ?', [100]);
DB::statement('drop table if exists old_logs');

Post::whereRaw('lower(title) like ?', ['%laravel%'])->get();
Post::selectRaw('count(*) as total, status')->groupBy('status')->get();
```

> **Xavfsizlik:** foydalanuvchi kiritgan qiymatni hech qachon SQL satriga yopishtirmang. Doim `?` va parametrlar massivini ishlating — bu SQL injection'dan himoya qiladi. Query Builder buni o'zi qiladi.

---

## So'rovlarni kuzatish (debug)

```php
// AppServiceProvider::boot() — faqat lokalda
DB::listen(function ($query) {
    logger()->debug($query->sql, ['bindings' => $query->bindings, 'time' => $query->time]);
});
```

```php
$query = Post::where('status', 'published');
$query->toSql();        // SQL satri
$query->dd();           // SQL + bindings chiqarib to'xtatadi
$query->dump();
```

Sekin so'rovlar haqida ogohlantirish:

```php
DB::whenQueryingForLongerThan(500, function ($connection) {
    logger()->warning('Sekin so\'rovlar', ['connection' => $connection->getName()]);
});
```

---

## Amaliyot

1. `php artisan db:show` va `php artisan db:table users` ni ishlating.
2. Tinker'da:

```php
DB::table('users')->insert(['name' => 'Test', 'email' => 't@t.uz', 'password' => bcrypt('secret')]);
DB::table('users')->count();
DB::table('users')->where('email', 'like', '%@t.uz')->get();
```

3. Tranzaksiya yozing: ichida istisno tashlab, yozuv saqlanmaganini tekshiring.
4. `DB::listen()` ni yoqing va sahifani ochib, nechta SQL so'rov ketganini loglardan ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/database>
- <https://laravel.com/docs/13.x/queries>

---

[← Oldingi: Validatsiya](13-validatsiya.md) · [Mundarija](README.md) · [Keyingi: Migratsiyalar →](15-migratsiyalar.md)
