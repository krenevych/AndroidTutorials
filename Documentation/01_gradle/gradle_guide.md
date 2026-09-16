# Довідник з Gradle для Android-розробника

Gradle — це потужна система автоматизації збірки, яка компілює вихідний код, підключає сторонні бібліотеки, збирає ресурси та генерує фінальний інсталяційний файл (`.apk` або `.aab`). За замовчуванням в Android-проектах використовується Kotlin DSL.

Ця документація об'єднує ключові аспекти роботи з Gradle: від розуміння структури проекту до налаштування власних задач та оптимізації збірки.

---

## 1. Gradle Wrapper (`gradlew`) та базові команди

**Gradle Wrapper** гарантує, що всі розробники та CI/CD сервери використовують однакову версію інструменту. Він автоматично завантажує Gradle за потреби.

Всі команди виконуються в терміналі кореневої папки проекту:
* **macOS / Linux:** `./gradlew <назва_задачі>`
* **Windows:** `gradlew.bat <назва_задачі>` (або `.\gradlew`)

### Основні команди CLI

1. Вивести інформацію про версію Gradle, JVM та ОС:
   ```bash
   ./gradlew --version
   ```
2. Переглянути список усіх доступних задач збірки:
   ```bash
   ./gradlew tasks
   ```
3. Очистити директорію `build/` (видалити проміжні артефакти):
   ```bash
   ./gradlew clean
   ```
4. Зібрати налагоджувальний APK. Результат зберігається в `app/build/outputs/apk/debug/app-debug.apk`:
   ```bash
   ./gradlew assembleDebug
   ```
5. Зібрати оптимізований релізний APK:
   ```bash
   ./gradlew assembleRelease
   ```
6. Зібрати debug-версію та одразу встановити на підключений девайс:
   ```bash
   ./gradlew installDebug
   ```
7. Запустити локальні юніт-тести (JUnit):
   ```bash
   ./gradlew test
   ```
8. Примусово завершити всі фонові процеси (Gradle Daemons):
   ```bash
   ./gradlew --stop
   ```

---

## 2. Структура Android-проекту з Gradle

Коли ви створюєте новий проект в Android Studio, генерується стандартна структура директорій. Налаштування розділяються на глобальні (для всього проекту) та локальні (для конкретних модулів).

```text
MyApplication/
├── gradle/
│   ├── wrapper/
│   │   └── gradle-wrapper.properties  # Точна версія Gradle для збірки
│   └── libs.versions.toml             # Каталог версій залежностей (Version Catalog)
├── gradlew                            # Скрипт запуску Gradle Wrapper для macOS/Linux
├── gradlew.bat                        # Скрипт запуску Gradle Wrapper для Windows
├── gradle.properties                  # Системні налаштування (JVM, кешування, AndroidX)
├── settings.gradle.kts                # Опис репозиторіїв та перелік модулів проекту
├── build.gradle.kts                   # Кореневий файл збірки (загальні плагіни)
├── local.properties                   # Локальні конфігурації (шлях до SDK), не додається в Git
└── app/
    ├── build.gradle.kts               # Конфігурація конкретного модуля (додатка)
    └── src/
        ├── main/                      # Основний код додатка
        │   ├── AndroidManifest.xml    # Головний конфігураційний файл
        │   ├── java/ (або kotlin/)    # Вихідний код
        │   └── res/                   # Ресурси (layout, strings, drawable)
        ├── test/                      # Локальні юніт-тести (JVM)
        └── androidTest/               # Інструментальні тести (Емулятор / Пристрій)
```

---

## 3. Ключові конфігураційні файли (settings та build)

У Gradle проектах налаштування розділені на рівень усього проекту та рівень окремих модулів. 

### 3.1. Файл налаштувань проекту (`settings.gradle.kts`)

Цей файл виконується найпершим під час ініціалізації збірки. Він відповідає за дві критично важливі речі:
1. **Підключення модулів:** вказує Gradle, які саме модулі входять до складу проекту.
2. **Централізовані репозиторії:** визначає, звідки Gradle має завантажувати плагіни (`pluginManagement`) та сторонні бібліотеки (`dependencyResolutionManagement`).

**Приклад settings.gradle.kts:**

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    // Забороняє модулям використовувати власні репозиторії (усе централізовано тут)
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // maven { url = uri("https://jitpack.io") } // Приклад підключення кастомного репозиторію
    }
}

rootProject.name = "MyApplication"
include(":app")
// include(":feature:auth") // Приклад підключення додаткових модулів (якщо проект багатомодульний)
```

### 3.2. Кореневий файл збірки (`build.gradle.kts` у корені проекту)
Цей файл (Top-level build file) відповідає за конфігурацію, спільну для всіх модулів у проекті. Зазвичай тут підключаються глобальні плагіни з використанням `apply false`. Це означає, що плагін лише реєструється в проекті (визначається його версія для всіх модулів), але безпосередньо не застосовується до кореневої директорії.

**Приклад кореневого build.gradle.kts:**

```kotlin
plugins {
    // Підключаємо плагіни з Version Catalog (libs.versions.toml) 
    // apply false означає, що плагін не застосовується до кореневого проекту, 
    // а лише фіксує версію для вкладених модулів
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.ksp) apply false
}
```
> **Важливо:** Для додавання нових інструментів (наприклад, Hilt, Firebase Crashlytics, SafeArgs) версію плагіна слід оголошувати саме тут або у `libs.versions.toml`, а застосовувати (без `apply false`) — вже в модульному файлі.

### 3.3. Конфігурація модуля (`app/build.gradle.kts`)

Цей файл описує правила компіляції та пакування коду для конкретного модуля (наприклад, додатку). На відміну від кореневого файлу, тут плагіни застосовуються для виконання реальних задач, а також налаштовуються Android-специфічні параметри.

### Блок plugins

Цей блок застосовує плагіни (версії яких зареєстровані в кореневому `build.gradle.kts` або `libs.versions.toml`) безпосередньо до поточного модуля. Саме завдяки цим плагінам Gradle "розуміє", що це Android-додаток та як компілювати Kotlin код.

> **Примітка:** Плагіни — це інструменти, які розширюють можливості самої системи збірки Gradle (компіляція, генерація коду, обробка ресурсів). Вони використовуються **виключно під час збірки** і, на відміну від бібліотек (`dependencies`), **не потрапляють** у кінцевий додаток (APK/AAB).

```kotlin
plugins {
    // Вказує, що це Android додаток (генерує APK)
    alias(libs.plugins.android.application)
    
    // Вказує, що в проекті використовується Kotlin
    alias(libs.plugins.kotlin.android)
    
    // Інші плагіни, наприклад, для серіалізації або KSP
    // alias(libs.plugins.kotlin.serialization)
}
```

### Параметри SDK та Версіонування

Блок `android { ... }` стає доступним виключно завдяки застосованому вище плагіну `android.application` (або `android.library`). Саме в ньому сконцентровані всі специфічні для платформи Android налаштування.

```kotlin
android {
    namespace = "com.example.myapp"
    compileSdk = 35 // Версія Android SDK для компіляції коду (дозволяє використовувати нові API)

    defaultConfig {
        applicationId = "com.example.myapp" // Унікальний ідентифікатор у Google Play та на ОС
        minSdk = 26     // Мінімальна версія ОС, на якій запуститься додаток
        targetSdk = 35  // Версія ОС, під яку протестовано додаток

        versionCode = 1       // Число для магазину (зростає з кожним релізом)
        versionName = "1.0.0" // Рядок, який бачить користувач
    }

    buildTypes {
        debug {
            // Суфікс дозволяє тримати debug і release версії як окремі додатки на пристрої
            applicationIdSuffix = ".debug"
        }
        release {
            isMinifyEnabled = true   // Вмикає обфускацію та мініфікацію коду (R8/ProGuard)
            isShrinkResources = true // Видаляє невикористані ресурси (зображення, xml)
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

---

## 4. Керування залежностями (Version Catalog)

У цьому розділі йдеться про підключення зовнішніх бібліотек (наприклад, для роботи з мережею, завантаження зображень тощо) до вашого проекту. На відміну від плагінів, ці бібліотеки компілюються разом із вашим кодом і безпосередньо потрапляють у фінальний додаток (або використовуються локально для тестування). Підключення потрібних бібліотек виконується у блоці `dependencies { ... }` відповідного модульного файлу `build.gradle.kts`.

Наприклад (пряме підключення):
```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
}
```
> **Анатомія залежності:** У рядку `"com.squareup.retrofit2:retrofit:2.9.0"`:
> * `com.squareup.retrofit2` — це **group ID** (організація або проект, що створила бібліотеку).
> * `retrofit` — це **artifact ID** (конкретна назва самої бібліотеки).
> * `2.9.0` — це **версія** бібліотеки.

Сучасний стандарт в Android — виносити централізовану конфігурацію версій усіх бібліотек у файл `gradle/libs.versions.toml` (так званий Version Catalog).

### Структура `libs.versions.toml`

- `[versions]` — зберігає версії.
- `[libraries]` — оголошує самі залежності (бібліотеки).
- `[plugins]` — підключені плагіни.
- `[bundles]` — дозволяє об'єднувати декілька бібліотек під одним іменем.

**Приклад:**

```toml
[versions]
kotlin = "2.0.20"
coil = "2.6.0"
retrofit = "2.9.0"
junit = "4.13.2"

[libraries]
coil-kt = { module = "io.coil-kt:coil", version.ref = "coil" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
junit = { module = "junit:junit", version.ref = "junit" }

[bundles]
network = ["retrofit"]

[plugins]
android-application = { id = "com.android.application", version = "8.7.0" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

### Підключення в `app/build.gradle.kts`

Залежності додаються у блоці `dependencies`. 
Вони мають різні "зони видимості" (Scopes):

* **`implementation`**: Основна залежність; бібліотека потрапить у кінцевий APK та буде доступна в коді додатку.
* **`testImplementation`**: Залежність тільки для локальних Unit-тестів (наприклад, JUnit, директорія `src/test/`). Вона **не потрапляє** у фінальний APK-файл.
* **`androidTestImplementation`**: Тільки для інструментальних тестів (`src/androidTest/`).
* **`api`**: Пакується в збірку та транслюється іншим залежним модулям.
* **`compileOnly`**: Потрібна лише під час компіляції (не потрапляє в APK).

```kotlin
dependencies {
    implementation(libs.coil.kt)
    implementation(libs.bundles.network) // Підключення групи бібліотек

    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
}
```

### Аналіз дерева залежностей
Іноді різні бібліотеки тягнуть різні версії однієї і тієї ж транзитивної залежності. Тоді виникає конфлікт версій. 

Для проведення діагностики використовуйте такі команди:
- `./gradlew :app:dependencies --configuration releaseRuntimeClasspath` — будує повне дерево залежностей.
- `./gradlew :app:dependencyInsight --dependency <назва> --configuration releaseRuntimeClasspath` — детально показує, звідки підтягується конкретна бібліотека і як Gradle вирішує конфлікт версій (Resolution Strategy зазвичай обирає найновішу версію).



---

## 5. Build Variants, Product Flavors та BuildConfig

### Product Flavors
Дозволяють створювати різні версії додатка (наприклад, платна/безкоштовна) з одного коду. Комбінація Build Types та Product Flavors утворює **Build Variants** (наприклад, `freeDebug`).

```kotlin
android {
    flavorDimensions += "tier"
    productFlavors {
        create("free") {
            dimension = "tier"
            applicationIdSuffix = ".free"
        }
        create("paid") {
            dimension = "tier"
            applicationIdSuffix = ".paid"
        }
    }

    // Вимкнення непотрібних комбінацій для економії часу збірки
    variantFilter {
        if (flavors.any { it.name == "free" } && buildType.name == "release") {
            ignore = true
        }
    }
}
```

### Генерація коду (BuildConfig)
Щоб передавати параметри збірки прямо у код (наприклад, різні URL серверів), увімкніть `buildConfig`:

```kotlin
android {
    buildFeatures { buildConfig = true }
}

buildTypes {
    debug {
        buildConfigField("String", "BASE_URL", "\"https://dev.api.example.com/\"")
        buildConfigField("Boolean", "IS_LOGGING_ENABLED", "true")
    }
}
```

### Мініфікація та Обфускація (R8 / ProGuard)
R8 — це компілятор, який оптимізує, мініфікує (видаляє невикористаний код) та обфускує (змінює імена класів/методів) код релізної збірки. Це суттєво зменшує розмір фінального APK і ускладнює реверс-інжиніринг. Зазвичай вмикається тільки для релізних збірок.
```kotlin
buildTypes {
    release {
        isMinifyEnabled = true // вмикає обфускацію коду
        isShrinkResources = true // видаляє невикористані ресурси (картинки, xml)
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
    }
}
```

---

## 6. Власні задачі (Custom Tasks) та Життєвий цикл

Задачі реєструються через об'єкт `tasks`. Існує дві фази: **Configuration** (виконується завжди під час ініціалізації) та **Execution** (блок `doLast` / `doFirst`, виконується тільки під час запуску задачі).

### Приклад простої задачі
```kotlin
tasks.register("printProjectInfo") {
    doLast {
        println("Модуль: ${project.name}")
        println("Шлях до збірки: ${project.layout.buildDirectory.get()}")
    }
}
// Запуск: ./gradlew printProjectInfo
```

### Задача копіювання (`Copy`) та зв'язування задач
Налаштування ланцюжків виконання (Task Dependency) дозволяє автоматизувати рутинні дії.
Наприклад, копіювання файлів обфускації (mapping.txt) після збірки релізу в окрему архівну папку:

```kotlin
val backupMappingFile by tasks.registering(Copy::class) {
    // Вказуємо звідки і куди копіювати
    from(layout.buildDirectory.dir("outputs/mapping/release"))
    into(layout.buildDirectory.dir("reports/mappings_archive"))
}

// Прив'язуємо нашу задачу до існуючої задачі життєвого циклу
tasks.named("assembleRelease") {
    finalizedBy(backupMappingFile) // Виконати задачу автоматично ПІСЛЯ assembleRelease
}
```
*(Можна також використовувати `dependsOn`, щоб ваша задача виконувалась ДО вказаної).*

---

## 7. Профілювання та Кешування

Для прискорення роботи Gradle, налаштовується файл **`gradle.properties`**:

* `org.gradle.jvmargs=-Xmx4096m -XX:+UseParallelGC` — Збільшує пам'ять до 4 ГБ та оптимізує збирання сміття.
* `org.gradle.parallel=true` — Дозволяє паралельно збирати незалежні модулі.
* `org.gradle.caching=true` — Вмикає Build Cache (кеш результатів задач).
* `org.gradle.configuration-cache=true` — Кешує фазу конфігурації.

**Аналіз часу збірки:**
Додайте прапорець `--profile`, щоб отримати детальний звіт:
```bash
./gradlew assembleDebug --profile
```
Звіт у форматі HTML буде згенеровано у папці `build/reports/profile/`.