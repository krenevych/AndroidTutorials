# Практикум: Поглиблена автоматизація збірки в Gradle

**Тема:** Аналіз графа залежностей, оптимізація компіляції, конфігурація Build Variants (Product
Flavors), генерація коду через `BuildConfig`, обфускація (R8/ProGuard) та інтеграція Custom Tasks у
пайплайн збірки.

---

### Мета роботи

1. Навчитися діагностувати та розв'язувати конфлікти транзитивних залежностей.
2. Побудувати матрицю варіантів збірки (Build Variants) для розділення продукту за функціоналом і
   середовищами (Dev/Prod).
3. Налаштувати R8/ProGuard для релізної збірки та налагодити автоматичну генерацію конфігураційних
   констант.
4. Оптимізувати продуктивність збірки за допомогою кешування конфігурації.

---

## Блок 1. Глибокий аналіз залежностей та бандли

1. **Діагностика транзитивних залежностей:**
    * За допомогою команди `./gradlew :app:dependencies --configuration releaseRuntimeClasspath`
      згенеруйте повне дерево бібліотек.
    * Використовуючи задачу `dependencyInsight`, перевірте, які компоненти підтягують бібліотеку
      `androidx.core:core-ktx`, та поясніть правила вирішення конфлікту версій (Resolution
      Strategy).
2. **Групування бібліотек у бандли (Bundles):**
    * У `gradle/libs.versions.toml` налаштуйте бандл для мережевого стеку, об'єднавши Retrofit та
      Kotlinx Serialization:
      ```toml
      [bundles]
      network = ["retrofit", "retrofit-converter-kotlinx-serialization"]
      ```
    * Підключіть увесь бандл одним рядком у `app/build.gradle.kts`.

---

## Блок 2. Генерація змінних середовища та конфігурація R8

1. **Генерація класу `BuildConfig`:**
    * Увімкніть підтримку через `buildFeatures.buildConfig = true`.
    * Додайте динамічні поля `BASE_URL`:
        * для `debug`: `"https://dev.api.example.com/"`
        * для `release`: `"https://api.example.com/"`
    * Додайте прапорець `IS_LOGGING_ENABLED` (значення `true` для `debug`, `false` для `release`).
2. **Оптимізація та мініфікація коду (R8):**
    * У блоці `buildTypes { release { ... } }`:
        * увімкніть мініфікацію коду (`isMinifyEnabled = true`);
        * увімкніть видалення невикористаних ресурсів (`isShrinkResources = true`);
        * підключіть стандартний файл правил оптимізації
          `getDefaultProguardFile("proguard-android-optimize.txt")`.
    * Зберіть релізний APK (`./gradlew assembleRelease`) та порівняйте його розмір із файлом
      `app-debug.apk`.

---

## Блок 3. Мульти-конфігурація: Product Flavors та фільтрація варіантів

Створіть розділення додатку на безкоштовну версію з рекламою та платну версію без реклами.

1. **Оголошення вимірів та флейворів:**
   ```kotlin
   flavorDimensions += "tier"
   productFlavors {
       create("free") {
           dimension = "tier"
           applicationIdSuffix = ".free"
           buildConfigField("Boolean", "SHOW_ADS", "true")
       }
       create("paid") {
           dimension = "tier"
           applicationIdSuffix = ".paid"
           buildConfigField("Boolean", "SHOW_ADS", "false")
       }
   }
   ```
2. **Фільтрація зайвих варіантів (Variant Filter):**
    * За допомогою блоку `variantFilter` вимкніть побудову варіанту `freeRelease` (безкоштовна
      версія має компілюватися лише для внутрішнього тестування у `debug`).

---

## Блок 4. Інтеграція задач у життєвий цикл (Task Lifecycle)

1. **Створення задачі типу `Copy`:**
    * Створіть задачу `backupMappingFile`, яка автоматично копіює мапінг-файл обфускації з папки
      `build/outputs/mapping/` до окремого каталогу `build/reports/mappings_archive/`.
2. **Зв'язування задач через `dependsOn` / `finalizedBy`:**
    * Налаштуйте ланцюжок так, щоб задача збереження мапінгу (`backupMappingFile`) автоматично
      виконувалася щоразу після завершення релізної збірки (`assembleRelease`).

---

## Блок 5. Профілювання та кешування збірки

1. **Налаштування параметрів компілятора (`gradle.properties`):**
    * Додайте оптимізаційні прапорці:
      ```properties
      org.gradle.jvmargs=-Xmx4096m -XX:+UseParallelGC
      org.gradle.parallel=true
      org.gradle.caching=true
      org.gradle.configuration-cache=true
      ```
2. **Аналіз часу збірки:**
    * Запустіть компіляцію з генерацією звіту профілювання: `./gradlew assembleDebug --profile`.
    * Знайдіть згенерований звіт у `build/reports/profile/` та визначте задачу, яка витратила
      найбільше часу на конфігурацію і виконання.

---

