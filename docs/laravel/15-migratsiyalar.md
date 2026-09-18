# 15 — Migratsiyalar

[← Oldingi: Ma'lumotlar bazasi](14-malumotlar-bazasi.md) · [Mundarija](README.md) · [Keyingi: Eloquent asoslari →](16-eloquent-asoslari.md)

---

## Migratsiya nima

Migratsiya — bazaning tuzilishini **kodda** saqlash usuli. Har bir o'zgarish (yangi jadval, yangi ustun) alohida faylga yoziladi va git orqali jamoaga tarqaladi.

> **Symfony bilan solishtirish:** Doctrine Migrations bilan bir xil g'oya. Farqi: Doctrine migratsiyani entity'lardan **avtomatik generatsiya** qiladi (`doctrine:migrations:diff`), Laravel'da esa migratsiyani **siz yozasiz**, model esa alohida turadi. Ya'ni Laravel'da "schema manbai" — migratsiyalar, modellar emas.

---

## Yaratish va ishga tushirish

```shell
php artisan make:migration create_posts_table
php artisan make:migration add_status_to_posts_table
```

Nom konvensiyasi muhim: `create_X_table` bo'lsa Laravel `Schema::create` shabloni bilan, `add_..._to_X_table` bo'lsa `Schema::table` shabloni bilan yaratadi.

```shell
php artisan migrate               # bajarilmagan migratsiyalarni ishga tushirish
php artisan migrate --pretend     # faqat SQL'ni ko'rsatish, bajarmaslik
php artisan migrate:status
php artisan migrate:rollback      # oxirgi "batch"ni qaytarish
php artisan migrate:rollback --step=1
php artisan migrate:reset         # hammasini qaytarish
php artisan migrate:refresh       # reset + migrate
php artisan migrate:fresh         # BARCHA jadvallarni tashlab, qaytadan
php artisan migrate:fresh --seed
```

> **Diqqat:** `migrate:fresh` — barcha ma'lumotni o'chiradi. Productionda hech qachon ishlatmang. Productionda faqat `php artisan migrate --force`.

---

## Migratsiya tuzilmasi

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();                                        // bigint unsigned, primary, auto
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->string('slug')->unique();
            $table->text('body');
            $table->string('status')->default('draft')->index();
            $table->unsignedInteger('views')->default(0);
            $table->timestamp('published_at')->nullable();
            $table->timestamps();                                 // created_at, updated_at
            $table->softDeletes();                                // deleted_at (ixtiyoriy)
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

`down()` — `up()` ning teskarisi. Uni to'g'ri yozish `rollback` ishlashi uchun zarur.

---

## Ustun tiplari (eng kerakli)

```php
$table->id();                          // bigIncrements('id')
$table->uuid('id')->primary();
$table->ulid('id')->primary();

$table->string('name', 100);
$table->text('body');
$table->longText('content');

$table->integer('count');
$table->unsignedBigInteger('user_id');
$table->decimal('price', total: 10, places: 2);   // pul uchun: float EMAS!
$table->boolean('is_active');

$table->date('birthday');
$table->dateTime('starts_at');
$table->timestamp('published_at')->nullable();
$table->timestamps();
$table->softDeletes();

$table->json('meta');
$table->enum('status', ['draft', 'published']);   // ko'chirish qiyin: string + check afzal
$table->binary('file');
$table->foreignId('category_id');
$table->morphs('commentable');                    // commentable_id + commentable_type
$table->nullableMorphs('imageable');
```

> **Nega pul uchun `decimal`?** `float`/`double` — ikkilik kasr, `0.1 + 0.2 !== 0.3`. Pulda bu xato to'planadi. `decimal(10,2)` aniq saqlaydi (yoki butun songa — tiyinda saqlash).

Modifikatorlar:

```php
->nullable()
->default('draft')
->unsigned()
->index()
->unique()
->comment('Izoh')
->after('title')          // MySQL
->useCurrent()            // timestamp uchun
```

---

## Tashqi kalitlar

Qisqa yozuv (konvensiyaga tayanadi):

```php
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
```

Bu `users` jadvalining `id` ustuniga bog'laydi va foydalanuvchi o'chirilsa postlari ham o'chadi.

To'liq yozuv:

```php
$table->foreignId('author_id')
    ->constrained(table: 'users', column: 'id')
    ->cascadeOnUpdate()
    ->restrictOnDelete();
```

Variantlar: `cascadeOnDelete()`, `restrictOnDelete()`, `nullOnDelete()`, `noActionOnDelete()`.

> **SQLite eslatmasi:** SQLite'da tashqi kalitlar standart ravishda yoqilgan (Laravel yoqadi), lekin ustunni o'zgartirish (`change()`) cheklangan. Test va o'rganish uchun muammo emas.

---

## Mavjud jadvalni o'zgartirish

```php
public function up(): void
{
    Schema::table('posts', function (Blueprint $table) {
        $table->string('subtitle')->nullable()->after('title');
        $table->index(['status', 'published_at']);
    });
}

public function down(): void
{
    Schema::table('posts', function (Blueprint $table) {
        $table->dropIndex(['status', 'published_at']);
        $table->dropColumn('subtitle');
    });
}
```

Ustunni o'zgartirish:

```php
$table->string('title', 500)->nullable()->change();
$table->renameColumn('body', 'content');
```

> **Ogohlantirish:** `change()` da ustunning **barcha** modifikatorlarini qayta yozish kerak. Agar avval `->nullable()->default('x')` bo'lgan bo'lsa va siz `change()` da ularni yozmasangiz — ular yo'qoladi.

Indekslar:

```php
$table->index('status');
$table->unique(['user_id', 'slug']);
$table->fullText('body');           // MySQL/PostgreSQL
$table->dropIndex('posts_status_index');
$table->dropUnique(['user_id', 'slug']);
```

---

## Tekshiruvlar va yordamchilar

```php
Schema::hasTable('posts');
Schema::hasColumn('posts', 'slug');
Schema::getColumnListing('posts');
Schema::rename('posts', 'articles');
Schema::dropIfExists('posts');
Schema::disableForeignKeyConstraints();
```

Bazaga bog'liq shart:

```php
if (DB::getDriverName() === 'mysql') {
    // faqat MySQL uchun
}
```

---

## Boshlang'ich migratsiyalar (bu loyihada)

```
0001_01_01_000000_create_users_table.php    → users, password_reset_tokens, sessions
0001_01_01_000001_create_cache_table.php    → cache, cache_locks
0001_01_01_000002_create_jobs_table.php     → jobs, job_batches, failed_jobs
```

Ya'ni sessiya, kesh va navbat uchun jadvallar standart ravishda bor — chunki `.env` da ular `database` drayveriga sozlangan.

---

## Migratsiyalarni "siqish" (squash)

Loyihada 200 ta migratsiya yig'ilganda:

```shell
php artisan schema:dump                 # database/schema/mysql-schema.sql yaratadi
php artisan schema:dump --prune         # eski migratsiya fayllarini o'chiradi
```

Yangi muhitda `migrate` avval shu dump'ni yuklaydi, keyin qolgan migratsiyalarni bajaradi — bu testlarni ham tezlashtiradi.

---

## Yaxshi amaliyotlar

1. **Bajarilgan migratsiyani tahrirlamang.** Jamoadagi boshqa odamda u allaqachon ishlab bo'lgan. Yangi migratsiya yozing.
2. Bitta migratsiya — bitta mantiqiy o'zgarish.
3. Ma'lumot ko'chirish (data migration) kerak bo'lsa, uni alohida migratsiyada yoki alohida Artisan buyrug'ida qiling.
4. Tez-tez qidiriladigan ustunlarga **indeks** qo'ying (`where`, `join`, `order by` da qatnashadiganlar).
5. Modelni migratsiya bilan birga yarating:

```shell
php artisan make:model Post -mfsc
# -m migration, -f factory, -s seeder, -c controller
```

---

## Amaliyot

1. `php artisan make:model Post -mf` bilan model + migratsiya + factory yarating.
2. Migratsiyada `user_id`, `title`, `slug` (unique), `body`, `status`, `published_at` ustunlarini ta'riflang.
3. `php artisan migrate` ni bajaring, keyin `php artisan db:table posts` bilan natijani ko'ring.
4. `php artisan migrate:rollback` qiling va jadval yo'qolganini tekshiring, so'ng yana `migrate`.
5. Yangi migratsiya bilan `views` ustunini qo'shing va `migrate --pretend` bilan qanday SQL bajarilishini ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/migrations>
- <https://laravel.com/docs/13.x/migrations#available-column-types>

---

[← Oldingi: Ma'lumotlar bazasi](14-malumotlar-bazasi.md) · [Mundarija](README.md) · [Keyingi: Eloquent asoslari →](16-eloquent-asoslari.md)
