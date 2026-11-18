# Build Variants (Варіанти збірки) в Android

Уявіть, що ви розробляєте застосунок і хочете зробити дві версії:
1. **Безкоштовну** (з рекламою, підключається до тестового сервера).
2. **Платну / Pro** (без реклами, підключається до реального сервера (production)).

Крім того, кожну з цих версій вам треба:
* Запускати у режимі розробки (**Debug**), щоб бачити логи та швидко тестувати.
* Збирати у фінальний релізний пакет (**Release**) для завантаження в Google Play (оптимізований, підписаний ключем).

Замість того, щоб копіювати весь проект і створювати окремі проекти для кожної версії, в Android (за допомогою системи збірки **Gradle**) існує потужний механізм: **Build Variants (Варіанти збірки)**.

---

## 1. Що таке Build Types та Product Flavors?

Будь-який `Build Variant` (кінцевий APK чи AAB файл) формується як комбінація двох сутностей: **Build Type** + **Product Flavor**.

### 1. Build Type (Тип збірки)
Визначає, **ЯК** збирається ваш додаток (технічні налаштування компіляції та пакування).
За замовчуванням в Android існують два типи:
* **`debug`** — для розробки. Код не оптимізований, можна ставити брейкпоінти (дебажити), додаток підписується стандартним debug-ключем, який підходить лише для емуляторів/телефонів.
* **`release`** — для продакшену (Google Play). Код мініфіковано (мінімізовано розмір), обфусковано (щоб ніхто не міг легко прочитати ваш код за допомогою реверс-інжинірингу) через R8/ProGuard, додаток підписується вашим унікальним криптографічним ключем розробника.

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
Там уже є блок `buildTypes`.

```kotlin
android {
    // ...

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

---

## 4. BuildConfig та доступ до змінних у Коді

Вище у `buildTypes` ми написали магічний рядок:
`buildConfigField("String", "BASE_URL", "\"https://api.testserver.com/\"")`

Але як дістати його у Kotlin-коді (наприклад, щоб передати у Retrofit)?
Gradle під час збірки автоматично генерує спеціальний Kotlin-клас, який називається **`BuildConfig`**.

> ⚠️ **Важливо в сучасному Android:** Починаючи з нових версій Gradle, генерація класу `BuildConfig` вимкнена за замовчуванням. Щоб її увімкнути, додайте це у ваш `build.gradle.kts` модуля:
```kotlin
android {
    buildFeatures {
        buildConfig = true
    }
}
```

Після синхронізації проекту (`Sync Now`) ви зможете написати у своєму коді:

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
Кожен `Flavor` або `BuildType` має право на свою власну папку з кодом та ресурсами.

За замовчуванням весь ваш код лежить у:
`app/src/main/java/...`
та ресурси у:
`app/src/main/res/...`

Якщо ви створили флейвор `free`, ви можете створити для нього папку:
`app/src/free/res/values/strings.xml`
`app/src/free/res/drawable/ic_logo.png`

**Як це працює при збірці:**
Коли ви збираєте варіант `freeDebug`, Gradle бере ресурси з `src/main`, а потім "накладає" поверх них ресурси з `src/free`. Якщо є збіг (наприклад, іконка з такою ж назвою), файл із папки `free` **перезапише** базовий файл. 

Таким чином, ви можете мати різні іконки, різні кольори теми, різні переклади та навіть різні класи Kotlin для безкоштовної та платної версій додатку, тримаючи їх в одному проекті!

---

## Підсумок

1. **Build Type (Тип збірки)** — відповідає за технічні налаштування (debug, release, мініфікація, ключі підпису).
2. **Product Flavor (Різновид)** — відповідає за бізнес-версії (free/paid, dev/prod), різні іконки, ресурси, URL.
3. **Build Variant** — це перехрещення типу та різновиду (напр. `freeDebug`), яке ви обираєте в IDE.
4. **`applicationIdSuffix`** дозволяє встановлювати кілька версій вашого додатка на один телефон одночасно.
5. **`BuildConfig`** дозволяє передавати змінні з Gradle-файлу прямо у ваш Kotlin-код (дуже корисно для базових URL та ключів API).
6. **Папки флейворів** (`src/free/res/`) дозволяють повністю замінювати ресурси для різних версій продукту.
