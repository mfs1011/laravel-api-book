# Brief: Symfony qo'llanmasini yozdirish (boshqa mashinada davom ettirish uchun)

Bu fayl — **topshiriq spetsifikatsiyasi**. Boshqa kompyuterda repo'ni klon qilib, quyidagi promptni Claude Code'ga bering. Hech qanday oldingi suhbat konteksti kerak emas.

## Boshlash (yangi mashinada)

```shell
git clone git@github.com:mfs1011/laravel-api-book.git
cd laravel-api-book
git switch docs/symfony-brief
claude
```

Keyin quyidagi "PROMPT" bo'limini to'liq nusxalab yuboring.

> Eslatma: bu repo Laravel loyihasi. Symfony qo'llanmasi uchun alohida Symfony loyihasi bo'lsa yaxshi (`symfony new demo --webapp`) — shunda versiyalar va kod haqiqiy muhitda tekshiriladi. Bo'lmasa ham qo'llanma yoziladi, faqat "tekshirildi" deb yozilmaydi.

---

## PROMPT (shuni nusxalang)

```
Menga Symfony bo'yicha to'liq qo'llanma kerak, o'zbek tilida (lotin), noldan boshlab, to'g'ri
o'rganish ketma-ketligida. Men Laravel'ni yaxshi bilaman, shuning uchun har bobda
"Laravel bilan solishtirish" bo'limi bo'lsin.

Talablar:

1. Fayllar: docs/symfony/ papkasida, alohida md fayllar, NN-nom.md ko'rinishida (01-kirish.md ...).
   README.md — mundarija, barcha boblarga havola bilan.
2. Har bir bob oxirida navigatsiya: [← Oldingi](..) · [Mundarija](README.md) · [Keyingi →](..)
   Boblar orasida ichki havolalar ko'p bo'lsin ([12-bob](12-...md) kabi).
3. Ma'lumot RASMIY hujjatdan olinsin: https://symfony.com/doc/current/...
   Har bob oxirida "Rasmiy hujjat" havolalari bo'lsin.
4. Sodda tushuntir, lekin "nega bunday?" degan savol qolmasin — har bir qoidaning sababini yoz.
5. Chala bo'lmasin: har bobda ishlaydigan kod misollari + "Amaliyot" (mashq) bo'limi.
6. Versiyalarni taxmin qilma. Avval `composer show --direct`, `php -v`, `symfony console about`
   bilan aniqla va qo'llanmani aynan shu versiyaga moslab yoz. Versiyani README'da jadval qilib ko'rsat.
7. Yozib bo'lgach TEKSHIR: (a) uzilgan ichki havola yo'qligini skript bilan tekshir,
   (b) to'liq PHP misollarini `php -l` bilan sintaksis xatosiga tekshir, (c) natijani hisobot qilib ayt.
8. Har bir bob tuzilmasi: tushuncha → nega shunday → kod → Laravel bilan solishtirish (jadval)
   → tipik xatolar → Amaliyot → Rasmiy hujjat → navigatsiya.

Boblar ketma-ketligi (shu tartibda, kerak bo'lsa moslashtir):

I. Poydevor
 01 Kirish: Symfony nima, falsafasi, versiyalar (LTS), Laravel'dan farqi
 02 O'rnatish: symfony CLI, Flex, `symfony new`, `symfony server:start`, .env va .env.local
 03 Papkalar tuzilmasi: src/, config/, templates/, var/, public/, bin/console
 04 So'rovning hayot sikli: HttpKernel, kernel.* eventlari, front controller
 05 Service container va autowiring: services.yaml, avtokonfiguratsiya, binding, tag'lar
 06 Bundlelar va Flex retseptlari
 07 Konfiguratsiya: YAML, parametrlar, muhitlar, secrets vault (`secrets:set`)

II. HTTP qatlami
 08 Routing: #[Route] atributlari, parametrlar, requirements, nomlangan route, prefiks
 09 Controller: AbstractController, argument resolver, #[MapEntity], #[MapRequestPayload]
 10 Request/Response: HttpFoundation, JsonResponse, status kodlari, fayllar
 11 Event listener/subscriber va kernel eventlari (Laravel middleware'ining muqobili)
 12 Validatsiya: Constraint atributlari, ValidatorInterface, #[MapRequestPayload] bilan
 13 Formalar (qisqacha — API uchun majburiy emas)

III. Ma'lumotlar bazasi
 14 Doctrine ORM: entity, Data Mapper, EntityManager, persist/flush
 15 Migratsiyalar: doctrine:migrations:diff/migrate
 16 Repository va DQL, QueryBuilder, indekslar
 17 Aloqalar: OneToMany, ManyToMany, N+1 va JOIN FETCH, EXTRA_LAZY
 18 Fixtures va test ma'lumotlari (DoctrineFixturesBundle, zenstruck/foundry)

IV. API qurish
 19 Serializer: guruhlar, normalizer, DTO
 20 API Platform (agar ishlatilsa) yoki qo'lda REST API
 21 Xavfsizlik: security.yaml, authenticator, JWT (LexikJWTAuthenticationBundle), Passport
 22 Avtorizatsiya: rollar, Voter'lar, #[IsGranted]
 23 Xatoliklar va Monolog

V. Fon ishlari va xizmatlar
 24 Console buyruqlari (#[AsCommand]), Scheduler
 25 Messenger: transport, handler, retry, failure transport
 26 EventDispatcher va Doctrine lifecycle events
 27 Cache, Filesystem, HttpClient
 28 Mailer va Notifier

VI. Sifat va yakun
 29 Testlash: PHPUnit, WebTestCase, KernelTestCase, DAMA/doctrine-test-bundle
 30 Twig va AssetMapper/Encore
 31 Amaliy loyiha: noldan to'liq REST API (entity → migratsiya → voter → DTO → controller → testlar)
 32 Unumdorlik va deploy: preload, opcache, cache:warmup, Docker
 33 Ekotizim va keyingi qadamlar
 34 Laravel → Symfony lug'ati (jadval: Eloquent→Doctrine, Policy→Voter, Queue→Messenger, ...)

VII. Production darajasi
 35 Real-time: Mercure
 36 Ko'p tillilik: Translation komponenti, locale, Intl
 37 Ilg'or Doctrine va unumdorlik: custom type, JSON, qulflar, batch processing, ikkinchi daraja kesh
 38 API pro: idempotentlik, ETag, webhook, OpenAPI (NelmioApiDocBundle), versiyalash
 39 Arxitektura: qatlamlar, DTO, CQRS-lite, bundle yozish
 40 CI/CD, Docker va production operatsiyalari + yakuniy tekshiruv ro'yxati

Git qoidalari: main'ga to'g'ridan-to'g'ri push qilma, alohida branch och, commit xabarida
AI attribution bo'lmasin, PR'ni men aytmagunimcha ochma.
```

---

## Bu qo'llanma (Laravel versiyasi) qanday yozilgan — namuna sifatida

`docs/laravel/` da 41 fayl bor. Symfony versiyasi ham xuddi shu qolipda bo'lishi kerak:

- Har bob: tushuncha → "nega bunday?" → kod → solishtirish jadvali → tipik xatolar → Amaliyot → rasmiy havola → navigatsiya
- Uzunlik: bob boshiga ~700–1000 so'z + kod
- Kod misollari haqiqiy muhitda tekshirilgan (`php -l`, tinker/console)
- Rasmiy hujjatga aniq havolalar, versiya raqami bilan

Sifat mezoni: "o'qib chiqqan odam production loyiha qila oladimi?" — 40-bobdagi yakuniy tekshiruv ro'yxati shu savolga javob beradi.

## Eslatmalar

- Bu repo PUBLIC. Session loglari (`~/.claude/projects/.../*.jsonl`) va shaxsiy `~/.claude/CLAUDE.md` bu yerga qo'shilmasin.
- Shaxsiy global qoidalar (branch/PR/attribution) boshqa mashinada yo'q bo'ladi — shuning uchun ular promptning oxirgi qatorida takrorlangan.
