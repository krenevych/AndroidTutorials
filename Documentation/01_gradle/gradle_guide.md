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

## 3. Конфігурація файлів build.gradle.kts (Кореневий та Модульний)

У Gradle проектах налаштування розділені на рівень усього проекту та рівень окремих модулів. 

### 3.1. Кореневий файл збірки (`build.gradle.kts` у корені проекту)
Цей файл (Top-level build file) відповідає за конфігурацію, спільну для всіх модулів у проекті. Зазвичай тут підключаються глобальні плагіни з використанням `apply false`. Це означає, що плагін лише реєструється в проекті (визначається його версія для всіх модулів), але безпосередньо не застосовується до кореневої директорії.

```kotlin
// Приклад кореневого build.gradle.kts
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

### 3.2. Конфігурація модуля (`app/build.gradle.kts`)

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
Залежності мають різні "зони видимості" (Scopes):

* **`implementation`**: Основна залежність; пакується в APK та доступна в коді.
* **`api`**: Пакується в збірку та транслюється іншим залежним модулям.
* **`compileOnly`**: Потрібна лише під час компіляції (не потрапляє в APK).
* **`testImplementation`**: Тільки для локальних тестів (`src/test/`).
* **`androidTestImplementation`**: Тільки для інструментальних тестів (`src/androidTest/`).

```kotlin
dependencies {
    implementation(libs.coil.kt)
    implementation(libs.bundles.network) // Підключення групи бібліотек

    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
}
```

> **Діагностика залежностей:**  
> Якщо виникає конфлікт версій, використовуйте:  
> `./gradlew :app:dependencies --configuration releaseRuntimeClasspath`

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

### Прив'язка до жипі циклу
Можна автоматизувати дії, наприклад, копіювання файлів після збірки релізу:
```kotlin
val backupMappingFile by tasks.registering(Copy::class) {
    from(layout.buildDirectory.dir("outputs/mapping/release"))
    into(layout.buildDirectory.dir("reports/mappings_archive"))
}

tasks.named("assembleRelease") {
    finalizedBy(backupMappingFile) // Виконати задачу автоматично ПІСЛЯ assembleRelease
}
```

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