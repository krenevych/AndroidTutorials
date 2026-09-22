# Завдання: Створення BackgroundDetector

## Мета завдання

Навчитися відстежувати загальний стан застосунку (чи він активний на екрані, чи згорнутий у фоні),
незважаючи на те, яка саме Activity зараз відкрита.
Для цього ви створите клас-утиліту `BackgroundDetector`, ініціалізуєте його у вашому `Application`
класі та навчитеся взаємодіяти з ним з будь-якого екрану.

## Навіщо це потрібно?

Коли у вас одна Activity, легко зрозуміти, коли програму згорнули (спрацьовує `onStop`). Але якщо у
вас 5 екранів, перехід між ними також викликає `onStop` для старого екрану. Як зрозуміти: користувач
просто перейшов на інший екран всередині вашої програми, чи він повністю згорнув ваш застосунок (
натиснув кнопку Home)?

Для вирішення цієї проблеми ми створимо спеціальний клас, який буде мати спільний (глобальний)
лічильник запущених Activity. Кожна наша Activity буде "відмічатися" у цього детектора при старті (
`onStart`) та при зупинці (`onStop`).

**Чому це працює? Секрет у послідовності життєвого циклу!**
Згадайте теорію: коли ви переходите з `MainActivity` на `SecondActivity`, система не зупиняє першу
миттєво. Натомість, послідовність викликів виглядає так:

1. `MainActivity.onPause()`
2. `SecondActivity.onStart()` *(наш лічильник стає = 2)*
3. `SecondActivity.onResume()`
4. `MainActivity.onStop()` *(наш лічильник стає = 1)*

Як бачите, `onStart` другої Activity викликається **раніше**, ніж `onStop` першої! Завдяки цьому наш
лічильник активних екранів ніколи не впаде до `0` під час переходів між екранами. Він стане нулем
виключно тоді, коли користувач дійсно згорне весь застосунок.

## Частина 1: Створення BackgroundDetector

1. **Створення класу:**
   У вашому проекті створіть новий Kotlin-клас з назвою `BackgroundDetector`.
   ```kotlin
   class BackgroundDetector {
       // Тут буде наш код
   }
   ```

2. **Логіка підрахунку:**
    * Створіть всередині класу лічильник:
      ```kotlin
      private var activeActivitiesCount = 0
      ```

    * Додайте функцію:
      ```kotlin
      fun onActivityStarted()
      ```
      У ній збільшуйте лічильник `activeActivitiesCount` на `1`.
    * Додайте функцію:
      ```kotlin
      fun onActivityStopped()
      ```
      У ній зменшуйте лічильник `activeActivitiesCount` на `1`.
3. **Логіка видимості (Foreground):**
    * Якщо після збільшення лічильник став дорівнювати `1`, це
      означає, що застосунок щойно вийшов із фону на передній план.
    * Виведіть лог: `Застосунок тепер ВИДИМИЙ на екрані (Foreground)`.
4. **Логіка невидимості (Background):**
    * Якщо після зменшення лічильника (в `onActivityStopped`) його
      значення стало дорівнювати `0`, значить жодна Activity більше не видима на екрані.
    * Виведіть лог: `Застосунок тепер у ФОНІ (Background)`.

## Частина 2: Ініціалізація в Application

Оскільки наш детектор не є Singleton-об'єктом, нам потрібно створити його єдиний екземпляр (
instance) і зберігати там, де він переживе зміну екранів — у класі `Application`.

1. Відкрийте ваш кастомний клас `MyApp` (який ви створили у попередньому завданні).
2. Створіть публічну властивість для детектора:
   ```kotlin
   val backgroundDetector = BackgroundDetector()
   ```

*(Тут нам навіть не потрібно нічого робити в `onCreate()`, оскільки об'єкт детектора створиться
автоматично разом із `MyApp`).*

## Частина 3: Реєстрація зсередини Activity

Щоб наш детектор працював, кожна Activity у нашому застосунку повинна самостійно повідомляти його
про свої зміни станів.

1. Відкрийте `MainActivity`.
2. Отримайте доступ до детектора через Application та викличте його метод в `onStart()`:
   ```kotlin
   override fun onStart() {
       super.onStart()
       val detector = (application as MyApp).backgroundDetector
       detector.onActivityStarted()
   }
   ```
3. Зробіть те саме в методі `onStop()`:
   ```kotlin
   override fun onStop() {
       super.onStop()
       val detector = (application as MyApp).backgroundDetector
       detector.onActivityStopped()
   }
   ```

## Частина 4: Тестування та аналіз логів

1. Створіть у проекті **другу Activity** (`SecondActivity`).
2. **Важливо:** Не забудьте також додати виклики `detector.onActivityStarted()` та
   `detector.onActivityStopped()` у відповідні методи `SecondActivity`!
3. Зробіть так, щоб з `MainActivity` можна було перейти на `SecondActivity`.
4. Запустіть програму та відкрийте **Logcat**.
5. Проробіть наступні кроки і проаналізуйте логи вашого `BackgroundDetector`:
    * **Запуск:** відкрилася `MainActivity` (лічильник = 1). Ви маєте побачити лог *"Застосунок тепер ВИДИМИЙ на екрані (Foreground)"*.
    * **Перехід:** ви натиснули кнопку, щоб відкрити `SecondActivity`. Подивіться в логи: чи виводиться повідомлення про перехід у фон? Його не має бути! (Лічильник стає `2`, потім `1`).
    * **Згортання:** натисніть кнопку Home на телефоні. Лічильник має впасти до нуля, і ви повинні побачити лог *"Застосунок тепер у ФОНІ (Background)"*.
    * **Відновлення:** відкрийте застосунок знову з меню недавніх. Лічильник має знову стати `1` і вивести лог *"Застосунок тепер ВИДИМИЙ на екрані (Foreground)"*.

## Частина 5: Автоматизація за допомогою registerActivityLifecycleCallbacks

У Частинах 1–4 ми прописували виклики детектора вручну у кожній Activity. Якщо в програмі 20 екранів, копіювати цей код у кожен з них — погана практика. 

Фреймворк Android має вбудований механізм `Application.ActivityLifecycleCallbacks`, який дозволяє класу `Application` автоматично слухати старт та зупинку **будь-якої Activity** у застосунку!

1. **Змініть `BackgroundDetector`:**
   Зробіть так, щоб `BackgroundDetector` реалізовував інтерфейс `Application.ActivityLifecycleCallbacks`:
   ```kotlin
   import android.app.Activity
   import android.app.Application
   import android.os.Bundle
   import timber.log.Timber // Переконайтеся, що Timber ініціалізовано у вашому MyApp

   class BackgroundDetector : Application.ActivityLifecycleCallbacks {
       private var activeActivitiesCount = 0

       override fun onActivityStarted(activity: Activity) {
           activeActivitiesCount++
           if (activeActivitiesCount == 1) {
               Timber.d("Застосунок тепер ВИДИМИЙ на екрані (Foreground)")
           }
       }

       override fun onActivityStopped(activity: Activity) {
           activeActivitiesCount--
           if (activeActivitiesCount == 0) {
               Timber.d("Застосунок тепер у ФОНІ (Background)")
           }
       }

       // Порожні реалізації для інших методів інтерфейсу:
       override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {}
       override fun onActivityResumed(activity: Activity) {}
       override fun onActivityPaused(activity: Activity) {}
       override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
       override fun onActivityDestroyed(activity: Activity) {}
   }
   ```

2. **Зареєструйте в `Application`:**
   У вашому класі `MyApp` у методі `onCreate()` додайте реєстрацію детектора:
   ```kotlin
   override fun onCreate() {
       super.onCreate()
   
       // ініціалізація Timber
   
       registerActivityLifecycleCallbacks(backgroundDetector)
   }
   ```

3. **Очистіть Activity:**
   Повністю видаліть виклики `detector.onActivityStarted()` та `detector.onActivityStopped()` з `MainActivity` та `SecondActivity`. 

4. **Протестуйте:**
   Запустіть застосунок та перевірте в Logcat, що логіка лічильника та виводу повідомлень про видимість/фон працює точно так само, хоча класи Activity тепер взагалі не містять коду детектора!

## Частина 6: Рефакторинг за допомогою ProcessLifecycleOwner та DefaultLifecycleObserver

У Частині 5 ми позбулися коду в Activity, але змушені були тримати лічильник `activeActivitiesCount` та купу порожніх методів `ActivityLifecycleCallbacks`. 

Якщо нам потрібен тільки статус всього застосунку (Foreground/Background), у Jetpack є ще досконаліший інструмент — `ProcessLifecycleOwner`.

1. **Додайте залежність:**
   У файл `build.gradle.kts` вашого модуля додайте бібліотеку:
   ```kotlin
   implementation("androidx.lifecycle:lifecycle-process:2.8.0")
   ```

2. **Модифікуйте `BackgroundDetector`:**
   Змініть клас `BackgroundDetector` так, щоб він реалізовував інтерфейс `DefaultLifecycleObserver`:
   ```kotlin
   import androidx.lifecycle.DefaultLifecycleObserver
   import androidx.lifecycle.LifecycleOwner
   import timber.log.Timber // Переконайтеся, що Timber ініціалізовано у вашому MyApp

   class BackgroundDetector : DefaultLifecycleObserver {
       override fun onStart(owner: LifecycleOwner) {
           Timber.d("Застосунок перейшов у FOREGROUND (видимий на екрані)")
       }

       override fun onStop(owner: LifecycleOwner) {
           Timber.d("Застосунок перейшов у BACKGROUND (згорнутий у фон)")
       }
   }
   ```
   *(Зверніть увагу: більше немає ні лічильника, ні порожніх методів! `ProcessLifecycleOwner` сам відстежує життєвий цикл процесу під капотом).*

3. **Зареєструйте в `Application`:**
   Змініть реєстрацію у вашому класі `MyApp`:
   ```kotlin
   override fun onCreate() {
       super.onCreate()
       
       // ініціалізація Timber
   
       ProcessLifecycleOwner.get().lifecycle.addObserver(BackgroundDetector())
   }
   ```

4. **Повторіть тестування з Частини 4:**
   Запустіть програму та перевірте Logcat. 
   * Подивіться, як красиво і без жодного зайвого рядка коду вирішується ця задача за допомогою Jetpack Lifecycle!

Успіхів! Це дуже розповсюджений патерн, який використовується в реальних додатках (наприклад, для того, щоб показати екран введення пін-коду, коли користувач повертається в банківський застосунок після його згортання).