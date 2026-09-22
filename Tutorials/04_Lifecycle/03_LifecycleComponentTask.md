# Завдання: Компоненти Jetpack Lifecycle (LifecycleOwner та LifecycleObserver)

## Мета завдання
Навчитися виносити логіку, залежну від життєвого циклу, в окремі класи за допомогою Jetpack Lifecycle. Ви створите власний `LifecycleObserver` за допомогою двох різних підходів (`DefaultLifecycleObserver` та `LifecycleEventObserver`), навчитеся перевіряти поточний стан через `lifecycle.currentState` та реєструвати централізоване відстеження всіх Activity у класі `Application`.

*(Зауваження: Тему `ProcessLifecycleOwner` у цьому завданні ми **не використовуємо** — її буде розглянуто в наступній вправі).*

---

## Частина 1: Створення компонента з `DefaultLifecycleObserver`

1. **Створення компонента `AudioPlayer`:**
   * Створіть клас `AudioPlayer`, який реалізує інтерфейс `DefaultLifecycleObserver`.
   * **Поведінка:**
     * При настанні події `ON_START` (перевизначте метод `onStart`) виводьте лог через Timber: *"AudioPlayer: Відтворення фонового аудіо відновлено"*.
     * При настанні події `ON_STOP` (перевизначте метод `onStop`) виводьте лог через Timber: *"AudioPlayer: Відтворення фонового аудіо поставлено на паузу"*.

2. **Підключення до MainActivity:**
   * Відкрийте `MainActivity`.
   * Створіть екземпляр `AudioPlayer`.
   * У методі `onCreate()` підпишіть плеєр на життєвий цикл вашої Activity за допомогою `lifecycle.addObserver(...)`.
   * Переконайтеся, що у самій `MainActivity` немає жодних ручних викликів чи перевизначень `onStart()`/`onStop()` для плеєра.

3. **Перевірка у Logcat:**
   * Запустіть програму. Згорніть її кнопкою Home і знову відкрийте.
   * Переконайтеся в панелі **Logcat**, що `AudioPlayer` автоматично реагує на згортання та розгортання екрану.

---

## Частина 2: Використання `LifecycleEventObserver`

Тепер реалізуємо другий спостерігач, який використовує альтернативний інтерфейс з єдиним методом для всіх подій.

1. **Створення обзервера `AnalyticsTracker`:**
   * Створіть клас `AnalyticsTracker`, який реалізує інтерфейс `LifecycleEventObserver`.
   * **Поведінка:**
     * Реалізуйте єдиний метод `onStateChanged(source: LifecycleOwner, event: Lifecycle.Event)`.
     * За допомогою конструкції `when (event)` обробіть такі події:
       * `ON_RESUME` — виводьте лог через Timber: *"AnalyticsTracker: Користувач взаємодіє з екраном (ON_RESUME)"*.
       * `ON_PAUSE` — виводьте лог через Timber: *"AnalyticsTracker: Екран втратив фокус (ON_PAUSE)"*.
       * Для решти подій не робіть нічого (`else -> {}`).

2. **Підключення до MainActivity:**
   * У методі `onCreate()` вашої `MainActivity` підпишіть і цей обзервер за допомогою `lifecycle.addObserver(...)`.

3. **Перевірка у Logcat:**
   * Переконайтеся, що обидва обзервери (`AudioPlayer` та `AnalyticsTracker`) працюють паралельно, отримують свої події незалежно один від одного і не засмічують код `MainActivity`.

---

## Частина 3: Відстеження Activity та її стану (Lifecycle.State) через Application

1. **Реєстрація у класі `Application`:**
   * Відкрийте ваш кастомний клас `MyApp` (унаслідуваний від `Application`).
   * У методі `onCreate()` переконайтеся, що Timber ініціалізовано для Debug-збірки.
   * Викличте системний метод `registerActivityLifecycleCallbacks(...)`, передавши анонімний об'єкт, що реалізує інтерфейс `Application.ActivityLifecycleCallbacks`.

2. **Логіка відстеження та виведення стану:**
   * У кожному колбеку інтерфейсу (`onActivityCreated`, `onActivityStarted`, `onActivityResumed`, `onActivityPaused`, `onActivityStopped`, `onActivityDestroyed`) виводьте лог через Timber.
   * У лог-повідомленні вкажіть:
     * Назву події / методу (наприклад, `onActivityCreated`).
     * Назву екрану (`activity.javaClass.simpleName`).
     * **Поточний стан життєвого циклу цієї Activity** через властивість `activity.lifecycle.currentState`.

3. **Перевірка у Logcat:**
   * Запустіть застосунок, відкрийте другу Activity (`SecondActivity`) та закрийте її.
   * Подивіться в Logcat, в якому саме стані (`CREATED`, `STARTED`, `RESUMED` тощо) знаходиться `activity.lifecycle.currentState` під час виклику кожного системного колбеку.

Успіхів! Ви опанували сучасний підхід до управління життєвим циклом в Android.