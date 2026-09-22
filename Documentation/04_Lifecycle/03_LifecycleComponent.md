# Jetpack Lifecycle: LifecycleOwner, LifecycleObserver та клас Lifecycle

У попередніх уроках ми розбирали життєвий цикл `Activity` та `Application` через пряме перевизначення методів (`onCreate`, `onStart`, `onStop` тощо). Однак у реальних застосунках спроба керувати всіма фоновими процесами (GPS, плеєр, підключення до мережі, логування) безпосередньо всередині `Activity` призводить до появи величезної кількості спагеті-коду (так звані "Fat Activities").

Для вирішення цієї проблеми Google створив бібліотеку **Jetpack Lifecycle**, яка дозволяє винести логіку, залежну від життєвого циклу, в окремі класи.

## 1. Проблема традиційного підходу

Уявіть, що вашому екрану потрібно відстежувати геопозицію користувача (GPS). При традиційному підході ви б писали подібний код:

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var locationTracker: LocationTracker

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        locationTracker = LocationTracker(this)
    }

    override fun onStart() {
        super.onStart()
        locationTracker.startTracking() // Вмикаємо GPS
    }

    override fun onStop() {
        super.onStop()
        locationTracker.stopTracking() // Вимикаємо GPS
    }
}
```

**Чому це погано?**
1. **Порушення Single Responsibility Principle:** `MainActivity` змушена керувати роботою GPS, замість того щоб відповідати лише за відображення UI.
2. **Дублювання коду:** Якщо GPS потрібен на 5 різних екранах, вам доведеться копіювати виклики `startTracking()` та `stopTracking()` у всі 5 `Activity`.
3. **Ризик Race Conditions (помилок синхронізації):** Якщо `startTracking()` виконує асинхронну ініціалізацію, може статися ситуація, коли екран вже зупинився (`onStop`), а GPS тільки-но ввімкнувся і продовжує садити батарею.

---

## 2. Тріада Jetpack Lifecycle: Основи

Бібліотека Jetpack Lifecycle базується на трьох головних поняттях:

```
+------------------+         повертає Lifecycle         +-------------------+
|  LifecycleOwner  | ---------------------------------> |     Lifecycle     |
| (Activity/Fragment)|                                  | (States & Events) |
+------------------+                                    +-------------------+
                                                                  |
                                                                  | спостерігає за станом
                                                                  v
                                                        +-------------------+
                                                        | LifecycleObserver |
                                                        | (Ваш GPS/Плеєр)   |
                                                        +-------------------+
```

### 1. [`Lifecycle`](https://developer.android.com/reference/kotlin/androidx/lifecycle/Lifecycle) (Клас)
Це об'єкт, який містить інформацію про поточний стан компонента (наприклад, `Activity`) і дозволяє іншим класам підписуватися на зміну цього стану.

Він оперує двома основними поняттями:
* **Events (Події):** Події життєвого циклу (енум [`Lifecycle.Event`](https://developer.android.com/reference/kotlin/androidx/lifecycle/Lifecycle.Event)), які генеруються системою (`ON_CREATE`, `ON_START`, `ON_RESUME`, `ON_PAUSE`, `ON_STOP`, `ON_DESTROY`).
* **States (Стани):** Поточний стан компонента (енум [`Lifecycle.State`](https://developer.android.com/reference/kotlin/androidx/lifecycle/Lifecycle.State)), у якому перебуває об'єкт (`INITIALIZED`, `CREATED`, `STARTED`, `RESUMED`, `DESTROYED`).

### 2. [`LifecycleOwner`](https://developer.android.com/reference/kotlin/androidx/lifecycle/LifecycleOwner) (Інтерфейс)
Це об'єкт, який **має** життєвий цикл. Інтерфейс містить лише один метод: `getLifecycle(): Lifecycle`.
Усі сучасні базові класи Android (такі як `AppCompatActivity`, `ComponentActivity`, `Fragment`) вже імплементують інтерфейс `LifecycleOwner`.

### 3. [`LifecycleObserver`](https://developer.android.com/reference/kotlin/androidx/lifecycle/LifecycleObserver) та його нащадки [`DefaultLifecycleObserver`](https://developer.android.com/reference/kotlin/androidx/lifecycle/DefaultLifecycleObserver) і [`LifecycleEventObserver`](https://developer.android.com/reference/kotlin/androidx/lifecycle/LifecycleEventObserver)
Це об'єкти, які хочуть **спостерігати** за життєвим циклом іншого компонента.

Для створення власних спостерігачів в Android використовують два основних підходи:

* **`DefaultLifecycleObserver`** *(Рекомендований)* — надає окремі готові методи (`onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`). Ви перевизначаєте лише ті з них, які вам дійсно потрібні:

  ```kotlin
  class MyDefaultObserver : DefaultLifecycleObserver {
      override fun onStart(owner: LifecycleOwner) {
          Timber.d("Екран став видимим")
      }

      override fun onStop(owner: LifecycleOwner) {
          Timber.d("Екран заховався")
      }
  }
  ```

* **`LifecycleEventObserver`** — надає єдиний метод `onStateChanged(source: LifecycleOwner, event: Lifecycle.Event)`, який викликається при **будь-якій** події життєвого циклу. Всередині цього методу ви обробляєте потрібні події за допомогою конструкції `when (event)`:

  ```kotlin
  class MyEventObserver : LifecycleEventObserver {
      override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
          when (event) {
              Lifecycle.Event.ON_START -> Timber.d("Екран став видимим")
              Lifecycle.Event.ON_STOP -> Timber.d("Екран заховався")
              else -> {}
          }
      }
  }
  ```

### 4. Реєстрація та видалення спостерігачів: `addObserver` та `removeObserver`
Щоб зв'язати спостерігача з об'єктом, який має життєвий цикл (`LifecycleOwner`), використовуються спеціальні методи класу `Lifecycle`:

* **[`addObserver(observer)`](https://developer.android.com/reference/kotlin/androidx/lifecycle/Lifecycle#addobserver)** — підписує спостерігача на життєвий цикл. З цього моменту спостерігач почне отримувати всі наступні події (а також поточний стан):
  ```kotlin
  val myObserver = MyDefaultObserver()
  lifecycle.addObserver(myObserver)
  ```
* **[`removeObserver(observer)`](https://developer.android.com/reference/kotlin/androidx/lifecycle/Lifecycle#removeobserver)** — відписує спостерігача, якщо ви хочете припинити стеження за подіями раніше, ніж екран буде знищено:
  ```kotlin
  lifecycle.removeObserver(myObserver)
  ```

> 💡 **Важливо:** Якщо ви підписали обзервер на `Activity` чи `Fragment`, вам **не обов'язково** викликати `removeObserver` вручну під час закриття екрану. Коли `LifecycleOwner` досягає стану `DESTROYED`, система автоматично відписує всі зареєстровані спостерігачі, захищаючи вас від витоків пам'яті (Memory Leaks). Виклик `removeObserver` потрібен лише тоді, коли ви хочете зупинити спостереження достроково (наприклад, за певною умовою бізнес-логіки).

---

## 3. Практичний приклад: Створення Lifecycle-Aware компонента

Давайте перепишемо наш `LocationTracker` так, щоб він сам керував своїм життєвим циклом, роблячи `MainActivity` абсолютно чистою.

### Крок 1. Створюємо Observer
Створюємо клас `LocationTracker`, який реалізує `DefaultLifecycleObserver`:

```kotlin
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.LifecycleOwner
import timber.log.Timber

class LocationTracker : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        // Цей метод автоматично викличеться, коли Activity/Fragment перейде в стан STARTED
        startTracking()
    }

    override fun onStop(owner: LifecycleOwner) {
        // Цей метод автоматично викличеться, коли Activity/Fragment перейде в стан STOPPED
        stopTracking()
    }

    private fun startTracking() {
        Timber.d("GPS відстеження УВІМКНЕНО")
    }

    private fun stopTracking() {
        Timber.d("GPS відстеження ВИМКНЕНО")
    }
}
```

### Крок 2. Підписуємо Observer у MainActivity
Тепер у `MainActivity` нам достатньо лише створити об'єкт трекера і передати його в `lifecycle.addObserver()`:

```kotlin
class MainActivity : AppCompatActivity() {

    private val locationTracker = LocationTracker()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Додаємо наш спостерігач до життєвого циклу MainActivity
        lifecycle.addObserver(locationTracker)
    }
    
    // Більше НІЯКИХ перевизначень onStart() чи onStop() для GPS не потрібно!
}
```

**Що відбудеться під капотом?**
Коли `MainActivity` перейде у стан `onStart()`, система сама сповістить `locationTracker`, і він виконає свій метод `onStart()`. Коли екран згорнеться (`onStop()`), `locationTracker` автоматично зупинить GPS.

---

## 4. Перевірка поточного стану (Lifecycle.State)

Іноді всередині якогось класу вам потрібно перевірити, чи екран зараз видимий, перш ніж виконувати операцію (наприклад, показати діалогове вікно).

Ви можете перевірити поточний стан через властивість `lifecycle.currentState`:

```kotlin
if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
    // Екран як мінімум видимий (STARTED або RESUMED)
    // Безпечно оновлювати UI або показувати діалог
}
```

Метод `isAtLeast(State)` повертає `true`, якщо поточний стан є рівним або "вищим" у прапорцях життєвого циклу, ніж переданий стан.

---

## 5. Спостереження за кожною Activity з класу Application

Якщо вам потрібно централізовано відстежувати події створення, запуску чи знищення **кожної окремої Activity** у вашому застосунку, ви можете скористатися методом `registerActivityLifecycleCallbacks(...)` класу `Application`. Цей метод приймає об'єкт інтерфейсу [`Application.ActivityLifecycleCallbacks`](https://developer.android.com/reference/kotlin/android/app/Application.ActivityLifecycleCallbacks), який і містить усі необхідні колбеки для відстеження стану екранів.

Найкращою практикою є винесення цієї логіки в окремий клас:

**Крок 1. Створюємо клас-колбек:**
```kotlin
import android.app.Activity
import android.app.Application
import android.os.Bundle
import timber.log.Timber

class ActivityLoggerCallbacks : Application.ActivityLifecycleCallbacks {
    
    override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
        Timber.d("Створено Activity: ${activity.javaClass.simpleName}")
    }

    override fun onActivityStarted(activity: Activity) {}
    override fun onActivityResumed(activity: Activity) {}
    override fun onActivityPaused(activity: Activity) {}
    override fun onActivityStopped(activity: Activity) {}
    override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
    
    override fun onActivityDestroyed(activity: Activity) {
        Timber.d("Знищено Activity: ${activity.javaClass.simpleName}")
    }
}
```

**Крок 2. Реєструємо його у вашому `Application`:**
```kotlin
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        // ініціалізація Timber
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }

        // Передаємо екземпляр нашого кастомного класу
        registerActivityLifecycleCallbacks(ActivityLoggerCallbacks())
    }
}
```
* **Як це працює:** Метод `registerActivityLifecycleCallbacks` приймає об'єкт, що реалізує інтерфейс `Application.ActivityLifecycleCallbacks`. Системні колбеки викликаються для кожної Activity вашого застосунку, передаючи пряме посилання на екземпляр цієї `Activity`. Винесення логіки в окремий клас залишає `MyApplication` чистим і полегшує тестування.

---

## 6. Спостереження за життєвим циклом всього застосунку: ProcessLifecycleOwner

Іноді виникає потреба знати не про окремі Activity, а про стан усієї програми загалом: коли весь застосунок переходить на передній план (Foreground) чи згортається у фон (Background) — незалежно від того, скільки екранів відкриває користувач.

Для цього використовується спеціальний клас [`ProcessLifecycleOwner`](https://developer.android.com/reference/kotlin/androidx/lifecycle/ProcessLifecycleOwner).

1. Додайте залежність у `build.gradle.kts`:
   ```kotlin
   implementation("androidx.lifecycle:lifecycle-process:2.8.0")
   ```

2. **Створіть клас-спостерігач:**
   ```kotlin
   import androidx.lifecycle.DefaultLifecycleObserver
   import androidx.lifecycle.LifecycleOwner
   import timber.log.Timber

   class AppLifecycleObserver : DefaultLifecycleObserver {

       override fun onStart(owner: LifecycleOwner) {
           Timber.d("Застосунок перейшов у FOREGROUND (видимий)")
       }

       override fun onStop(owner: LifecycleOwner) {
           Timber.d("Застосунок перейшов у BACKGROUND (згорнутий)")
       }
   }
   ```

3. **Зареєструйте Observer у вашому класі `Application`:**
   ```kotlin
   class MyApplication : Application() {

       override fun onCreate() {
           super.onCreate()

           // ініціалізація Timber
           if (BuildConfig.DEBUG) {
               Timber.plant(Timber.DebugTree())
           }

           // Підписуємося на життєвий цикл всього ПРОЦЕСУ застосунку
           ProcessLifecycleOwner.get().lifecycle.addObserver(AppLifecycleObserver())
       }
   }
   ```
* **Як це працює:** `ProcessLifecycleOwner` розглядає весь процес програми як один великий `LifecycleOwner`. Він генерує події `ON_START`/`ON_RESUME`, коли відкривається перша Activity, і затримує виклик `ON_STOP`, щоб пересвідчитися, що користувач дійсно згорнув програму, а не просто перейшов між двома екранами.

---

## Підсумок

* **`LifecycleOwner`** — це компонент, **хто має** життєвий цикл (наприклад, `Activity` чи `Fragment`).
* **`LifecycleObserver`** — це клас, **хто реагує** на події життєвого циклу (використовуйте `DefaultLifecycleObserver` для окремих методів або `LifecycleEventObserver` для обробки всіх подій в одному місці).
* **`lifecycle.addObserver()` / `removeObserver()`** — підписують або відписують спостерігача. При досягненні стану `DESTROYED` система відписує спостерігачі автоматично для запобігання витокам пам'яті.
* **`lifecycle.currentState`** — дозволяє безпечно перевірити поточний стан екрану (наприклад, через `isAtLeast(Lifecycle.State.STARTED)`).
* **`registerActivityLifecycleCallbacks`** — дозволяє класу `Application` централізовано відстежувати створення та знищення кожної `Activity` у системі.
* **`ProcessLifecycleOwner`** — представляє життєвий цикл усього процесу програми та дозволяє легко визначати, коли застосунок у цілому виходить на передній план (`FOREGROUND`) чи ховається у фон (`BACKGROUND`).
* Застосування Jetpack Lifecycle робить ваш код модульним, безпечним та позбавляє від появи перевантажених "Fat Activities".