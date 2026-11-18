# Build Variants (Варіанти збірки) в Android

Уявіть, що ви розробляєте застосунок і хочете зробити дві версії:

1. **Безкоштовну** (з рекламою).
2. **Платну / Pro** (без реклами).

Крім того, кожну з цих версій вам треба:

* Запускати у режимі розробки (**Debug**) з тестовим сервером, щоб ставити брейкпоінти та швидко тестувати.
* Збирати для випуску (**Release**) з production-сервером, увімкнувши оптимізацію та налаштувавши підписання для Google Play.

У цьому прикладі `free` / `paid` визначають версію продукту, а `debug` / `release` — режим збірки та сервер. Обидві версії продукту можна перевіряти на тестовому сервері.

Замість того, щоб копіювати весь проект і створювати окремі проекти для кожної версії, в Android (за допомогою системи збірки **Gradle**) існує потужний механізм: **Build Variants (Варіанти збірки)**.

---

## 1. Що таке Build Types та Product Flavors?

**Build Variant** — конфігурація, за якою збирається застосунок. APK чи AAB — артефакти, які можна отримати під час збірки цієї конфігурації.

Якщо flavors не оголошені, варіанти можуть називатися просто `debug` і `release`. Якщо flavors є, варіант поєднує **один Build Type та по одному Product Flavor з кожного оголошеного flavor dimension**. Для одного виміру це звична комбінація `free` + `debug` → `freeDebug`.

### 1. Build Type (Тип збірки)

Визначає, **ЯК** збирається ваш додаток (технічні налаштування компіляції та пакування).
За замовчуванням в Android існують два типи:

* **`debug`** — для розробки. За замовчуванням дозволяє налагодження, не вмикає мініфікацію R8 та підписується debug-ключем для встановлення на пристрої й емулятори. Такий підпис не підходить для публікації в Google Play.
* **`release`** — для випуску. За замовчуванням налагодження вимкнене, але сама назва типу не вмикає оптимізацію та не налаштовує підписання. Мініфікацію, оптимізацію й обфускацію коду через R8 потрібно увімкнути явно; для автоматичного підписання потрібен `signingConfig`.

R8 використовує правила у форматі ProGuard. Обфускація ускладнює читання коду, але не гарантує захисту від його аналізу. [Оптимізація R8](https://developer.android.com/topic/performance/app-optimization/enable-app-optimization), [підписання застосунку](https://developer.android.com/studio/publish/app-signing).

### 2. Product Flavor (Різновид продукту)

Визначає, **ЩО** збирається (бізнес-логіка, ресурси, іконки, конфігурації).
Ви самі створюєте флейвори. Наприклад, ви можете створити флейвори: `free` та `paid`, або `demo` та `full`, або `devServer` та `prodServer`.

### 3. Build Variant (Варіант збірки)

Це їхня матриця (перехрещення). Якщо у вас є `debug` / `release` та `free` / `paid`, Gradle автоматично згенерує 4 варіанти збірки:

1. `freeDebug`
2. `freeRelease`
3. `paidDebug`
4. `paidRelease`

Ви обираєте потрібний варіант в Android Studio (в панелі **Build Variants**) перед запуском додатка.

---

## 2. Як налаштувати Build Types?

Відкрийте файл `build.gradle.kts` вашого **модуля (app)**. Знайдіть блок `android { ... }`.
Там уже є блок `buildTypes`. Доповніть його та одразу увімкніть генерацію `BuildConfig`: нижче ми додаємо до нього власне поле `BASE_URL`.

Починаючи з **Android Gradle Plugin (AGP) 8.0**, генерація `BuildConfig` за замовчуванням вимкнена. Її налаштовуємо через `buildFeatures` у модулі; це поведінка AGP, а не самої системи Gradle. [Зміни в AGP 8.0](https://developer.android.com/build/releases/agp-8-0-0-release-notes#default-changes).

```kotlin
android {
    // ...

    buildFeatures {
        buildConfig = true
    }

    buildTypes {
        getByName("release") {
            isMinifyEnabled = true // Увімкнути стиснення та обфускацію коду
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            // Наприклад, в release ми хочемо інший базовий URL
            buildConfigField("String", "BASE_URL", "\"https://api.myproduction.com/\"")
        }
        
        getByName("debug") {
            // Для дебагу зазвичай isMinifyEnabled = false (за замовчуванням)
            applicationIdSuffix = ".debug" // Щоб можна було встановити debug і release одночасно на телефон!
            versionNameSuffix = "-DEBUG"
            buildConfigField("String", "BASE_URL", "\"https://api.testserver.com/\"")
        }
    }
}
```

Цей фрагмент вмикає оптимізацію коду для `release`, але ще не налаштовує його підписання. Щоб отримати пакет для публікації, задайте release `signingConfig` або скористайтеся **Build → Generate Signed Bundle / APK**. Створення ключа та конфігурацію підписання описано в [інструкції Android](https://developer.android.com/studio/publish/app-signing#generate-key).

> 💡 **Магія `applicationIdSuffix`:** Якщо базовий пакет вашого додатка `com.example.app`, то додавши `.debug`, ваш дебажний додаток отримає ідентифікатор `com.example.app.debug`. Для операційної системи Android це **два абсолютно різні додатки**. Це дозволить вам мати на телефоні одразу дві іконки: звичайний додаток і тестовий!

---

## 3. Як налаштувати Product Flavors?

Щоб створити `Product Flavors`, вам спочатку треба задати "вимір" (flavor dimension). Наприклад, вимір "версія" (version) або "середовище" (environment). Потім додати самі флейвори.

Додаємо це у той самий блок `android { ... }`:

```kotlin
android {
    // ...

    // 1. Оголошуємо виміри (можна мати декілька, наприклад "tier" і "environment")
    flavorDimensions += "tier"

    // 2. Описуємо самі флейвори
    productFlavors {
        create("free") {
            dimension = "tier"
            // Змінюємо ім'я додатка, щоб юзер бачив різницю на робочому столі
            manifestPlaceholders["appName"] = "MyApp Free" 
            applicationIdSuffix = ".free"
        }
        create("paid") {
            dimension = "tier"
            manifestPlaceholders["appName"] = "MyApp Pro"
            applicationIdSuffix = ".pro"
        }
    }
}
```

> *Примітка:* Щоб `manifestPlaceholders["appName"]` запрацював, у вашому `AndroidManifest.xml` треба змінити хардкод на плейсхолдер: `android:label="${appName}"`.

За базового `applicationId = "com.example.app"` спільна конфігурація build types та flavors дасть такі результати:

| Варіант | Application ID | Сервер |
| --- | --- | --- |
| `freeDebug` | `com.example.app.free.debug` | Тестовий |
| `freeRelease` | `com.example.app.free` | Production |
| `paidDebug` | `com.example.app.pro.debug` | Тестовий |
| `paidRelease` | `com.example.app.pro` | Production |

Суфікс build type додається після суфікса flavor. Наведені налаштування змінюють назву, ідентифікатор і URL; логіку показу реклами потрібно реалізувати окремо, наприклад у коді flavor, як у розділі 5.

---

## 4. BuildConfig та доступ до змінних у Коді

Вище у `buildTypes` ми написали магічний рядок:
`buildConfigField("String", "BASE_URL", "\"https://api.testserver.com/\"")`

Але як дістати його у Kotlin-коді (наприклад, щоб передати у Retrofit)?
AGP генерує клас **`BuildConfig`** із полями для вибраного варіанта. Цей клас доступний із Kotlin; його генерацію ми вже увімкнули через `buildFeatures { buildConfig = true }` у розділі 2.

Виконайте Gradle Sync та збірку вибраного варіанта, щоб згенерувати клас. У Kotlin імпортуйте `BuildConfig` свого модуля: його пакет визначається `namespace`. Суфікс `applicationIdSuffix` цей пакет не змінює. Наприклад, за `namespace = "com.example.app"` потрібен `import com.example.app.BuildConfig`.

Після цього у коді з уже підключеним Retrofit можна використати:

```kotlin
val retrofit = Retrofit.Builder()
    .baseUrl(BuildConfig.BASE_URL) // Береться напряму з поточного Build Variant!
    .build()

if (BuildConfig.DEBUG) {
    Log.d("App", "Ми у режимі розробки!")
}
```

---

## 5. Різні ресурси та код для різних Flavors

Флейвори вміють керувати не тільки рядками у Gradle, а й цілими файлами! 
Кожен `Flavor`, `Build Type` та конкретний варіант збірки може мати власний **source set** — набір каталогів із кодом і ресурсами.

За замовчуванням весь ваш код лежить у:
`app/src/main/java/...`
та ресурси у:
`app/src/main/res/...`

Якщо ви створили флейвор `free`, ви можете створити для нього папку:
`app/src/free/res/values/strings.xml`
`app/src/free/res/drawable/ic_logo.png`

### Ресурси: накладання за пріоритетом

Для `freeDebug` пріоритет ресурсів, від найвищого до найнижчого, такий:

1. `src/freeDebug/res/` — конкретний варіант.
2. `src/debug/res/` — build type.
3. `src/free/res/` — product flavor.
4. `src/main/res/` — спільні ресурси.

За збігу ресурсу з однаковою конфігурацією обирається значення з вищим пріоритетом. Наприклад, `ic_logo` із `free` замінить спільну іконку в результаті збірки, якщо її не перевизначили в `debug` або `freeDebug`. Файли в `main` на диску залишаються без змін.

У `values/strings.xml` об’єднуються окремі ресурси за іменами: перевизначення `app_name` не видаляє решту спільних рядків із `main`.

### Kotlin та Java: спільна компіляція

Код усіх активних source sets компілюється разом. Однаковий клас з тим самим повним ім’ям у `main` та `free` спричинить помилку дублювання. Для альтернативних реалізацій розмістіть клас у `free` та `paid`, а з `main` його приберіть.

Наприклад, `app/src/free/java/com/example/app/AdsConfig.kt`:

```kotlin
package com.example.app

object AdsConfig {
    const val ENABLED = true
}
```

А `app/src/paid/java/com/example/app/AdsConfig.kt`:

```kotlin
package com.example.app

object AdsConfig {
    const val ENABLED = false
}
```

Спільний код із `main` може використовувати `AdsConfig.ENABLED` для вибору поведінки. До кожного варіанта потрапить одна реалізація, оскільки flavors `free` та `paid` належать одному виміру й не використовуються одночасно. Сам прапорець не підключає рекламний SDK і не показує рекламу — це робить код застосунку.

Так можна мати різні ресурси й реалізації класів для версій продукту в одному проєкті. [Правила source sets та їхнього пріоритету](https://developer.android.com/build/build-variants#sourcesets).

---

## Підсумок

1. **Build Type (Тип збірки)** — відповідає за технічні налаштування (debug, release, мініфікація, ключі підпису).
2. **Product Flavor (Різновид)** — відповідає за бізнес-версії (free/paid, dev/prod), різні іконки, ресурси, URL.
3. **Build Variant** — конфігурація з одного build type та, за наявності flavors, одного flavor з кожного виміру (наприклад, `freeDebug`).
4. **`applicationIdSuffix`** дозволяє встановлювати кілька версій вашого додатка на один телефон одночасно.
5. **`BuildConfig`** дозволяє передавати змінні з Gradle-файлу прямо у ваш Kotlin-код (дуже корисно для базових URL та ключів API).
6. **Source sets** дозволяють доповнювати й перевизначати ресурси за пріоритетом та підключати код для конкретної версії. Kotlin- і Java-класи компілюються разом, тому дублювати клас між активними source sets не можна.
