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

### 3. [`LifecycleObserver`](https://developer.android.com/reference/kotlin/androidx/lifecycle/LifecycleObserver) та [`DefaultLifecycleObserver`](https://developer.android.com/reference/kotlin/androidx/lifecycle/DefaultLifecycleObserver) (Інтерфейси)
Це об'єкти, які хочуть **спостерігати** за життєвим циклом іншого компонента. На практиці для створення власних обзерверів рекомендується використовувати інтерфейс `DefaultLifecycleObserver`, який надає готові порожні методи для відстеження подій.

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

## Підсумок

* **`LifecycleOwner`** — це той, **хто має** життєвий цикл (наприклад, `Activity` чи `Fragment`).
* **`LifecycleObserver`** (`DefaultLifecycleObserver`) — це той, **хто реагує** на події життєвого циклу.
* **`lifecycle.addObserver()`** — зв'язує спостерігача з власником життєвого циклу.
* Використання цієї тріади робить ваш код модульним, запобігає витокам пам'яті (Memory Leaks) та позбавляє від перевантажених `Activity`.