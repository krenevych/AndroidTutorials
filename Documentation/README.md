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
* [Корутини під капотом (`03_Coroutines_Under_The_Hood.md`)](06_Concurrency/03_Coroutines_Under_The_Hood.md) — механізм CPS, прихований параметр `Continuation`, Машина станів (State Machine), легковажність корутин та кооперативне скасування.
