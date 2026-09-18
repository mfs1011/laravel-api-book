# 24 — Artisan va konsol

[← Oldingi: Xatoliklar va loglar](23-xatoliklar-va-loglar.md) · [Mundarija](README.md) · [Keyingi: Navbatlar →](25-navbatlar.md)

---

## Artisan nima

Artisan — Laravel'ning CLI'si; `symfony/console` ustida qurilgan. Ya'ni Symfony Console'dan bilganlaringiz shu yerda ham ishlaydi, ustiga Laravel generatorlari va Prompts qo'shilgan.

```shell
php artisan list                    # barcha buyruqlar
php artisan help migrate            # bitta buyruq haqida
php artisan about                   # ilova holati
```

---

## Eng kerakli buyruqlar

```shell
# Ishga tushirish
php artisan serve
php artisan dev                      # server + vite + queue + loglar birga
php artisan tinker

# Generatorlar
php artisan make:model Post -mfsc
php artisan make:controller PostController --api --model=Post --requests
php artisan make:request StorePostRequest
php artisan make:resource PostResource
php artisan make:policy PostPolicy --model=Post
php artisan make:job ProcessPayment
php artisan make:event OrderShipped
php artisan make:listener SendShipmentNotification --event=OrderShipped
php artisan make:observer PostObserver --model=Post
php artisan make:command SyncProducts
php artisan make:test PostApiTest --pest

# Baza
php artisan migrate
php artisan migrate:status
php artisan migrate:fresh --seed
php artisan db:seed
php artisan db:show

# Ko'rish
php artisan route:list --path=api
php artisan event:list
php artisan schedule:list
php artisan queue:failed

# Kesh
php artisan optimize
php artisan optimize:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

Har bir `make:` buyrug'ining variantlarini `--help` bilan ko'ring.

---

## O'z buyrug'ingizni yozish

```shell
php artisan make:command SyncProducts
```

```php
namespace App\Console\Commands;

use App\Services\ProductSync;
use Illuminate\Console\Command;

class SyncProducts extends Command
{
    /**
     * Buyruq nomi va argumentlari.
     */
    protected $signature = 'products:sync
                            {source : Manba nomi}
                            {--limit=100 : Nechta mahsulot}
                            {--dry-run : Faqat ko\'rsatish, saqlamaslik}';

    protected $description = 'Mahsulotlarni tashqi manbadan sinxronlaydi';

    public function handle(ProductSync $sync): int
    {
        $source = $this->argument('source');
        $limit = (int) $this->option('limit');

        $this->info("Sinxronlash boshlandi: {$source}");

        $products = $sync->fetch($source, $limit);

        $bar = $this->output->createProgressBar($products->count());

        foreach ($products as $product) {
            if (! $this->option('dry-run')) {
                $sync->store($product);
            }
            $bar->advance();
        }

        $bar->finish();
        $this->newLine();

        $this->table(['Manba', 'Soni'], [[$source, $products->count()]]);
        $this->components->info('Tayyor.');

        return self::SUCCESS;      // 0. Xato bo'lsa: self::FAILURE
    }
}
```

```shell
php artisan products:sync eskiz --limit=50 --dry-run
```

`app/Console/Commands/` dagi buyruqlar **avtomatik ro'yxatdan o'tadi** — hech narsa qo'shish shart emas.

### Chiqish (output)

```php
$this->info('Yashil');
$this->comment('Sariq');
$this->error('Qizil');
$this->warn('Ogohlantirish');
$this->line('Oddiy');
$this->newLine();
$this->table(['Ustun'], [['qiymat']]);
$this->components->info('...');
$this->components->twoColumnDetail('Nomi', 'Qiymati');
```

### Kirish (Laravel Prompts)

```php
use function Laravel\Prompts\{text, password, confirm, select, multiselect, search, spin, progress};

$name = text('Ismingiz?', required: true);
$ok = confirm('Davom etamizmi?');
$role = select('Rol tanlang', ['admin', 'editor'], default: 'editor');
$ids = multiselect('Foydalanuvchilar', User::pluck('name', 'id')->all());

$result = spin(fn () => $api->fetch(), 'Yuklanmoqda...');

progress(label: 'Ko\'chirilmoqda', steps: $items, callback: fn ($item) => $item->sync());
```

Argument berilmasa avtomatik so'rash:

```php
use Illuminate\Contracts\Console\PromptsForMissingInput;

class SyncProducts extends Command implements PromptsForMissingInput
{
    protected function promptForMissingArgumentsUsing(): array
    {
        return ['source' => 'Qaysi manbadan sinxronlaymiz?'];
    }
}
```

### Bir vaqtda faqat bitta nusxa ishlashi

```php
use Illuminate\Contracts\Console\Isolatable;

class SyncProducts extends Command implements Isolatable {}
```

```shell
php artisan products:sync eskiz --isolated
```

**Nega kerak?** Cron har 5 daqiqada ishga tushirsa va oldingi nusxa hali tugamagan bo'lsa, ikkitasi bir vaqtda ishlab ma'lumotni buzishi mumkin.

---

## Closure buyruqlari

Kichik ishlar uchun alohida klass shart emas — `routes/console.php`:

```php
use Illuminate\Support\Facades\Artisan;

Artisan::command('users:count', function () {
    $this->info('Foydalanuvchilar: '.\App\Models\User::count());
})->purpose('Foydalanuvchilar sonini ko\'rsatadi');
```

---

## Boshqa joydan chaqirish

```php
use Illuminate\Support\Facades\Artisan;

Artisan::call('products:sync', ['source' => 'eskiz', '--limit' => 50]);
Artisan::queue('products:sync', ['source' => 'eskiz']);     // navbatga
$output = Artisan::output();

// buyruq ichidan boshqa buyruq
$this->call('cache:clear');
$this->callSilently('queue:restart');
```

---

## Scheduler — vazifalarni rejalashtirish

Rejalar `routes/console.php` da ta'riflanadi:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('products:sync eskiz')->hourly();
Schedule::command('sanctum:prune-expired --hours=24')->daily();
Schedule::command('model:prune')->daily();
Schedule::command('queue:prune-failed --hours=168')->weekly();

Schedule::call(fn () => Cache::forget('stats'))->everyFiveMinutes();

Schedule::job(new GenerateDailyReport)->dailyAt('02:00');
```

Chastota metodlari:

```php
->everyMinute() ->everyFiveMinutes() ->everyThirtyMinutes()
->hourly() ->hourlyAt(15)
->daily() ->dailyAt('13:00') ->twiceDaily(1, 13)
->weekly() ->weeklyOn(1, '8:00')          // 1 = dushanba
->monthly() ->quarterly() ->yearly()
->cron('0 */6 * * *')                      // xohlagan cron ifodasi
->timezone('Asia/Tashkent')
```

Qo'shimcha cheklovlar:

```php
Schedule::command('report:send')
    ->dailyAt('09:00')
    ->weekdays()
    ->timezone('Asia/Tashkent')
    ->withoutOverlapping()          // oldingisi tugamaguncha boshlanmasin
    ->onOneServer()                 // bir nechta serverda faqat bittasida
    ->runInBackground()
    ->emailOutputOnFailure('admin@example.com')
    ->when(fn () => config('features.reports'));
```

### Serverda ishga tushirish

Faqat **bitta** cron yozuvi kerak:

```cron
* * * * * cd /var/www/loyiha && php artisan schedule:run >> /dev/null 2>&1
```

Laravel har daqiqada `schedule:run` ni chaqiradi va o'sha daqiqada bajarilishi kerak bo'lgan vazifalarni topadi.

Lokalda sinash:

```shell
php artisan schedule:list
php artisan schedule:test          # bitta vazifani tanlab ishga tushirish
php artisan schedule:work          # lokal "cron" emulyatori
```

Doimiy ishlaydigan jarayon sifatida (Docker, supervisor):

```shell
php artisan schedule:work
```

---

## Tinker

```shell
php artisan tinker
```

```php
User::count();
Post::factory()->create();
config('app.name');
cache()->put('x', 1, 60);
```

Bir qatorda:

```shell
php artisan tinker --execute 'echo App\Models\User::count();'
```

> Qo'shtirnoqqa e'tibor bering: tashqi tomonda **bitta** tirnoq ishlating, aks holda shell `$` belgilarini o'zgaruvchi deb o'qiydi.

---

## Amaliyot

1. `php artisan make:command SyncProducts` yarating, argument va option qo'shing, `handle()` da `$this->table()` bilan natija chiqaring.
2. `--dry-run` bayrog'ini qo'llab-quvvatlang.
3. `Isolatable` interfeysini qo'shing va `--isolated` bilan ikki marta parallel ishga tushirib ko'ring.
4. `routes/console.php` da `Schedule::command('products:sync eskiz')->everyFiveMinutes()->withoutOverlapping();` yozing va `php artisan schedule:list` bilan tekshiring.
5. `php artisan schedule:work` ni ishga tushirib, vazifa bajarilishini kuzating.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/artisan>
- <https://laravel.com/docs/13.x/scheduling>
- <https://laravel.com/docs/13.x/prompts>

---

[← Oldingi: Xatoliklar va loglar](23-xatoliklar-va-loglar.md) · [Mundarija](README.md) · [Keyingi: Navbatlar →](25-navbatlar.md)
