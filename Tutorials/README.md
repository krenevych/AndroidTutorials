# Зміст студентських завдань (Tutorials)

У цій директорії зберігаються всі практичні лабораторні роботи та завдання для студентів курсу Android 2026. Завдання розділені за відповідними теоретичними модулями.

---

## 01. Gradle
* [Завдання 1. Знайомство з Gradle та налаштування проекту (`task01.md`)](01_gradle/task01.md)
* [Завдання 2. Налаштування Flavor'ів та Build Types (`task02.md`)](01_gradle/task02.md)

---

## 02. ADB (Android Debug Bridge)
* [Завдання 1. Основи роботи з ADB (`task01.md`)](02_adb/task01.md)

---

## 03. Традиційний View Approach
* [Завдання 1. Базовий Калькулятор (`01_CalculatorTask.md`)](03_ViewApproach/01_CalculatorTask.md)
* [Завдання 2. Повноцінний Калькулятор зі збереженням стану (`02_CalculatorTaskFull.md`)](03_ViewApproach/02_CalculatorTaskFull.md)

---

## 04. Життєвий цикл (Lifecycle)
* [Завдання 1. Клас Application та глобальний стан (`01_ApplicationTask.md`)](04_Lifecycle/01_ApplicationTask.md)
* [Завдання 2. Дослідження життєвого циклу Activity (`02_ActivityLifecycleTask.md`)](04_Lifecycle/02_ActivityLifecycleTask.md)
* [Завдання 3. Створення Lifecycle-Aware Компонента (`03_LifecycleComponentTask.md`)](04_Lifecycle/03_LifecycleComponentTask.md)
* [Завдання 4. Детектор фонового режиму (`04_BackgroundDetectorTask.md`)](04_Lifecycle/04_BackgroundDetectorTask.md)
* [Завдання 5. Context та дослідження Memory Leaks (`05_ContextAndMemoryLeakTask.md`)](04_Lifecycle/05_ContextAndMemoryLeakTask.md)

---

## 05. ViewModel та архітектура MVVM
* [Завдання 1. Таймер на базі MVVM та LiveData (`01_TimerMVVMTask.md`)](05_ViewModel_MVVM/01_TimerMVVMTask.md)

---

## 06. Багатопоточність (Concurrency)
* [Симулятор кубиків на Threads (`01_DiceTask.md`)](06_Concurrency/01_DiceTask.md) — багатопотокова гральні кістки на `Thread`, `Handler`, `runOnUiThread` та MVVM.
* [Симулятор кубиків на Coroutines (`02_DiceCoroutinesTask.md`)](06_Concurrency/02_DiceCoroutinesTask.md) — асинхронне обертання кубиків на `suspend`, `delay`, `viewModelScope.launch`, `async` / `awaitAll`.
* [Завантажувач файлів на Flow (`04_1_FileDownloadFlowTask.md`)](06_Concurrency/04_1_FileDownloadFlowTask.md) — створення `Flow<Int>`, `emit()`, `flowOn(Dispatchers.IO)`, `onCompletion` та оновлення `ProgressBar`.
* [Живий пошук на операторах Flow (`04_2_LiveSearchFlowTask.md`)](06_Concurrency/04_2_LiveSearchFlowTask.md) — реалізація Search-as-you-type з використанням `debounce`, `filter`, `distinctUntilChanged`, `map`, `flowOn`.
* [Порівняння Flow та StateFlow (`05_StateFlowVsFlowTask.md`)](06_Concurrency/05_StateFlowVsFlowTask.md) — порівняльне дослідження генерації чисел Фібоначчі на холодній `Flow` та гарячій `StateFlow`, ефект повторних підписок та повертання екрану.

---

## 08. Системні компоненти Android
* [Журнал системних подій на BroadcastReceiver та Compose (`01_BroadcastReceiverComposeTask.md`)](08_SystemComponents/01_BroadcastReceiverComposeTask.md) — динамічне слухання системних бродкастів, `callbackFlow`, `ViewModel` + `StateFlow`, відображення у Compose `LazyColumn` із розгортаними картками `ExpandableCard`, видалення подій та симуляція через ADB.
* [Фоновий секундомір на Foreground Service (`02_1_ForegroundServiceTask.md`)](08_SystemComponents/02_1_ForegroundServiceTask.md) — створення безперервної фонової служби для типу `specialUse`, дозволи `FOREGROUND_SERVICE` та `POST_NOTIFICATIONS`, створення `NotificationChannel` та запуск `startForeground()`.
* [Гібридна служба та LocalBinder (`02_2_BoundServiceTask.md`)](08_SystemComponents/02_2_BoundServiceTask.md) — перетворення таймера на Bound Service, створення `LocalBinder` для підключення `Activity` через `ServiceConnection` (`onStart`/`onStop`), експонування `StateFlow` зі служби у Compose UI.
