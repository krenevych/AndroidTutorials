# Робота з Gradle в Android

Gradle — це потужна система автоматизації збірки, яка використовується за замовчуванням в Android проектах. Ця документація містить базові та поглиблені концепції для керування процесом збірки, налаштування модулів та залежностей.

---

## 1. Основи Gradle Wrapper (`gradlew`)

**Gradle Wrapper** (`gradlew`) — це скрипт, який дозволяє запускати задачі Gradle без необхідності попереднього встановлення Gradle у системі. Він автоматично завантажує необхідну версію Gradle, вказану в налаштуваннях проекту.

Всі команди виконуються в терміналі кореневої папки проекту:
- На macOS/Linux: `./gradlew <задача>`
- На Windows: `gradlew.bat <задача>`

### Базові команди
- `./gradlew tasks` — Виводить список усіх доступних задач для проекту.
- `./gradlew --version` — Показує поточну версію Gradle та інформацію про середовище (наприклад, версію Java).
- `./gradlew clean` — Очищає проект (видаляє каталог `build/`), що корисно перед створенням "чистої" збірки.
- `./gradlew assembleDebug` — Збирає проект і створює APK-файл для відлагодження (Debug). Зібраний APK зазвичай знаходиться за шляхом: `app/build/outputs/apk/debug/app-debug.apk`.
- `./gradlew assembleRelease` — Збирає оптимізований релізний APK.

---

## 2. Базова конфігурація модуля (`build.gradle.kts`)

Основні налаштування Android-додатку знаходяться у файлі модуля, зазвичай це `app/build.gradle.kts` (в блоці `android { ... }`).

### Параметри SDK та Версіонування
- **`compileSdk`** — Версія Android SDK, яка використовується для компіляції коду. (Наприклад, 34). Дозволяє використовувати нові API в коді.
- **`minSdk`** — Мінімальна версія Android, на якій може бути запущений додаток.
- **`targetSdk`** — Версія Android, під яку додаток був протестований та оптимізований.
- **`versionCode`** — Цілочисельне значення версії (наприклад, `2`). Збільшується з кожним новим релізом для Google Play.
- **`versionName`** — Рядкове представлення версії для користувачів (наприклад, `"1.0.1"`).

### Build Types (Типи збірок)
Блок `buildTypes` описує, як збирається додаток (наприклад, `debug` для розробки та `release` для публікації).

- **`applicationIdSuffix`** — Додає суфікс до `applicationId`. Це дозволяє встановити debug і release версії як окремі додатки на одному пристрої.
  ```kotlin
  buildTypes {
      debug {
          applicationIdSuffix = ".debug"
      }
  }
  ```

---

## 3. Управління залежностями та Version Catalog

Сучасний підхід до управління залежностями в Gradle — використання **Version Catalog** (файл `gradle/libs.versions.toml`). Це дозволяє централізовано керувати версіями бібліотек.

### Структура `libs.versions.toml`
- `[versions]` — зберігає версії.
- `[libraries]` — оголошує самі залежності (бібліотеки).
- `[plugins]` — підключені плагіни.
- `[bundles]` — дозволяє об'єднувати декілька бібліотек під одним іменем.

**Приклад:**
```toml
[versions]
coil = "2.6.0"
retrofit = "2.9.0"

[libraries]
coil = { module = "io.coil-kt:coil", version.ref = "coil" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }

[bundles]
network = ["retrofit"]
```

### Підключення залежностей у `build.gradle.kts`
Залежності додаються у блоці `dependencies`. Вони мають різну "зону видимості" (Scope):
- **`implementation`** — основна залежність; бібліотека потрапить у кінцевий APK та буде доступна в коді додатку.
- **`testImplementation`** — залежність тільки для локальних Unit-тестів (наприклад, JUnit). Вона **не потрапляє** у фінальний APK-файл.

Підключення (через Version Catalog):
```kotlin
dependencies {
    implementation(libs.coil)
    // Підключення бандлу (групи бібліотек):
    implementation(libs.bundles.network)
}
```

### Аналіз дерева залежностей
Іноді різні бібліотеки тягнуть різні версії однієї і тієї ж транзитивної залежності. Для діагностики:
- `./gradlew :app:dependencies --configuration releaseRuntimeClasspath` — будує повне дерево залежностей.
- `./gradlew :app:dependencyInsight --dependency <назва> --configuration releaseRuntimeClasspath` — детально показує, звідки підтягується конкретна бібліотека і як Gradle вирішує конфлікт версій (Resolution Strategy зазвичай обирає найновішу версію).

---

## 4. Конфігурація Build Variants та Product Flavors

**Product Flavors** дозволяють створювати різні версії додатку (наприклад, платна та безкоштовна) з одного кодового базису. Разом із `Build Types` вони утворюють **Build Variants** (наприклад, `freeDebug`, `paidRelease`).

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
}
```

### Фільтрація варіантів (`variantFilter`)
Якщо певна комбінація не має сенсу (наприклад, `freeRelease`), її можна вимкнути, щоб зекономити час збірки:
```kotlin
android {
    variantFilter {
        if (flavors.any { it.name == "free" } && buildType.name == "release") {
            ignore = true
        }
    }
}
```

---

## 5. Генерація коду (BuildConfig) та Обфускація (R8)

### BuildConfig
`BuildConfig` — це згенерований клас, що містить конфігураційні константи. Для використання необхідно увімкнути фічу:
```kotlin
android {
    buildFeatures {
        buildConfig = true
    }
}
```
Додавання власних полів (наприклад, різних `BASE_URL` для debug/release або Flavors):
```kotlin
buildTypes {
    debug {
        buildConfigField("String", "BASE_URL", "\"https://dev.api.example.com/\"")
        buildConfigField("Boolean", "IS_LOGGING_ENABLED", "true")
    }
}
```

### Мініфікація та Обфускація (R8 / ProGuard)
R8 — це компілятор, який оптимізує, мініфікує (видаляє невикористаний код) та обфускує (змінює імена класів/методів) код релізної збірки, зменшуючи розмір APK і ускладнюючи реверс-інжиніринг.
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

Ви можете розширювати можливості збірки власними задачами. Вони додаються у файл `build.gradle.kts`.

### Проста задача
```kotlin
tasks.register("printProjectInfo") {
    doLast {
        println("Модуль: ${project.name}")
        println("Шлях до збірки: ${project.layout.buildDirectory.get()}")
    }
}
```
*Запуск:* `./gradlew printProjectInfo`

### Задача копіювання (`Copy`) та зв'язування задач
Налаштування ланцюжків виконання (Task Dependency).
```kotlin
val backupMappingFile by tasks.registering(Copy::class) {
    // Вказуємо звідки і куди копіювати
    from(layout.buildDirectory.dir("outputs/mapping/release"))
    into(layout.buildDirectory.dir("reports/mappings_archive"))
}

// Прив'язуємо нашу задачу до життєвого циклу
tasks.named("assembleRelease") {
    finalizedBy(backupMappingFile) // Автоматично виконати після assembleRelease
}
```
*(Можна також використовувати `dependsOn`, щоб задача виконувалась ДО вказаної).*

---

## 7. Профілювання та Кешування збірки

Для пришвидшення компіляції використовуються налаштування у файлі `gradle.properties`:

- `org.gradle.jvmargs=-Xmx4096m -XX:+UseParallelGC` — виділяє більше оперативної пам'яті (4ГБ) для JVM-процесу Gradle та вказує збирач сміття.
- `org.gradle.parallel=true` — дозволяє паралельно збирати незалежні модулі.
- `org.gradle.caching=true` — вмикає кеш результатів виконання задач (Build Cache).
- `org.gradle.configuration-cache=true` — вмикає кешування фази конфігурації (Configuration Cache), суттєво пришвидшуючи наступні запуски.

### Профілювання (Profiling)
Щоб дізнатися, на що витрачається час під час компіляції, використовуйте прапорець `--profile`:
```bash
./gradlew assembleDebug --profile
```
Після виконання, детальний HTML-звіт профілювання (з інформацією про час конфігурації та виконання кожної задачі) буде згенеровано у папці `build/reports/profile/`.