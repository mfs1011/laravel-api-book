# 21 — Autentifikatsiya

[← Oldingi: API Resurslar](20-api-resurslar.md) · [Mundarija](README.md) · [Keyingi: Avtorizatsiya →](22-avtorizatsiya.md)

---

## Tushunchalar: guard va provider

`config/auth.php` da ikki narsa ta'riflanadi:

- **Provider** — foydalanuvchini **qayerdan** olish (`eloquent` + `User` modeli, yoki `database`).
- **Guard** — foydalanuvchini **qanday** aniqlash (`session` cookie orqali, `sanctum` token orqali).

```php
'guards' => [
    'web' => ['driver' => 'session', 'provider' => 'users'],
],

'providers' => [
    'users' => ['driver' => 'eloquent', 'model' => App\Models\User::class],
],
```

> **Symfony bilan solishtirish:** guard ≈ firewall, provider ≈ user provider. Tushunchalar deyarli bir xil.

---

## Qaysi yo'lni tanlash

| Holat | Yechim |
| --- | --- |
| Monolit sayt (Blade sahifalar) | Sessiya (`web` guard) + starter kit |
| Mobil ilova yoki tashqi mijozlar uchun API | **Sanctum API tokenlari** |
| Alohida domendagi SPA (Vue/React) | **Sanctum SPA rejimi** (cookie) |
| To'liq OAuth2 server kerak | **Passport** |
| Auth backend kerak, UI o'zimniki | **Fortify** |

Ko'p hollarda javob — **Sanctum**.

---

## Sanctum o'rnatish

```shell
php artisan install:api
```

Bu buyruq:

1. `laravel/sanctum` paketini o'rnatadi;
2. `personal_access_tokens` jadvali uchun migratsiya qo'shadi;
3. `routes/api.php` yaratadi va `bootstrap/app.php` ga ulaydi.

Keyin:

```shell
php artisan migrate
```

Modelga trait qo'shing:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

---

## Token bilan API autentifikatsiyasi

### Ro'yxatdan o'tish va kirish endpointlari

```php
// routes/api.php
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login'])->middleware('throttle:login');

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/me', fn (Request $request) => $request->user());
    Route::post('/logout', [AuthController::class, 'logout']);
});
```

```php
namespace App\Http\Controllers\Api;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function register(Request $request)
    {
        $data = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'confirmed', Password::defaults()],
        ]);

        $user = User::create($data);   // 'password' => 'hashed' cast avtomatik hash qiladi

        return response()->json([
            'token' => $user->createToken('api')->plainTextToken,
            'user' => UserResource::make($user),
        ], 201);
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
            'device_name' => ['required', 'string'],
        ]);

        $user = User::where('email', $credentials['email'])->first();

        if (! $user || ! Hash::check($credentials['password'], $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['Kiritilgan ma\'lumotlar mos kelmadi.'],
            ]);
        }

        return response()->json([
            'token' => $user->createToken($credentials['device_name'])->plainTextToken,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->noContent();
    }
}
```

### Mijoz tomonidan ishlatish

```shell
curl -H "Authorization: Bearer 1|AbCdEf..." -H "Accept: application/json" \
     http://127.0.0.1:8000/api/me
```

> **Nega `plainTextToken` faqat bir marta ko'rsatiladi?** Bazada tokenning **hash**i saqlanadi. Bu parol bilan bir xil mantiq: baza o'g'irlansa ham tokenlar ishlatib bo'lmaydigan holatda bo'ladi. Shuning uchun token yo'qolsa — yangisini yaratasiz, eskisini tiklab bo'lmaydi.

---

## Token qobiliyatlari (abilities)

```php
$token = $user->createToken('mobil-ilova', ['post:read', 'post:create']);
```

Tekshirish:

```php
if ($request->user()->tokenCan('post:create')) {
    // ...
}
```

Route darajasida:

```php
Route::post('/posts', ...)->middleware(['auth:sanctum', 'ability:post:create']);
Route::get('/posts', ...)->middleware(['auth:sanctum', 'abilities:post:read,post:list']);
```

`ability` — kamida bittasi bo'lsa yetarli; `abilities` — hammasi bo'lishi shart.

---

## Tokenlarni boshqarish

```php
$user->tokens;                                   // barcha tokenlar
$user->tokens()->delete();                       // hammasini bekor qilish
$user->tokens()->where('id', $id)->delete();     // bittasini
$request->user()->currentAccessToken();          // joriy token
$token->last_used_at;
```

Muddat qo'yish — `config/sanctum.php`:

```php
'expiration' => 60 * 24 * 30,   // daqiqada: 30 kun. null bo'lsa — cheksiz
```

Muddati o'tganlarni tozalash:

```shell
php artisan sanctum:prune-expired --hours=24
```

`routes/console.php` da rejalashtiring:

```php
Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

---

## SPA rejimi (bir xil domendagi frontend)

Agar frontend (Vue/React) sizning domeningizda bo'lsa, token o'rniga **cookie sessiya** ishlatish xavfsizroq (token JS'da saqlanmaydi):

1. `.env` da: `SANCTUM_STATEFUL_DOMAINS=localhost:5173` va `SESSION_DOMAIN=localhost`
2. Frontend avval `GET /sanctum/csrf-cookie` ni chaqiradi;
3. Keyin `POST /login` qiladi;
4. Keyingi so'rovlar cookie bilan ketadi (`withCredentials: true`).

CORS'da `supports_credentials => true` bo'lishi kerak ([12-bob](12-sorov-va-javob.md)).

> **Nega "token JS'da saqlash xavfli"?** XSS zaifligi bo'lsa, `localStorage` dagi tokenni o'g'irlash oson. `HttpOnly` cookie'ni esa JavaScript o'qiy olmaydi.

---

## Sessiya (web) autentifikatsiyasi

```php
use Illuminate\Support\Facades\Auth;

if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
    $request->session()->regenerate();     // session fixation'dan himoya

    return redirect()->intended('/dashboard');
}

Auth::logout();
$request->session()->invalidate();
$request->session()->regenerateToken();
```

Foydalanuvchini olish:

```php
Auth::user();  auth()->user();  $request->user();
Auth::id();    auth()->check(); auth()->guest();
```

---

## Parollar

```php
$hash = Hash::make($plain);        // bcrypt (config/hashing.php)
Hash::check($plain, $hash);
Hash::needsRehash($hash);
```

`User` modelida `'password' => 'hashed'` cast bo'lgani uchun `User::create(['password' => 'secret'])` yozsangiz ham parol avtomatik hash'lanadi — qo'lda `Hash::make()` qilish shart emas.

Parolni tiklash (`password_reset_tokens` jadvali skeletonda bor):

```php
use Illuminate\Support\Facades\Password;

Password::sendResetLink($request->only('email'));
Password::reset($credentials, fn ($user, $password) => $user->forceFill([
    'password' => $password,
])->save());
```

---

## Starter kitlar

To'liq tayyor auth (ro'yxatdan o'tish, profil, 2FA) kerak bo'lsa:

```shell
laravel new mening-loyiham   # va starter kit tanlang (React / Vue / Livewire)
```

O'rganish bosqichida esa qo'lda yozib ko'rish foydaliroq — nima qayerda ishlayotgani ko'rinadi.

---

## Amaliyot

1. `php artisan install:api && php artisan migrate` ni bajaring.
2. `User` modeliga `HasApiTokens` qo'shing.
3. `register`, `login`, `me`, `logout` endpointlarini yozing.
4. `curl` bilan token oling va `/api/me` ni token bilan va tokensiz chaqirib, 200 va 401 javoblarini solishtiring.
5. Tokenga `post:create` qobiliyatini bering va `ability:post:create` middleware'ini sinang.
6. `config/sanctum.php` da `expiration` ni 1 daqiqaga qo'yib, muddat o'tgach 401 kelishini tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/authentication>
- <https://laravel.com/docs/13.x/sanctum>
- <https://laravel.com/docs/13.x/passwords> — parolni tiklash
- <https://laravel.com/docs/13.x/hashing>

---

[← Oldingi: API Resurslar](20-api-resurslar.md) · [Mundarija](README.md) · [Keyingi: Avtorizatsiya →](22-avtorizatsiya.md)
