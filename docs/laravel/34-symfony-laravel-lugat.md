# 34 — Symfony → Laravel lug'ati

[← Oldingi: Keyingi qadamlar](33-keyingi-qadamlar.md) · [Mundarija](README.md)

---

## Tushunchalar

| Symfony | Laravel | Bob |
| --- | --- | --- |
| `bin/console` | `php artisan` | [24](24-artisan-va-konsol.md) |
| `src/` | `app/` | [03](03-papkalar-tuzilmasi.md) |
| `templates/` | `resources/views/` | [30](30-blade-va-frontend.md) |
| `var/` | `storage/` | [03](03-papkalar-tuzilmasi.md) |
| `config/packages/*.yaml` | `config/*.php` | [08](08-konfiguratsiya-va-muhit.md) |
| `config/services.yaml` | `AppServiceProvider::register()` | [06](06-service-provider.md) |
| `config/bundles.php` | `bootstrap/providers.php` + auto-discovery | [06](06-service-provider.md) |
| `src/Kernel.php` | `bootstrap/app.php` | [03](03-papkalar-tuzilmasi.md) |
| Bundle | Package + Service Provider | [06](06-service-provider.md) |
| Autowiring | Zero-config resolution | [05](05-service-container.md) |
| `#[Route]` atributi | `routes/web.php`, `routes/api.php` | [09](09-marshrutlash.md) |
| `ParamConverter` / `MapEntity` | Route model binding | [09](09-marshrutlash.md) |
| `kernel.request` listener | Middleware | [10](10-middleware.md) |
| Form + Validator | Form Request | [13](13-validatsiya.md) |
| Doctrine Entity + Repository | Eloquent Model | [16](16-eloquent-asoslari.md) |
| Doctrine Migrations | Laravel Migrations | [15](15-migratsiyalar.md) |
| DQL / QueryBuilder | Eloquent / Query Builder | [14](14-malumotlar-bazasi.md) |
| Fixtures / Foundry | Factory + Seeder | [18](18-factory-va-seeder.md) |
| Serializer / API Platform | API Resource | [20](20-api-resurslar.md) |
| Firewall | Guard | [21](21-autentifikatsiya.md) |
| User Provider | Auth provider | [21](21-autentifikatsiya.md) |
| Voter | Policy | [22](22-avtorizatsiya.md) |
| `isGranted('ROLE_X')` | `Gate::allows('x')` / `$user->can(...)` | [22](22-avtorizatsiya.md) |
| Messenger | Queue + Job | [25](25-navbatlar.md) |
| EventDispatcher | Event + Listener | [26](26-hodisalar-va-observerlar.md) |
| Doctrine lifecycle callbacks | Model observers | [26](26-hodisalar-va-observerlar.md) |
| Cache component | `Cache` fasadi | [27](27-kesh-va-fayllar.md) |
| Flysystem/VichUploader | `Storage` fasadi | [27](27-kesh-va-fayllar.md) |
| HttpClient | `Http` fasadi | [27](27-kesh-va-fayllar.md) |
| Mailer + Notifier | Mailable + Notification | [28](28-mail-va-bildirishnoma.md) |
| Twig | Blade | [30](30-blade-va-frontend.md) |
| Webpack Encore | Vite | [30](30-blade-va-frontend.md) |
| `#[AsCommand]` | `make:command` + `$signature` | [24](24-artisan-va-konsol.md) |
| Profiler / Debug toolbar | Telescope / Debugbar | [33](33-keyingi-qadamlar.md) |
| `dump()` / VarDumper | `dump()` / `dd()` (bir xil komponent) | [23](23-xatoliklar-va-loglar.md) |

---

## Buyruqlar

| Symfony | Laravel |
| --- | --- |
| `symfony serve` | `php artisan serve` / `php artisan dev` |
| `bin/console make:controller` | `php artisan make:controller` |
| `bin/console make:entity` | `php artisan make:model -m` |
| `bin/console doctrine:migrations:migrate` | `php artisan migrate` |
| `bin/console doctrine:fixtures:load` | `php artisan db:seed` |
| `bin/console debug:router` | `php artisan route:list` |
| `bin/console debug:container` | `php artisan about` (to'liq analogi yo'q) |
| `bin/console debug:event-dispatcher` | `php artisan event:list` |
| `bin/console cache:clear` | `php artisan optimize:clear` |
| `bin/console messenger:consume` | `php artisan queue:work` |
| `php bin/phpunit` | `php artisan test` |

---

## Kod naqshlari yonma-yon

### Kontroller

```php
// Symfony
#[Route('/posts/{id}', methods: ['GET'])]
public function show(Post $post): JsonResponse
{
    return $this->json($post);
}
```

```php
// Laravel — routes/api.php
Route::get('/posts/{post}', [PostController::class, 'show']);

// PostController
public function show(Post $post)
{
    return PostResource::make($post);
}
```

### Ma'lumot saqlash

```php
// Symfony
$post = new Post();
$post->setTitle('Salom');
$em->persist($post);
$em->flush();
```

```php
// Laravel
$post = Post::create(['title' => 'Salom']);
```

### So'rov

```php
// Symfony
$posts = $repo->createQueryBuilder('p')
    ->where('p.status = :s')->setParameter('s', 'published')
    ->orderBy('p.createdAt', 'DESC')
    ->setMaxResults(10)
    ->getQuery()->getResult();
```

```php
// Laravel
$posts = Post::where('status', 'published')->latest()->take(10)->get();
```

### Servis bog'lash

```yaml
# Symfony: config/services.yaml
App\Contracts\SmsSender: '@App\Services\EskizSmsSender'
```

```php
// Laravel: AppServiceProvider::register()
$this->app->bind(SmsSender::class, EskizSmsSender::class);
```

### Ruxsat tekshirish

```php
// Symfony
$this->denyAccessUnlessGranted('EDIT', $post);
```

```php
// Laravel
$this->authorize('update', $post);
// yoki: #[Authorize('update', 'post')]
```

### Fon ishi

```php
// Symfony
$bus->dispatch(new SendWelcomeEmail($user->getId()));
```

```php
// Laravel
SendWelcomeEmail::dispatch($user);
```

---

## Nomlash konvensiyalari

| Narsa | Laravel konvensiyasi | Misol |
| --- | --- | --- |
| Model | Birlikda, StudlyCase | `Post`, `OrderItem` |
| Jadval | Ko'plikda, snake_case | `posts`, `order_items` |
| Pivot jadval | Ikki model birlikda, alifbo tartibida | `post_tag` |
| Tashqi kalit | `{model}_id` | `user_id` |
| Kontroller | `{Model}Controller` | `PostController` |
| Form Request | `{Amal}{Model}Request` | `StorePostRequest` |
| Resource | `{Model}Resource` | `PostResource` |
| Policy | `{Model}Policy` | `PostPolicy` |
| Observer | `{Model}Observer` | `PostObserver` |
| Job | Fe'l + ot | `ProcessPodcast` |
| Event | O'tgan zamon | `OrderPlaced` |
| Listener | Fe'l bilan | `SendOrderConfirmation` |
| Route nomi | `{resurs}.{amal}` | `posts.show` |
| Migratsiya | `create_x_table`, `add_y_to_x_table` | `add_status_to_posts_table` |

---

[← Oldingi: Keyingi qadamlar](33-keyingi-qadamlar.md) · [Mundarija](README.md)
