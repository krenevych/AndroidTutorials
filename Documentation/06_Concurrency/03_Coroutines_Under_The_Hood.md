# Як корутини працюють під капотом: CPS, Continuation та State Machine

У попередніх уроках ми навчилися використовувати корутини у коді. Тепер розберемося, як компілятор Kotlin та JVM реалізують цю "магію" під капотом без створення важких системних потоків.

---

## 1. CPS (Continuation-Passing Style) та прихований параметр

Коли компілятор Kotlin бачить ключове слово **`suspend`**, він застосовує трансформацію під назвою **Continuation-Passing Style (CPS)**.

Компілятор бере вашу `suspend` функцію і під час компіляції в байткод **додає додатковий прихований параметр** у кінець списку аргументів: `completion: Continuation<T>`.

### Приклад трансформації компілятором:

**Код на Kotlin, який пишете ви:**
```kotlin
suspend fun loadTemperature(city: String): String
```

**Байткод/Java-еквівалент, який генерує компілятор:**
```java
// Ключове слово suspend зникає, а замість нього з'являється параметр Continuation
Object loadTemperature(String city, Continuation<String> completion)
```

### Що таке `Continuation`?
`Continuation` — це інтерфейс фреймворку Kotlin, який представляє собою **колбек-поінт** (збережену точку відновлення) корутини:

```kotlin
interface Continuation<in T> {
    val context: CoroutineContext
    fun resumeWith(result: Result<T>) // Відновлює виконання корутини з результати чи помилкою
}
```

---

## 2. State Machine (Машина станів)

Як функція пам'ятає, з якого саме рядка їй потрібно продовжити виконання після призупинення?

Для кожної `suspend` функції компілятор генерує анонімний клас **State Machine (Машину станів)**.

Усі точки призупинення (`suspendPoints` — виклики інших `suspend` функцій) розбиваються на мітки (`label = 0, 1, 2...`), а всі локальні змінні зберігаються у полях об'єкта `Continuation`.

### Схематичний принцип роботи Машини Станів:

```kotlin
// 1. Код, який пишете ви на Kotlin:
suspend fun loadData() {
    val city = loadCity()            // Точка призупинення 1 (label: 0 -> 1)
    val temp = loadTemperature(city) // Точка призупинення 2 (label: 1 -> 2)
    updateUI(city, temp)             // Завершення (label = 2)
}
```

**Сгенерована компілятором Машина станів (спрощена схема в байткоді):**
```kotlin
fun loadData(completion: Continuation<Any?>): Any? {
    // Створюємо або отримуємо існуючий об'єкт Машини станів (Continuation)
    val sm = completion as? LoadDataContinuation ?: object : ContinuationImpl(completion) {
        var label = 0
        var result: Any? = null
        var city: String? = null

        // Цей метод викликається автоматично при відновленні корутини (resumeWith)
        override fun invokeSuspend(result: Result<Any?>): Any? {
            this.result = result
            return loadData(this) // РЕКУРСИВНИЙ ВХІД у loadData з оновленим label!
        }
    }

    when (sm.label) {
        0 -> {
            sm.label = 1
            return loadCity(sm) // Запускаємо loadCity та передаємо sm як Continuation
        }
        1 -> {
            // Корутина відновилася з результатом loadCity()
            sm.city = sm.result as String
            sm.label = 2
            return loadTemperature(sm.city!!, sm) // Запускаємо loadTemperature
        }
        2 -> {
            // Корутина відновилася з результатом loadTemperature()
            val temp = sm.result as String
            updateUI(sm.city!!, temp)
            return Unit // Роботу повністю завершено!
        }
        else -> error("Invalid state")
    }
}
```

Завдяки цьому `suspend` функція не блокує потік:
1. Під час виклику `loadCity(sm)` вона зберігає `sm.label = 1` та повертає прапорець `COROUTINE_SUSPENDED`, миттєво звільняючи потік.
2. Коли асинхронне завантаження `loadCity()` завершується, воно викликає `sm.resumeWith(result)`, який всередині викликає `invokeSuspend()`.
3. `invokeSuspend()` повторно викликає `loadData(sm)` — заходить у підрозділ `when (sm.label == 1)` і продовжує виконання з наступного рядка!

---

## 3. Перетворення колбеків у `suspend` функції (`suspendCoroutine`)

У реальних проектах вам часто доводиться працювати із застарілим або стороннім кодом (Legacy Code), який працює на асинхронних колбеках (Callback).

Уявіть, що у вас є застаріла функція завантаження температури, яка приймає лямбду-колбек:
```kotlin
fun loadTemperature(city: String, onResult: (Int) -> Unit)
```

Щоб перетворити таку функцію з колбеком у сучасну послідовну `suspend` функцію, в Kotlin використовується функція **`suspendCoroutine`**.

Вона надає прямий доступ до об'єкта **`Continuation`**, дозволяючи "розморозити" та відновити корутину за допомогою методів:
* **`continuation.resume(value)`** — для повернення успішного результату;
* **`continuation.resumeWithException(error)`** — для повернення помилки.

### Приклад обгортання у `suspend` функцію:

```kotlin
// Огортаємо асинхронний виклик з колбеком у suspendCoroutine
suspend fun loadTemperatureSuspend(city: String): Int = suspendCoroutine { continuation ->
    
    // Викликаємо функцію з колбеком
    loadTemperature(city) { temp ->
        if (temp != null) {
            // Відновлюємо корутину та повертаємо значення температури
            continuation.resume(temp)
        } else {
            // Відновлюємо корутину з помилкою
            continuation.resumeWithException(Exception("Не вдалося завантажити температуру"))
        }
    }
}
```

### Виклики у коді:
Тепер у вашій `Activity` чи `ViewModel` цей асинхронний виклик перетворюється на звичайний зручний послідовний рядок коду без жодних вкладених лямбд:

```kotlin
lifecycleScope.launch {
    try {
        val temperature = loadTemperatureSuspend("Київ")
        binding.tvTemperatureValue.text = "$temperature°C"
    } catch (e: Exception) {
        binding.tvTemperatureValue.text = "Помилка: ${e.message}"
    }
}
```

---

## 4. Легковажність корутин (Lightweight Threads)

Чому кажуть, що корутини — це "легкі потоки"?

* **Системний `Thread` в ОС:** Це важкий об'єкт операційної системи Linux/Android. Для кожного `Thread` виділяється близько 1 МБ стек-пам'яті в RAM, а переключення між потоками (Context Switch) вимагає переривань на рівні ядра процесора. Створення 100 000 системних потоків миттєво вб'є застосунок з `OutOfMemoryError`.
* **Корутина:** Це **звичайний об'єкт Kotlin в оперативній пам'яті** (екземпляр Машини станів `ContinuationImpl`). Вона займає усього кілька сотень байт. 100 000 корутин можуть спокійно виконуватися на невеличкому пулі з 4-8 справжніх системних потоків!

---

> 💡 **Готовий розв'язок:**
> Повний робочий код прикладів з корутинами можна переглянути в репозиторії [krenevych/Concurrency](https://github.com/krenevych/Concurrency) на гілці **`coroutines_under_the_hood`** (модуль **`coroutine`**).

---

## Підсумок

* **CPS (Continuation-Passing Style)** — компілятор додає прихований параметр `Continuation<T>` у кожну `suspend` функцію.
* **Continuation** — об'єкт-колбек, який зберігає точку відновлення та локальні змінні корутини.
* **State Machine** — компілятор розбиває `suspend` функцію на мітки (`label`) і перемикає їх при відновленні виконання.
* **`suspendCoroutine`** — дозволяє зручно обгортати застарілі функції з асинхронними колбеками у `suspend` функції.
* **Легковажність** — корутини це об'єкти в RAM, тому мільйони корутин можуть працювати на жменьці системних потоків.
