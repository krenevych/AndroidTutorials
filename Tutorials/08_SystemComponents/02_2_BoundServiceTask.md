# Завдання: Взаємодія з фоном через Bound Service

## Мета завдання
Зрозуміти концепцію **Прив'язаної служби (`Bound Service`)**, створити `LocalBinder` для двосторонньої взаємодії між `Activity` та `Service`, а також навчитися викликати публічні методи служби безпосередньо з Jetpack Compose-екрану.

---

## Вимоги до функціоналу застосунку

У попередньому завданні (Foreground Service) таймер просто логував свої тіки у Logcat. Це нормально, коли користувач не бачить додаток. Але коли користувач відкриває додаток, він хоче **бачити ці секунди на екрані**!

Ваше завдання — перетворити наш попередній сервіс-таймер на гібридний (**Foreground + Bound Service**), щоб `Activity` могла підключатися до нього і зчитувати поточний відлік.

### 1. Зміни у Службі
* Служба має зберігати поточну кількість відрахованих секунд у `StateFlow<Int>`.
* Служба повинна реалізувати та повертати `LocalBinder`, який даватиме доступ до екземпляра служби.
* Служба повинна надавати публічний метод `getTimerFlow(): StateFlow<Int>`.

### 2. Зміни у UI
* На екрані (Compose) має з'явитися великий `Text`, який показує поточні секунди таймера.
* У `Activity` в `onStart()` ви повинні викликати `bindService()`, а в `onStop()` — `unbindService()`.
* Після успішного `onServiceConnected`, `Activity` передає потік секунд із Сервісу у свій Compose UI, де він відображається через `collectAsStateWithLifecycle()`.

---

## Етапи виконання

### Крок 1. Створення LocalBinder
1. Відкрийте ваш `TimerForegroundService` з попереднього завдання.
2. Додайте змінну `private val _timerSeconds = MutableStateFlow(0)`. (Не забудьте збільшувати її у вашому циклі з `delay`).
3. Додайте публічний метод `fun getTimerFlow(): StateFlow<Int> = _timerSeconds.asStateFlow()`.
4. Створіть внутрішній клас `LocalBinder`:
```kotlin
inner class LocalBinder : Binder() {
    fun getService(): TimerForegroundService = this@TimerForegroundService
}
```
5. У методі `onBind()` поверніть створений біндер замість `null`:
```kotlin
private val binder = LocalBinder()

override fun onBind(intent: Intent?): IBinder {
    return binder
}
```

### Крок 2. Реалізація ServiceConnection у Activity
1. У `MainActivity` створіть змінну `timerFlow` (наприклад, типу `MutableStateFlow<Int>`), яка прокидатиме дані в Compose.
2. Створіть об'єкт `ServiceConnection`:
```kotlin
private var isBound = false

private val connection = object : ServiceConnection {
    override fun onServiceConnected(className: ComponentName, service: IBinder) {
        val binder = service as TimerForegroundService.LocalBinder
        val myService = binder.getService()
        isBound = true
        
        // Підключаємо потік з сервісу до нашої Activity!
        lifecycleScope.launch {
            myService.getTimerFlow().collect { seconds ->
                // Оновлюємо StateFlow для Compose
            }
        }
    }

    override fun onServiceDisconnected(arg0: ComponentName) {
        isBound = false
    }
}
```

### Крок 3. Життєвий цикл прив'язки
В `Activity` перевизначте методи `onStart` та `onStop`:
```kotlin
override fun onStart() {
    super.onStart()
    val intent = Intent(this, TimerForegroundService::class.java)
    // Зверніть увагу на BIND_AUTO_CREATE! 
    bindService(intent, connection, Context.BIND_AUTO_CREATE)
}

override fun onStop() {
    super.onStop()
    if (isBound) {
        unbindService(connection)
        isBound = false
    }
}
```

### Крок 4. Тестування гібридної служби
1. Запустіть додаток.
2. Натисніть кнопку "Старт Таймер" (Служба запуститься як Foreground і з'явиться сповіщення).
3. Переконайтеся, що на екрані `Activity` цифра збільшується кожну секунду (ви працюєте з Bound Service).
4. **Згорніть додаток.** Відкрийте Logcat і переконайтеся, що таймер продовжує йти у фоні (працює Foreground Service).
5. **Розгорніть додаток.** Зверніть увагу: UI миттєво підхопив актуальну цифру (наприклад, 14 секунд), оскільки `Activity` заново прив'язалася до служби через `onStart()`!

Успіхів! Ви створили професійну гібридну службу, яка не вмирає у фоні, але тісно комунікує з UI, коли користувач повертається!
