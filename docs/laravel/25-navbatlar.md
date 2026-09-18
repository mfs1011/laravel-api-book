# 25 — Navbatlar (Queues va Jobs)

[← Oldingi: Artisan va konsol](24-artisan-va-konsol.md) · [Mundarija](README.md) · [Keyingi: Hodisalar va observerlar →](26-hodisalar-va-observerlar.md)

---

## Nima uchun kerak

Foydalanuvchi so'rovi ichida pochta yuborish, rasm siqish yoki tashqi API'ga murojaat qilish — javobni sekinlashtiradi. Bunday ishlarni **navbatga** qo'yasiz: so'rov darhol javob qaytaradi, ish esa fonda bajariladi.

> **Symfony bilan solishtirish:** Messenger komponentining aynan analogi. Job ≈ Message + Handler bir klassda, worker ≈ `messenger:consume`.

---

## Drayverlar

`.env` dagi `QUEUE_CONNECTION`:

| Drayver | Qachon |
| --- | --- |
| `sync` | Darhol bajaradi (lokal test uchun; navbat yo'q) |
| `database` | Kichik/o'rta loyihalar (bu loyihada standart) |
| `redis` | Yuqori yuklama, Horizon bilan |
| `sqs`, `beanstalkd` | Bulutli yechimlar |

`database` uchun jadvallar allaqachon bor (`jobs`, `job_batches`, `failed_jobs`). Yo'q bo'lsa:

```shell
php artisan make:queue-table && php artisan migrate
```

---

## Job yaratish

```shell
php artisan make:job ProcessPodcast
```

```php
namespace App\Jobs;

use App\Models\Podcast;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;               // necha marta urinish
    public int $timeout = 120;           // soniya
    public int $backoff = 30;            // urinishlar orasidagi kutish
    public bool $afterCommit = true;     // tranzaksiya commit'idan keyin yuborilsin

    public function __construct(public Podcast $podcast) {}

    public function handle(AudioProcessor $processor): void
    {
        $processor->transcode($this->podcast);
    }

    public function failed(\Throwable $e): void
    {
        $this->podcast->update(['status' => 'failed']);
    }
}
```

Konstruktorga uzatilgan model **serializatsiya** qilinadi: bazaga faqat ID yoziladi, worker uni qaytadan yuklaydi. Shuning uchun:

- Job navbatga tushgandan keyin model o'zgarsa, worker **yangi** holatni oladi.
- Model o'chirilgan bo'lsa, job `ModelNotFoundException` bilan tushadi (yoki `$deleteWhenMissingModels = true` bilan o'chiriladi).

---

## Navbatga qo'yish

```php
ProcessPodcast::dispatch($podcast);
ProcessPodcast::dispatch($podcast)->onQueue('media');
ProcessPodcast::dispatch($podcast)->onConnection('redis');
ProcessPodcast::dispatch($podcast)->delay(now()->addMinutes(10));
ProcessPodcast::dispatchIf($shouldRun, $podcast);
ProcessPodcast::dispatchUnless($skip, $podcast);
ProcessPodcast::dispatchSync($podcast);          // darhol, navbatsiz
ProcessPodcast::dispatchAfterResponse($podcast); // javob yuborilgandan keyin
```

Zanjir (ketma-ket bajarish):

```php
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new ProcessPodcast($podcast),
    new OptimizePodcast($podcast),
    new ReleasePodcast($podcast),
])->dispatch();
```

Guruh (parallel + yakuniy hodisa):

```php
$batch = Bus::batch([
    new ImportCsv(1), new ImportCsv(2), new ImportCsv(3),
])->then(function ($batch) {
    // hammasi muvaffaqiyatli tugadi
})->catch(function ($batch, $e) {
    // birinchi xatolik
})->finally(function ($batch) {
    // har holatda
})->name('CSV import')->dispatch();

$batch->id; $batch->progress(); $batch->cancel();
```

Batch uchun job'ga `use Batchable;` trait'ini qo'shing.

---

## Worker'ni ishga tushirish

```shell
php artisan queue:work
php artisan queue:work --queue=high,default      # muhimlik tartibi
php artisan queue:work --tries=3 --timeout=90
php artisan queue:work --once
php artisan queue:listen                          # sekinroq, lekin kodni qayta yuklaydi
```

> **Muhim:** `queue:work` kodni **xotiraga yuklab** ishlaydi. Kodni o'zgartirsangiz, worker eski kodni bajaraveradi. Deploy'dan keyin har doim:

```shell
php artisan queue:restart
```

Lokal ishlab chiqishda `php artisan dev` worker'ni o'zi ko'taradi.

Boshqa foydali buyruqlar:

```shell
php artisan queue:pause          # yangi joblarni olishni to'xtatish
php artisan queue:resume
php artisan queue:monitor database:default --max=100
```

### Productionda

Supervisor (an'anaviy server) yoki konteyner jarayoni sifatida:

```ini
[program:laravel-worker]
command=php /var/www/loyiha/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
numprocs=4
```

`--max-time` yoki `--max-jobs` — worker vaqti-vaqti bilan qayta ishga tushsin (xotira sizib chiqishiga qarshi).

---

## Xatoliklar va qayta urinish

```php
public int $tries = 5;
public array $backoff = [10, 30, 60];           // progressiv kutish

public function retryUntil(): \DateTime
{
    return now()->addMinutes(30);               // vaqt chegarasi
}
```

Job ichida:

```php
$this->release(30);        // navbatga qaytarish, 30 soniyadan keyin
$this->fail($exception);   // darhol muvaffaqiyatsiz deb belgilash
$this->delete();
$this->attempts();
if ($this->batch()?->cancelled()) { return; }
```

Muvaffaqiyatsiz joblar `failed_jobs` jadvaliga tushadi:

```shell
php artisan queue:failed
php artisan queue:retry 5
php artisan queue:retry all
php artisan queue:forget 5
php artisan queue:flush
php artisan queue:prune-failed --hours=168
```

Global xabar berish:

```php
// AppServiceProvider::boot()
use Illuminate\Support\Facades\Queue;
use Illuminate\Queue\Events\JobFailed;

Queue::failing(function (JobFailed $event) {
    logger()->critical('Job tushdi', [
        'job' => $event->job->resolveName(),
        'error' => $event->exception->getMessage(),
    ]);
});
```

---

## Job middleware

```php
use Illuminate\Queue\Middleware\{WithoutOverlapping, RateLimited, ThrottlesExceptions, Skip};

public function middleware(): array
{
    return [
        new WithoutOverlapping($this->podcast->id),     // bir xil ID bilan parallel ishlamasin
        new RateLimited('external-api'),
        (new ThrottlesExceptions(10, 5 * 60))->backoff(5),
        Skip::when(fn () => $this->podcast->isArchived()),
    ];
}
```

`WithoutOverlapping` — bitta resurs ustida ikki job bir vaqtda ishlashining oldini oladi (masalan bitta foydalanuvchi balansini ikki job o'zgartirmasin).

---

## Unikallik

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;

class UpdateSearchIndex implements ShouldQueue, ShouldBeUnique
{
    public int $uniqueFor = 3600;

    public function uniqueId(): string
    {
        return $this->post->id;
    }
}
```

Bir xil `uniqueId` bilan ikkinchi job navbatga umuman qo'shilmaydi.

---

## Tranzaksiyalar bilan ehtiyotkorlik

```php
DB::transaction(function () use ($order) {
    $order->save();
    ProcessOrder::dispatch($order);      // ⚠️ commit'dan oldin navbatga tushishi mumkin
});
```

Worker job'ni `commit` sodir bo'lishidan oldin olib qolsa — bazada yozuv topilmaydi. Yechim:

```php
public bool $afterCommit = true;       // job klassida
```

yoki

```php
ProcessOrder::dispatch($order)->afterCommit();
```

yoki `config/queue.php` da ulanish uchun `'after_commit' => true`.

---

## Testlash

```php
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Bus;

Queue::fake();

// ... amalni bajaramiz ...

Queue::assertPushed(ProcessPodcast::class);
Queue::assertPushedOn('media', ProcessPodcast::class);
Queue::assertNotPushed(DeletePodcast::class);
Queue::assertCount(1);

Bus::fake();
Bus::assertChained([ProcessPodcast::class, ReleasePodcast::class]);
Bus::assertBatched(fn ($batch) => $batch->jobs->count() === 3);
```

Testda `QUEUE_CONNECTION=sync` bo'lgani uchun (`phpunit.xml` da) job'lar darhol bajariladi — bu ham qulay.

---

## Horizon (Redis uchun)

```shell
composer require laravel/horizon
php artisan horizon:install
php artisan horizon
```

Beradi: real vaqt paneli, metrikalar, worker'larni avtomatik masshtablash, muvaffaqiyatsiz joblarni ko'rish. Faqat `redis` drayveri bilan ishlaydi.

---

## Amaliyot

1. `php artisan make:job SendWelcomeEmail` yarating, `handle()` da log yozing.
2. Ro'yxatdan o'tish kontrollerida `SendWelcomeEmail::dispatch($user)` chaqiring.
3. `.env` da `QUEUE_CONNECTION=database` ekanini tekshiring, `php artisan queue:work` ni ishga tushiring va log paydo bo'lishini ko'ring.
4. `handle()` ichida ataylab istisno tashlang, `tries = 2` qo'ying va `php artisan queue:failed` ro'yxatida ko'ring, keyin `queue:retry` qiling.
5. Testda `Queue::fake()` bilan `assertPushed` yozing.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/queues>
- <https://laravel.com/docs/13.x/horizon>

---

[← Oldingi: Artisan va konsol](24-artisan-va-konsol.md) · [Mundarija](README.md) · [Keyingi: Hodisalar va observerlar →](26-hodisalar-va-observerlar.md)
