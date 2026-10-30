# Зміст Документації

Тут зібрані всі навчальні матеріали (туторіали) по розробці під Android, поділені за темами.

## 01. Основи та Gradle
* [Основи Gradle (`gradle_guide.md`)](01_gradle/gradle_guide.md) — базове розуміння того, як збирається проект, що таке залежності та як влаштовані файли конфігурації.
* [AndroidManifest (`AndroidManifestTutorial.md`)](01_gradle/AndroidManifestTutorial.md) — детальний розбір головного конфігураційного файлу ("паспорту") застосунку, дозволів та оголошення компонентів.

## 02. ADB (Android Debug Bridge)
* [Команди ADB (`adb_documentation.md`)](02_adb/adb_documentation.md) — довідник команд для взаємодії з пристроєм чи емулятором через термінал (встановлення APK, логування, тощо).

## 03. Побудова інтерфейсу (View Approach)
* [Простий застосунок (`01_SimpleViewAppTutorial.md`)](03_ViewApproach/01_SimpleViewAppTutorial.md) — створення першого застосунку з XML-розміткою, базовими UI-елементами, обробкою натискань через `findViewById` та логуванням.
* [View Binding (`02_ViewBindingTutorial.md`)](03_ViewApproach/02_ViewBindingTutorial.md) — сучасний підхід до взаємодії з UI без використання `findViewById`.
* [Навігація (Intents) (`03_StartActivityTutorial.md`)](03_ViewApproach/03_StartActivityTutorial.md) — як відкривати нові екрани, передавати між ними дані та використовувати неявні інтенти (виклик браузера, пошти тощо) та як працюють Intent Filters.

## 04. Життєвий цикл (Lifecycle)
* [Application (`01_Application.md`)](04_Lifecycle/01_Application.md) — розбір того, як влаштовані процеси в Android, що таке Application Sandbox та навіщо створювати власний клас-спадкоємець `Application`.
* [Activity Lifecycle (`02_ActivityLifecycle.md`)](04_Lifecycle/02_ActivityLifecycle.md) — детальний розбір станів життєвого циклу екрану (від `onCreate` до `onDestroy`), візуальна схема та опис того, що відбувається під час повороту пристрою чи згортання застосунку.
* [Jetpack Lifecycle (`03_LifecycleComponent.md`)](04_Lifecycle/03_LifecycleComponent.md) — використання архитектурних компонентів `LifecycleOwner`, `LifecycleObserver` (`DefaultLifecycleObserver`) та класу `Lifecycle` для створення Lifecycle-Aware компонентів.
* [Context в Android (`04_Context.md`)](04_Lifecycle/04_Context.md) — що таке Context, різниця між `Activity Context` та `Application Context`, життєвий цикл контексту та як уникати витоків пам'яті (Memory Leaks).

## 05. Архітектура MVVM та Jetpack ViewModel
* [MVVM, ViewModel та LiveData (`01_ViewModel_and_MVVM.md`)](05_ViewModel_MVVM/01_ViewModel_and_MVVM.md) — розбір проблем із втратою стану при повороті екрану та витоками пам'яті, знайомство з MVVM, життєвим циклом `ViewModel`, `LiveData`, інкапсуляцією та різницею між `setValue()` і `postValue()`.

## 06. Багатопоточність (Concurrency)
* [Threads, Looper та Handler (`01_Threads_Looper_Handler.md`)](06_Concurrency/01_Threads_Looper_Handler.md) — основи багатопотоковості в Android, правила Main Thread, помилка `CalledFromWrongThreadException`, механіка `MessageQueue`, `Looper` та `Handler`, відкладене виконання через `postDelayed()`.
* [Вступ до Kotlin Coroutines (`02_Coroutines_Basics.md`)](06_Concurrency/02_Coroutines_Basics.md) — асинхронність у синхронному стилі, `suspend` функції, диспатчери (`Dispatchers.Main`, `Dispatchers.IO`, `Dispatchers.Default`), білдери корутин (`launch`, `async`/`await`, `withContext`).
* [Корутини під капотом (`03_Coroutines_Under_The_Hood.md`)](06_Concurrency/03_Coroutines_Under_The_Hood.md) — механізм CPS, прихований параметр `Continuation`, Машина станів (State Machine), легковажність корутин.
* [Скасування та Обробка помилок у Корутинах (`04_Coroutines_Error_Handling.md`)](06_Concurrency/04_Coroutines_Error_Handling.md) — кооперативне скасування (`isActive`, `ensureActive()`, `yield()`), `try-catch`, прокидання `CancellationException`, `SupervisorJob` та `supervisorScope`, `CoroutineExceptionHandler`.
* [Холодні реактивні потоки: Kotlin Flow (`05_Kotlin_Flow.md`)](06_Concurrency/05_Kotlin_Flow.md) — створення потоків через `flow { emit(...) }`, споживання через `collect`, популярні оператори (`map`, `filter`, `debounce`, `catch`, `flowOn`), збереження контексту.
* [Гарячі реактивні потоки: StateFlow та SharedFlow (`06_StateFlow_and_SharedFlow.md`)](06_Concurrency/06_StateFlow_and_SharedFlow.md) — гарячі потоки для збереження стану екрану, `StateFlow` як заміна `LiveData`, інкапсуляція у ViewModel, оператор `stateIn(WhileSubscribed(5000))`, сумісність з Compose та `repeatOnLifecycle`.

## 07. Jetpack Compose
* [Вступ до Jetpack Compose (`01_Compose_Basics.md`)](07_Compose/01_Compose_Basics.md) — декларативний UI, формулювання $\text{UI} = f(\text{State})$, анотація `@Composable`, макети (`Column`, `Row`, `Box`), ланцюжки модифікаторів `Modifier`, базові елементи (`Text`, `Button`, `Image`) та прев'ю `@Preview`.
* [Управління станом та Recomposition (`02_Compose_State_and_Recomposition.md`)](07_Compose/02_Compose_State_and_Recomposition.md) — поняття стану, рекомпозиція, `mutableStateOf`, `remember`, `rememberSaveable` (збереження стану при повороті), підйом стану (State Hoisting) та приклад Expandable Card.
* [Списки у Compose: LazyColumn та LazyRow (`03_Lazy_Lists.md`)](07_Compose/03_Lazy_Lists.md) — сучасна заміна `RecyclerView`, ледача генерація видимих елементів, синтаксис `items()` та `item()`, відступи через `contentPadding` та `spacedBy()`, ключ оптимізації `key`.
* [CompositionLocal та Context (`04_CompositionLocal_and_Context.md`)](07_Compose/04_CompositionLocal_and_Context.md) — механізм неявної передачі даних через дерево Compose, `LocalContext.current` та 4 правила безпечної роботи з Android Context всередині UI (уникнення Memory Leaks та параметризації).
* [Корутини та Побічні ефекти (`05_Coroutines_and_Effects.md`)](07_Compose/05_Coroutines_and_Effects.md) — правило Side-Effect Free, безпечний запуск корутин по кліку (`rememberCoroutineScope`), автоматичний запуск при відкритті екрану (`LaunchedEffect`) та механіка ключів перезапуску.

## 08. Системні компоненти Android
* [BroadcastReceiver та міжпроцесна взаємодія (`01_BroadcastReceiver.md`)](08_SystemComponents/01_BroadcastReceiver.md) — 4 компоненти Android, системні та кастомні бродкасти, статична реєстрація в `AndroidManifest.xml` та динамічна у коді, прапорці безпеки Android 14+ (`RECEIVER_NOT_EXPORTED`), перетворення BroadcastReceiver у `Flow` через `callbackFlow` та відображення баннера у Compose.
* [Служби (Services) (`02_Services.md`)](08_SystemComponents/02_Services.md) — відмінності `Service` від `Coroutines`, 3 типи служб, створення `ForegroundService` із постійним сповіщенням (`startForeground`) та дозволи Android 14+ (`foregroundServiceType`).
* [Гарантоване фонове виконання: WorkManager (`03_WorkManager.md`)](08_SystemComponents/03_WorkManager.md) — гарантовані відкладені задачі, `CoroutineWorker`, умови запуску (Constraints: Wi-Fi, зарядка), `Result.retry()` та підсумкова таблиця вибору (Coroutines vs Service vs WorkManager).

## 09. Робота з даними (Data & Networking)
* [Локальні налаштування: SharedPreferences та DataStore (`01_Preferences.md`)](09_Data/01_Preferences.md) — недоліки застарілого `SharedPreferences`, переваги `Preferences DataStore`, створення строго типізованих ключів, читання даних через `Flow`, асинхронний запис через `edit {}` та магічна реактивна інтеграція з `ViewModel` і Compose.
* [Робота з мережею: Retrofit та OkHttp (`02_Retrofit.md`)](09_Data/02_Retrofit.md) — архітектура мережевого стеку, дозволи `INTERNET`, `Kotlinx Serialization`, інтерфейси `@GET`/`@Path`, логування через `OkHttpClient`, використання `suspend` функцій та обробка станів екрану (`Sealed Interface: Loading, Success, Error`) у `ViewModel`.
