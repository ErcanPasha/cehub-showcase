# CE-Hub

Marmara Üniversitesi mühendislik öğrencileri için interaktif ders ve çalışma platformu.

**Canlı:** [cehub.dev](https://cehub.dev) &nbsp;|&nbsp; **Durum:** canlıda, aktif geliştirme &nbsp;|&nbsp; **Kaynak kod:** private

CE-Hub, mühendislik derslerini sadece okumak yerine denemeyi öne alan bir platform. Formülün yanında kaydırıcıyla oynanabilen simülasyonlar, çözümlü örnekler ve alıştırmalar var. Tasarımı, frontend'i, backend'i ve yayın sürecini ben yürütüyorum. Platform 200'den fazla öğrenciye açık, yeni ders ve içerikler eklenmeye devam ediyor.

> Platform giriş gerektirir. Demo hesap için bana ulaşabilirsiniz (iletişim bilgileri en altta).

## Ekran görüntüleri

![İnteraktif ders: Why Steepest Descent?](docs/screenshots/interactive-lesson-steepest-descent.png)

*Makine Öğrenmesi dersinden bir interaktif materyal: 3B hata yüzeyi ve 2B izohips haritası, adım açısı kaydırıcısıyla birlikte canlı güncelleniyor.*

<!-- TODO: Study Area güncellemesi bitince ekle
![Study Area](docs/screenshots/study-area.png)

*Study Area: sayaç, geri sayım, CEP puanı ve istatistikler.*
-->

<!-- TODO: Ders portalı ekran görüntüsü ekle
![Ders portalı](docs/screenshots/course-portal.png)

*Ders ve konu listesi.*
-->

## İçindekiler

- [Öne çıkanlar](#öne-çıkanlar)
- [Genel mimari](#genel-mimari)
- [Ders sistemi](#ders-sistemi)
- [Study Area ve CEP](#study-area-ve-cep)
- [Backend mimarisi](#backend-mimarisi)
- [Kimlik doğrulama](#kimlik-doğrulama)
- [Frontend mimarisi](#frontend-mimarisi)
- [Teknoloji yığını](#teknoloji-yığını)
- [Tasarım kararları](#tasarım-kararları)
- [Yol haritası](#yol-haritası)
- [Kaynak kod ve iletişim](#kaynak-kod-ve-iletişim)

## Öne çıkanlar

- 6 ders, yaklaşık 190 interaktif materyal. Hepsi canlıda, yenileri eklenmeye devam ediyor.
- Arayüz ve içerik Türkçe ve İngilizce.
- Study Area: sayaç, geri sayım, çalışma süresine oranla CEP puanı, kişisel istatistikler ve opsiyonel genel liderlik tablosu.
- UI kütüphanesi kullanmadan, saf CSS ile kurulmuş 6 temalı tasarım sistemi.
- Her materyalde içerik hatası bildirme butonu.
- JWT tabanlı kimlik doğrulama, e-posta doğrulama ve şifre sıfırlama.
- Responsive arayüz.

## Genel mimari

```mermaid
flowchart LR
    U["Öğrenci<br/>tarayıcı"] --> FE
    subgraph Yayin["Yayın ortamı"]
        FE["Frontend<br/>React, TypeScript, Vite<br/>Vercel"]
        API["Backend API<br/>ASP.NET Core 8<br/>Render"]
        DB[("PostgreSQL<br/>Supabase")]
    end
    FE -->|"REST (JWT)"| API
    FE <-.->|"SignalR"| API
    API -->|"EF Core"| DB
    API -->|"E-posta"| MAIL["Resend"]
```

Frontend, backend ve veritabanı ayrı servislerde çalışıyor. SignalR altyapısı gerçek zamanlı özellikler (sohbet ve oyun) için hazırlandı, bu özellikler geliştirme aşamasında.

## Ders sistemi

Platformun ana konusu bu sistem. Bir ders, konulardan; bir konu, materyallerden oluşuyor. Materyal ya bir **konu anlatımı** ya da bir **alıştırma**.

### Bir materyal nasıl akıyor

```mermaid
flowchart LR
    subgraph Konu["Konu anlatımı"]
        direction TB
        K1["Giriş<br/>bu materyalde ne öğreneceksin"] --> K2["Sezgi<br/>günlük hayattan örnek"]
        K2 --> K3["Kavram ve formül<br/>KaTeX ile"]
        K3 --> K4["Simülasyon<br/>kavramın hemen altında"]
        K4 --> K5["Çözümlü örnek"]
        K5 --> K6["Sık hatalar ve<br/>mühendislik notları"]
        K6 --> K7["Özet kutusu"]
    end
    subgraph Alistirma["Alıştırma"]
        direction TB
        A1["Soru kartı<br/>kolaydan zora"] --> A2["Çözümü göster"]
        A2 --> A3["Adım adım çözüm"]
        A3 --> A4["Sonuç"]
    end
```

Simülasyon, açıkladığı kavramın hemen altında yer alıyor, sayfanın sonunda toplanmıyor. Önce sezgi, sonra formül geliyor.

### Materyaller nasıl kayıt oluyor

Yeni bir materyal eklemek için kayıt kodu yazılmıyor. Klasör yapısı kaydı kendisi üretiyor.

```text
interactiveLessons/
├── InteractiveRegistry.ts
├── lessonCore.css
└── CSE4288_machineLearning/
    └── topic4/
        └── 2_minibatch_sgd_epochs/
            ├── index.tsx        # dile göre içeriği seçer
            ├── ContentTR.tsx    # Türkçe içerik
            ├── ContentEN.tsx    # İngilizce içerik
            ├── Animations.tsx   # simülasyonlar
            └── Animations.css   # materyale özel stiller
```

```mermaid
flowchart LR
    A["Klasör yapısı<br/>Ders / topicN / Materyal"] --> B["Vite import.meta.glob<br/>derleme sırasında tarar"]
    B --> C["InteractiveRegistry<br/>anahtar: Ders__Materyal"]
    C --> D["React.lazy<br/>materyal açılınca yüklenir"]
    D --> E{"Arayüz dili"}
    E -->|"TR"| F["ContentTR"]
    E -->|"EN"| G["ContentEN"]
    F --> H["Simülasyonlar"]
    G --> H
```

Materyaller açıldıkları anda yüklendiği için, içerik arttıkça ilk yükleme büyümüyor.

### Simülasyonlar

- Her simülasyon tek bir kavrama odaklanıyor, bir materyalde genelde 1 ile 3 tane var.
- Sürekli animasyonlu simülasyonlarda React state'i her karede güncellenmiyor. Hareketli elemanlar `ref` ile tutulup doğrudan DOM üzerinden güncelleniyor.
- Animasyon döngüleri sadece ekranda görünürken ve sekme açıkken çalışıyor.
- `prefers-reduced-motion` ayarı destekleniyor, geniş içerik (matris, tablo, formül) sayfayı yatay taşırmadan kendi kutusunda kayıyor.
- Ekranda görünen istatistikler gerçek hesaptan çıkıyor, sabit sonuç gösterilmiyor.

### Dersler

| Ders | Konu |
|---|---|
| CSE2023 | Ayrık Matematik |
| CSE3048 | Sinyaller |
| CSE3055 | Veritabanı Sistemleri |
| CSE3264 | Otomata |
| CSE4288 | Makine Öğrenmesi |
| STAT2253 | İstatistik |

## Study Area ve CEP

Study Area, öğrencilerin sayaç veya geri sayım başlatıp çalıştıkları alan. Çalışma süresine oranla **CEP** (CE-Hub puanı) kazanılıyor. Kişisel istatistikler görülebiliyor, isteyenler genel liderlik tablosuna katılabiliyor. Bu bölüm şu anda güncelleniyor.

```mermaid
stateDiagram-v2
    direction LR
    state "Hazır" as Hazir
    state "Aktif - sayaç veya geri sayım" as Aktif
    state "Duraklatıldı" as Duraklatildi
    state "Tamamlandı" as Tamamlandi
    [*] --> Hazir
    Hazir --> Aktif: başlat
    Aktif --> Aktif: heartbeat
    Aktif --> Duraklatildi: duraklat
    Duraklatildi --> Aktif: devam et
    Aktif --> Tamamlandi: bitir
    Tamamlandi --> [*]
    note right of Tamamlandi : süreye oranla CEP kazanılır, istatistikler güncellenir
```

CEP bakiyesi ve hareket geçmişi kayıt altında tutuluyor. Kazanılan puanların harcanacağı alan için [yol haritasına](#yol-haritası) bakın.

## Backend mimarisi

Backend, .NET 8 üzerinde Clean Architecture ile katmanlara ayrılmış. Bağımlılık yönü içe doğru: Domain hiçbir katmana bağlı değil.

```mermaid
flowchart TB
    API["API<br/>Controller, Filter, Middleware, Hub"] --> APP["Application<br/>DTO, Validator, Service, arayüzler"]
    API --> INF["Infrastructure<br/>EF Core, Repository, JWT, e-posta, depolama"]
    INF --> APP
    APP --> DOM["Domain<br/>Entity, Enum"]
    INF --> DOM
    GAME["Game<br/>oyun kuralları, geliştirme aşamasında"] -.-> DOM
    INF -.-> GAME
    API -.-> GAME
```

### Bir isteğin yolu

```mermaid
sequenceDiagram
    participant C as İstemci
    participant M as Hata middleware'i
    participant F as Validation filter
    participant K as Controller
    participant S as Service
    participant R as Repository
    participant D as PostgreSQL
    C->>M: HTTP isteği
    M->>F: kimlik doğrulama ve yetki sonrası
    F->>F: istek doğrulaması (FluentValidation)
    F->>K: geçerli istek
    K->>S: yalnızca Service çağrısı
    S->>R: veri erişimi
    R->>D: EF Core sorgusu
    D-->>R: sonuç
    S->>D: UnitOfWork ile commit
    S-->>K: yanıt modeli
    K-->>C: 200 OK
```

Controller'da iş mantığı yok. İş kuralları Service katmanında, veri erişimi Repository katmanında.

### Hata yönetimi

İş kuralı ihlalleri her yerde aynı biçimde ele alınıyor ve kullanıcıya kendi dilinde ulaşıyor.

```mermaid
flowchart LR
    S["Service<br/>iş kuralı ihlali"] -->|"ApiException(KOD)"| M["Hata middleware'i<br/>KOD → HTTP durum kodu"]
    M -->|"success false, code KOD"| A["apiClient"]
    A --> E["getApiErrorMessage"]
    E --> I["i18n çevirisi<br/>tr / en"]
    I --> T["Toast<br/>kullanıcının dilinde mesaj"]
```

Backend'in ürettiği her hata kodunun frontend'de bir çevirisi var. Çevirisi olmayan kod kullanıcıya ham gösterilmiyor.

## Kimlik doğrulama

```mermaid
sequenceDiagram
    actor U as Kullanıcı
    participant FE as Frontend
    participant API as Backend
    participant Mail as Resend
    U->>FE: kayıt formu
    FE->>API: kayıt isteği
    API->>Mail: doğrulama kodu e-postası
    Mail-->>U: kod
    U->>FE: kodu girer
    FE->>API: e-posta doğrulama
    U->>FE: giriş
    FE->>API: giriş isteği
    API-->>FE: access token ve refresh token
    Note over FE,API: access token geçersiz olunca
    FE->>API: refresh token ile yenileme
    API-->>FE: yenilenmiş token
```

Şifre sıfırlama da aynı yapıda: e-postayla gelen kodla yeni şifre belirleniyor. Girişte e-postası doğrulanmamış kullanıcılar ayrı bir hata koduyla karşılanıyor.

## Frontend mimarisi

Veri akışı tek yönlü ve katmanlı. Sayfalar doğrudan API çağırmıyor.

```mermaid
flowchart LR
    P["Page<br/>düzen ve hook çağrısı"] --> H["Hook<br/>state, yükleme, hata"]
    H --> SV["Service<br/>API çağrıları"]
    SV --> AC["apiClient<br/>axios, /v1"]
    AC --> BE["Backend API"]
    P --> C["Component<br/>sunum"]
```

```text
src/
├── api/                 apiClient
├── components/          ui, layout, student/{modül}, admin/{modül}
├── context/             AuthProvider, ThemeProvider
├── hooks/               common, student, admin
├── interactiveLessons/  ders materyalleri ve registry
├── locales/             tr.json, en.json
├── pages/               public, student, admin
├── routes/              AppRoutes, korumalı rotalar
├── services/            API servisleri
├── store/               zustand store'ları
├── types/               modül başına tip dosyası
└── utils/
```

## Teknoloji yığını

| Alan | Teknoloji |
|---|---|
| Frontend | React 19, TypeScript (strict), Vite, react-router, axios, zustand |
| Stil | Saf CSS, tema değişkenleri, 6 tema |
| Çoklu dil | i18next, react-i18next (TR / EN) |
| Matematik | KaTeX |
| Backend | ASP.NET Core (.NET 8, C# 12), Clean Architecture |
| Veri | Entity Framework Core, PostgreSQL |
| Doğrulama ve güvenlik | FluentValidation, JWT (access + refresh) |
| Gerçek zamanlı | SignalR |
| E-posta | Resend |
| Loglama | Serilog |
| Yayın | Vercel (frontend), Render (API), Supabase (veritabanı) |

## Tasarım kararları

- **UI kütüphanesi yok, saf CSS var.** Tüm renkler CSS değişkenlerinden geliyor, bileşen CSS'inde sabit renk yok. 6 tema `data-theme` ile değişiyor.
- **Sıfır sabit metin.** Kullanıcıya görünen her metin i18n üzerinden geliyor, `tr.json` ve `en.json` anahtar kümeleri birebir aynı tutuluyor.
- **Katmanlı frontend.** Service, Hook, Page, Component sırası sabit. Sayfalarda servis çağrısı yapılmıyor.
- **İnce controller.** İş mantığı Service'te, hatalar tek tip `ApiException` koduyla taşınıyor.
- **Zaman UTC.** Backend her yerde UTC ile çalışıyor, tarihler kullanıcının yerel saatinde ve diline göre biçimlendirilerek gösteriliyor.
- **Klasörden kayıt.** Ders materyalleri klasör yapısından otomatik bulunuyor, bu yüzden içerik eklemek kod değişikliği gerektirmiyor.
- **Erişilebilirlik.** Klavye odağı görünür kalıyor, ikon butonlarında etiket var, hareket azaltma tercihi destekleniyor.

## Yol haritası

```mermaid
flowchart LR
    subgraph Canli["Canlı"]
        L1["İnteraktif ders sistemi<br/>6 ders, yaklaşık 190 materyal"]
        L2["Kimlik doğrulama ve profil"]
        L3["Study Area ve CEP<br/>güncelleniyor"]
    end
    subgraph Gelistirme["Geliştirme aşamasında"]
        G1["Hu(B)ahçe<br/>sohbet ve kişisel bahçe"]
        G2["Oyun katmanı"]
    end
    subgraph Planlanan["Planlanan"]
        P1["Çıkmış sorular ve<br/>önceki çalışmalar"]
        P2["Okulda kullanılan yazılımlar için<br/>mini tutorial'lar"]
        P3["Yeni dersler ve materyaller"]
    end
    Canli --> Gelistirme
    Gelistirme --> Planlanan
```

- **Hu(B)ahçe:** Üniversiteye yeni başlayanların ve kulüplerin kendi aralarında rahatça sohbet edebileceği bir alan. Kazanılan CEP'in harcandığı, eşyaların alındığı ve bitkilerin yetiştirildiği kişisel bir bahçe fikri de bunun parçası. Kapsam henüz netleşiyor.
- **Oyun katmanı:** Gerçek zamanlı, SignalR üzerinden çalışan bir oyun için altyapı hazırlanıyor. Kuralları yeniden tasarlanıyor.
- **Çıkmış sorular ve mini tutorial'lar:** Henüz plan aşamasında.

## Kaynak kod ve iletişim

CE-Hub'ın kaynak kodu private. Bu repo yalnızca tanıtım amaçlıdır ve kaynak kod içermez. Mimariyi ve kodu görüşmede gösterebilirim.

**Ömer Ercan Dayan**, Marmara Üniversitesi Bilgisayar Mühendisliği

- GitHub: [ErcahPasha](https://github.com/ErcahPasha)
- LinkedIn: [Ömer Ercan Dayan](https://www.linkedin.com/in/%C3%B6mer-ercan-dayan-041124256/)
- E-posta: dayanomerercan@gmail.com
- Canlı proje: [cehub.dev](https://cehub.dev)

Tüm hakları saklıdır.
