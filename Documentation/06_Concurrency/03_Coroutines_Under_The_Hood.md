# Як корутини працюють під капотом: CPS, Continuation та State Machine

У попередніх уроках ми навчилися використовувати корутини у коді. Тепер розберемося, як компілятор Kotlin та JVM реалізують цю "магію" під капотом без створення важких системних потоків.

> 📦 **Початковий проект:**
> Ви можете завантажити або склонувати заготовку цього проекту з GitHub за посиланням: [krenevych/Concurrency](https://github.com/krenevych/Concurrency) (гілка **`coroutine_under_the_hood_start`**, модуль **`coroutine`**).


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

### Наочна реалізація Машини Станів у коді

Давайте подивимося, як концептуально влаштована Машина станів та клас `Continuation` на реальному прикладі з проекту:

#### 1. Клас `LoadDataContinuation` (Збереження стану)
Створюється спеціалізований клас, який реалізує `Continuation<Unit>`, зберігає номер поточного кроку (`label`) та поля результатів між запускними етапами:

```kotlin
class LoadDataContinuation(
    private val activity: MainActivity,
    override val context: CoroutineContext = EmptyCoroutineContext
) : Continuation<Unit> {

    // Поточний стан стейт-машини
    var label: Int = 0

    // Поля для збереження результатів між кроками
    var city: String = ""
    var temperature: Int = 0

    override fun resumeWith(result: Result<Unit>) {
        if (result.isFailure) return
        
        // При кожному відновленні викликаємо метод стейт-машини
        activity.loadData(this)
    }
}
```

#### 2. Функція `loadData` (Стейт-машина)
Залежно від значення `label`, виконується відповідний крок асинхронного ланцюжка:

```kotlin
fun loadData(completion: LoadDataContinuation) {
    when (completion.label) {
        0 -> {
            // КРОК 0: Початок — готуємо UI, переводимо label в 1
            completion.label = 1

            binding.btnLoadData.isEnabled = false
            binding.progressBar.visibility = View.VISIBLE
            binding.tvCityValue.text = ""
            binding.tvTemperatureValue.text = ""

            // Запускаємо фонову задачу завантаження міста
            thread {
                loadCity(completion)
            }
        }

        1 -> {
            // КРОК 1: Відновлення — місто вже збережено в completion.city
            binding.tvCityValue.text = completion.city
            completion.label = 2

            // Запускаємо фонову задачу завантаження температури
            thread {
                loadTemperature(completion)
            }
        }

        2 -> {
            // КРОК 2: Відновлення — температура збережена в completion.temperature
            binding.tvTemperatureValue.text = completion.temperature.toString()
            binding.progressBar.visibility = View.GONE
            binding.btnLoadData.isEnabled = true
        }
    }
}
```

#### 3. Асинхронні кроки-функції
Після завершення фонової роботи функція записує результат безпосередньо у поновлюваний `completion` і викликає `resumeWith()`:

```kotlin
private fun loadCity(continuation: LoadDataContinuation) {
    Thread.sleep(3_000) // Імітація тривалої роботи у фоні

    runOnUiThread {
        continuation.city = "Kyiv" // Записуємо результат у поле
        continuation.resumeWith(Result.success(Unit)) // Відновлюємо стейт-машину
    }
}
```

### У чому головна суть цієї схеми?
1. Замість того, щоб чекати на відповідь і заблокувати потік (`Thread.sleep`), функція запускає фонову задачу, передаючи їй об'єкт `Continuation`, і **миттєво завершує поточний виклик**, звільняючи Main Thread.
2. Головний потік програми залишається повністю вільним для малювання UI, реакцій на кліки та анімації ProgressBar.
3. Коли фонова задача завершується, вона викликає метод `continuation.resumeWith()`, і Машина станів переходить на наступний крок (`label`), оновлюючи екран та продовжуючи роботу з місця паузи.

> 💡 **Готовий розв'язок:**
> Повний робочий код прикладів з корутинами можна переглянути в репозиторії [krenevych/Concurrency](https://github.com/krenevych/Concurrency) на гілці **`coroutine_under_the_hood_end`** (модуль **`coroutine`**).

---

## 3. Перетворення функцій з колбеками у `suspend` функції (`suspendCoroutine`)

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

## Підсумок

* **CPS (Continuation-Passing Style)** — компілятор додає прихований параметр `Continuation<T>` у кожну `suspend` функцію.
* **Continuation** — об'єкт-колбек, який зберігає точку відновлення та локальні змінні корутини.
* **State Machine** — компілятор розбиває `suspend` функцію на мітки (`label`) і перемикає їх при відновленні виконання.
* **`suspendCoroutine`** — дозволяє зручно обгортати застарілі функції з асинхронними колбеками у `suspend` функції.
* **Легковажність** — корутини це об'єкти в RAM, тому мільйони корутин можуть працювати на жменьці системних потоків.
