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
android { // Налаштування Android-модуля
    // ...

    buildFeatures { // Вмикаємо потрібні можливості генерації коду
        buildConfig = true // Генерувати клас BuildConfig, зокрема наші поля BASE_URL
    }

    buildTypes { // Налаштування технічних режимів збірки
        getByName("release") { // Доповнюємо наявний тип release, а не створюємо новий
            isMinifyEnabled = true // Увімкнути видалення невикористаного коду, оптимізацію та обфускацію R8
            proguardFiles( // Файли правил для R8, зокрема які класи та методи потрібно зберегти
                getDefaultProguardFile("proguard-android-optimize.txt"), // Стандартні правила Android з оптимізацією
                "proguard-rules.pro" // Додаткові правила цього модуля
            )
            // Аргументи: тип поля, його ім'я та вираз значення для згенерованого коду.
            // Екрановані лапки \" потрібні, щоб значення стало рядковим літералом.
            // У release поле BuildConfig.BASE_URL міститиме адресу production-сервера.
            buildConfigField("String", "BASE_URL", "\"https://api.myproduction.com/\"")
        }

        getByName("debug") { // Доповнюємо наявний тип debug
            // Для дебагу зазвичай isMinifyEnabled = false (за замовчуванням)
            applicationIdSuffix = ".debug" // Додати .debug до ID застосунку для окремого встановлення
            versionNameSuffix = "-DEBUG" // Додати суфікс до назви версії: наприклад, 1.0 → 1.0-DEBUG; ID не змінює
            // Те саме поле BuildConfig.BASE_URL, але з адресою тестового сервера для debug.
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
android { // Продовжуємо налаштування того самого Android-модуля
    // ...

    // Вимір — група взаємовиключних flavors. Назву tier (рівень продукту) обираємо самі.
    flavorDimensions += "tier" // Додати вимір tier до списку; можна також оголосити інші, наприклад environment

    productFlavors { // Описуємо версії продукту, які поєднуватимуться з build types
        create("free") { // Створити flavor free: частину назв freeDebug та freeRelease
            dimension = "tier" // Віднести free до виміру tier
            // Підставити MyApp Free замість ${appName} в Manifest — це видима назва застосунку.
            manifestPlaceholders["appName"] = "MyApp Free"
            applicationIdSuffix = ".free" // Додати .free до ID: com.example.app → com.example.app.free
        }
        create("paid") { // Створити альтернативний flavor paid: paidDebug та paidRelease
            dimension = "tier" // Той самий вимір: у варіант потрапить або free, або paid
            manifestPlaceholders["appName"] = "MyApp Pro" // Видима назва платної версії в Manifest
            applicationIdSuffix = ".pro" // Додати .pro до ID: com.example.app → com.example.app.pro
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

<a id="gradle-parameters"></a>

### Довідник параметрів Gradle для Android-модуля

У таблиці зібрані параметри та виклики Kotlin DSL з попередніх прикладів і кілька пов’язаних налаштувань. У колонці «Де задавати» шляхи наведено відносно блока `android { ... }` у модульному `build.gradle.kts`. `productFlavors` і `buildTypes` означають блок конкретного flavor або типу збірки, наприклад `create("free")` чи `getByName("debug")`.

| Параметр або виклик | Де задавати | Що означає | Приклад |
| --- | --- | --- | --- |
| `namespace` | Безпосередньо в `android` | Пакет згенерованих класів `R` і `BuildConfig` | `namespace = "com.example.app"` |
| `applicationId` | `defaultConfig` або `productFlavors` | Ідентифікатор застосунку на пристрої та в Google Play | `applicationId = "com.example.app"` |
| `applicationIdSuffix` | `productFlavors` або `buildTypes` | Додає суфікс до ID, дозволяючи окреме встановлення версій | `applicationIdSuffix = ".free"` → `com.example.app.free` |
| `versionCode` | `defaultConfig` або `productFlavors` | Цілий номер для порівняння версій; для наступного оновлення його збільшують | `versionCode = 2` |
| `versionName` | `defaultConfig` або `productFlavors` | Назва версії для користувача | `versionName = "1.1"` |
| `versionNameSuffix` | `productFlavors` або `buildTypes` | Додає суфікс до назви версії, не змінюючи ID | `versionNameSuffix = "-DEBUG"` → `1.1-DEBUG` |
| `flavorDimensions` | Безпосередньо в `android` | Список вимірів для групування flavors | `flavorDimensions += "tier"` |
| `dimension` | `productFlavors` | Вимір, до якого належить цей flavor | `dimension = "tier"` |
| `manifestPlaceholders` | `defaultConfig`, `productFlavors` або `buildTypes` | Значення для підстановки замість `${...}` у Manifest | `manifestPlaceholders["appName"] = "MyApp Free"` |
| `buildConfig` | `buildFeatures` | Вмикає генерацію класу `BuildConfig` | `buildConfig = true` |
| `buildConfigField(...)` | `defaultConfig`, `productFlavors` або `buildTypes` | Додає поле до `BuildConfig`: тип, ім’я, вираз значення | `buildConfigField("boolean", "ADS_ENABLED", "true")` |
| `isDebuggable` | `buildTypes` | Дозволяє налагодження застосунку | `isDebuggable = true` |
| `isMinifyEnabled` | `buildTypes` | Вмикає видалення невикористаного коду, оптимізацію й обфускацію R8 | `isMinifyEnabled = true` |
| `isShrinkResources` | `buildTypes` | Вмикає видалення невикористаних ресурсів; застосовується разом із мініфікацією коду | `isShrinkResources = true` |
| `proguardFiles(...)` | `defaultConfig`, `productFlavors` або `buildTypes` | Підключає файли правил R8 | `proguardFiles("proguard-rules.pro")` |
| `signingConfig` | `defaultConfig`, `productFlavors` або `buildTypes` | Обирає попередньо створену конфігурацію підписання | `signingConfig = signingConfigs.getByName("release")` |
| `getByName(...)` | Наприклад, `buildTypes` | Знаходить наявний елемент для налаштування | `getByName("debug") { ... }` |
| `create(...)` | Наприклад, `productFlavors` | Створює новий елемент конфігурації | `create("free") { ... }` |

`namespace`, `applicationId` і видима назва застосунку мають різне призначення. Наприклад, додавання `.free` змінює ID, але не пакет `BuildConfig`; назву під іконкою в нашому прикладі задає `appName` через Manifest. [Налаштування модуля](https://developer.android.com/build/configure-app-module), [налаштування варіантів](https://developer.android.com/build/build-variants).

Для R8 у таблиці використано той самий DSL, що й у прикладі розділу 2. У новому DSL AGP 9.3+ ці налаштування об’єднує блок `optimization`; відповідний синтаксис наведений у [документації оптимізації](https://developer.android.com/topic/performance/app-optimization/enable-app-optimization).

---

## 4. BuildConfig та доступ до змінних у Коді

Короткі пояснення `buildConfig` та `buildConfigField(...)` можна знайти в [довіднику параметрів Gradle](#gradle-parameters).

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

## 6. Кастомне налаштування через Variant API

`buildTypes` і `productFlavors` достатньо для більшості статичних налаштувань. Якщо потрібно відкинути певну комбінацію або обчислити значення для кожного варіанта, використовуйте **Variant API** через блок `androidComponents { ... }`.

Цей блок розміщують у модульному `build.gradle.kts` **поруч з `android { ... }`, а не всередині нього**. Його callback-и виконуються під час конфігурації збірки, а не під час запуску застосунку чи виконання окремої Gradle-задачі.

### Коли використовувати кожен callback?

| Callback | Що отримуємо | Для чого використовувати |
| --- | --- | --- |
| `finalizeDsl { ... }` | Об’єкт Android DSL | Завершити налаштування `buildTypes`, `productFlavors` та інших DSL-параметрів перед створенням варіантів; корисно в convention plugins |
| `beforeVariants { builder -> ... }` | `VariantBuilder` | Вимкнути непотрібні варіанти чи змінити доступні параметри builder до створення варіанта |
| `onVariants { variant -> ... }` | `Variant` | Налаштувати доступні властивості створеного варіанта, поля `BuildConfig`, підключення генераторів коду та роботу з артефактами |

Порядок етапів: налаштування `android` → `finalizeDsl` → `beforeVariants` → `onVariants`. Вимикати варіант у `onVariants` уже пізно. [Життєвий цикл Variant API](https://developer.android.com/build/extend-agp#variant-api-artifacts-and-tasks).

### Приклад: відкинути `paidDebug` та додати мітку збірки

Припустімо, у навчальному проєкті вирішили перевіряти платну версію лише у `release`. Це додаткове правило прикладу: сам Android не забороняє `paidDebug`.

До наявного `build.gradle.kts` додайте імпорт на початку файлу, а `androidComponents` — окремим блоком після `android`. Налаштування flavors та `buildFeatures { buildConfig = true }` з попередніх розділів залишаються.

```kotlin
import com.android.build.api.variant.BuildConfigField // Опис поля згенерованого BuildConfig

// plugins { ... } та android { ... } уже налаштовані вище.

androidComponents {
    // Обираємо конкретну комбінацію flavor paid і build type debug.
    beforeVariants(selector().withName("paidDebug")) { builder ->
        builder.enable = false // Не створювати цей варіант і його задачі збірки
    }

    // Налаштовуємо кожен варіант, який залишився увімкненим.
    onVariants(selector().all()) { variant ->
        variant.buildConfigFields?.put( // Додаємо поле, якщо генерація BuildConfig доступна
            "BUILD_LABEL", // Ім'я поля: у коді застосунку це BuildConfig.BUILD_LABEL
            BuildConfigField(
                type = "String", // Тип поля у згенерованому класі
                value = "\"${variant.name}\"", // Рядковий літерал, наприклад "freeDebug"
                comment = "Назва варіанта збірки" // Коментар біля згенерованого поля
            )
        )
    }
}
```

Тепер із чотирьох комбінацій залишаться `freeDebug`, `freeRelease` та `paidRelease`. `paidDebug` не буде доступний для збірки. У `freeDebug` поле `BuildConfig.BUILD_LABEL` матиме значення `"freeDebug"`, у `paidRelease` — `"paidRelease"`.

`buildConfigField(...)` з [довідника параметрів](#gradle-parameters) налаштовує поле через Android DSL. `variant.buildConfigFields` робить це через Variant API, коли вже відома конкретна комбінація. Не потрібно оголошувати `BUILD_LABEL` обома способами. [Офіційний приклад додавання полів через Variant API](https://github.com/android/gradle-recipes/tree/agp-9.0/addCustomBuildConfigFields).

### `selector()`: обробляємо потрібні варіанти

| Селектор | До яких варіантів застосує callback |
| --- | --- |
| `selector().all()` | До всіх відповідних варіантів |
| `selector().withBuildType("release")` | До всіх release-варіантів: `freeRelease`, `paidRelease` |
| `selector().withFlavor("tier" to "free")` | До варіантів із flavor `free` у вимірі `tier` |
| `selector().withName("paidRelease")` | До одного варіанта з точним ім’ям `paidRelease` |

Селектор обмежує виклики callback, але сам по собі не вимикає інші варіанти. Для вимкнення потрібне `builder.enable = false` у `beforeVariants`.

Якщо значення залежить від результату власної Gradle-задачі, підключайте її вихід через `Provider`, зберігаючи відкладене обчислення. Для генерованих файлів використовуйте API `sources` або `artifacts`, а задачі реєструйте через `tasks.register(...)`. Так Gradle може визначити залежності між задачами; не потрібно вгадувати назви внутрішніх задач AGP або змінювати файли в `build/` вручну. Приклади таких інтеграцій є в [Android Gradle recipes](https://github.com/android/gradle-recipes).

Після Gradle Sync перевірте список у **Build Variants**. Для модуля з назвою `app` можна зібрати APK командою `./gradlew :app:assembleFreeDebug` або AAB командою `./gradlew :app:bundlePaidRelease`. Для встановлення release APK та публікації AAB потрібне налаштоване підписання, як описано в розділі 2.

---

## Підсумок

1. **Build Type (Тип збірки)** — відповідає за технічні налаштування (debug, release, мініфікація, ключі підпису).
2. **Product Flavor (Різновид)** — відповідає за бізнес-версії (free/paid, dev/prod), різні іконки, ресурси, URL.
3. **Build Variant** — конфігурація з одного build type та, за наявності flavors, одного flavor з кожного виміру (наприклад, `freeDebug`).
4. **`applicationIdSuffix`** дозволяє встановлювати кілька версій вашого додатка на один телефон одночасно.
5. **`BuildConfig`** дозволяє передавати змінні з Gradle-файлу прямо у ваш Kotlin-код (дуже корисно для базових URL та ключів API).
6. **Source sets** дозволяють доповнювати й перевизначати ресурси за пріоритетом та підключати код для конкретної версії. Kotlin- і Java-класи компілюються разом, тому дублювати клас між активними source sets не можна.
7. **Variant API** дає доступ до кастомного налаштування: `beforeVariants` — для відбору варіантів, `onVariants` — для роботи з доступними властивостями й артефактами створених варіантів.
