# Гарячі реактивні потоки: StateFlow та SharedFlow

У той час як звичайний `Flow` є холодним і заново запускається з нуля для кожного нового підписника, **Гарячі потоки (Hot Streams)** — `StateFlow` та `SharedFlow` — використовуються для збереження та передачі будь-якого активного стану системи (стан UI екрану, активна сесія користувача, статус з'єднання з мережею, бізнес-події) між різними компонентами та шарами застосунку.

Їхні екземпляри існують незалежно від наявності підписників. **`StateFlow` зберігає поточне значення**, а збереження попередніх значень у **`SharedFlow`** залежить від налаштування `replay`. Обидва типи можна використовувати у `ViewModel` та з **Jetpack Compose**.

---

## 1. Що таке Гарячі потоки (Hot Streams)?

* **Холодний потік (`Flow`)** — як запис фільму на диску: не грає, поки ви не натиснете Play (`collect`), і грає з нуля для кожного нового глядача.
* **Гарячий потік (`StateFlow` / `SharedFlow`)** — як спільна пряма трансляція: підписники підключаються до одного джерела. Новий підписник не запускає незалежний повтор із початку. Отримання попередніх значень залежить від типу потоку та його налаштувань.

---

## 2. `StateFlow` — Сучасна заміна `LiveData`

[`StateFlow`](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow) — це спеціальний гарячий потік, який **завжди зберігає поточний стан** (останнє значення) і негайно віддає його кожному новому підписнику.

### Основні властивості `StateFlow`:
* Має **обов'язкове початкове значення** (`initialValue`).
* Завжди зберігає **останнє значення** у пам'яті (доступне через `stateFlow.value`).
* **Автоматично відфільтровує дублікати** (якщо нове значення дорівнює попередньому `_state.value == newValue`, воно не надсилається повторно).
* Повністю сумісний з **Jetpack Compose** (через `collectAsStateWithLifecycle()`).

---

### Інкапсуляція StateFlow у ViewModel

Згідно з найкращими практиками Android, у `ViewModel` створюється приватний змінний `MutableStateFlow` та публічний незамінний `StateFlow` (через `.asStateFlow()`):

```kotlin
class NewsViewModel : ViewModel() {

    // Приватний змінний потік
    private val _uiState = MutableStateFlow<String>("Завантаження...")
    
    // Публічний потік тільки для читання для UI
    val uiState: StateFlow<String> = _uiState.asStateFlow()

    fun loadNews() {
        viewModelScope.launch {
            val news = repository.fetchNews()
            _uiState.value = news // Просте оновлення значення у Main Thread
        }
    }
}
```

---

## 3. Безпечна підписка на StateFlow у View та Jetpack Compose

### Варіант А: В Android Views (через `repeatOnLifecycle`)
Щоб підписка у `Activity` зупинялася при згортанні екрану у фон (`onStop`), використовують **`repeatOnLifecycle`**:

```kotlin
class MainActivity : AppCompatActivity() {

    private val viewModel: NewsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            // Автоматично відписується в onStop і відновлює підписку в onStart!
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { news ->
                    binding.tvNews.text = news
                }
            }
        }
    }
}
```

### Варіант Б: У Jetpack Compose (через `collectAsStateWithLifecycle`)
У Jetpack Compose `StateFlow` конвертується у стан Compose всього за один рядок:

```kotlin
@Composable
fun NewsScreen(viewModel: NewsViewModel = viewModel()) {
    // Конвертує StateFlow у Compose State
    val newsState by viewModel.uiState.collectAsStateWithLifecycle()

    Text(text = newsState)
}
```

---

## 4. Чому розробники переходять з LiveData на StateFlow?

Хоча `LiveData` довгий час була стандартом в Android, у сучасних проектах її масово замінюють на **`StateFlow`**. Для цього є 3 ключові причини:

### 1. Повна незалежність від Android (Unit-тести та KMP)
`LiveData` жорстко прив'язана до Android-фреймворку (`androidx.lifecycle`). 
* При написанні юніт-тестів для `ViewModel` з `LiveData` вам доводилося додавати спеціальне JUnit-правило `InstantTaskExecutorRule`. Для `StateFlow` жодних правил не потрібно — це чистий Kotlin, який тестується миттєво на звичайній JVM.
* `ViewModel` із `StateFlow` можна без змін перевикористовувати у **Kotlin Multiplatform (KMP)** для спільної бізнес-логіки під iOS, Desktop та Web!

### 2. Потужність та гнучкість операторів Flow
Якщо у вас є кілька джерел даних (наприклад: *пошуковий рядок* + *фільтри категорій* + *сортування*), у `LiveData` доводилося використовувати складний та громіздкий `MediatorLiveData` чи `Transformations.switchMap`.

Зі `StateFlow` це робиться в один читабельний ланцюжок завдяки багатому набору операторів (`combine`, `flatMapLatest`, `debounce`, `map`).

### 3. Першокласна сумісність із Jetpack Compose
У сучасному декларативному UI-фреймворку від Google (Jetpack Compose) підписка на стан `StateFlow` записується всього в один лаконічний рядок:
```kotlin
val state by viewModel.uiState.collectAsStateWithLifecycle()
```
Це робить підключення `ViewModel` до Compose-інтерфейсу максимально простим та безпечним.

### 4. Підсумкова порівняльна таблиця: LiveData vs StateFlow

| Характеристика | `LiveData` | `StateFlow` |
| :--- | :--- | :--- |
| **Залежність** | Залежить від Android (`androidx.lifecycle`) | **Чистий Kotlin** (без прив'язки до Android / KMP-ready) |
| **Початкове значення** | Необов'язкове | **Обов'язкове** |
| **Підтримка Compose** | Потребує конвертації | **Першокласна нативна підтримка** |
| **Оператори** | Обмежені (`map`, `switchMap`) | Багатий набір операторів Kotlin Flow (`combine`, `debounce`) |

---

## 5. Золотий стандарт ViewModel: Оператор `stateIn`

Найчастіше у репозиторії ви отримуєте звичайний холодний `Flow` (наприклад, з бази даних Room чи файлу). Щоб перетворити його на гарячий `StateFlow` усередині `ViewModel`, використовується оператор **`stateIn`**:

```kotlin
class UserViewModel(repository: UserRepository) : ViewModel() {

    // Автоматично перетворює холодний Flow з бази у гарячий StateFlow
    val userState: StateFlow<User> = repository.getUserFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000), // Золотий стандарт!
            initialValue = User.EMPTY
        )
}
```

> 💡 **Чому саме `SharingStarted.WhileSubscribed(5000)`?**
> Цей параметр каже: *"Тримай потік активним, поки є хоч один спостерігач. Якщо спостерігач зник (наприклад, екран перевернувся), почекай паузу у 5 секунд. Якщо за 5 секунд ніхто не підписався знову — зупини фоновий потік для збереження ресурсів"*.

### Як зчитувати такий `StateFlow` у UI?
Оскільки оператор `stateIn` повертає стандартний `StateFlow`, у `Activity` чи Compose підписка на нього відбувається точно так само, як і на будь-який інший `StateFlow` (через `repeatOnLifecycle` у Views або `collectAsStateWithLifecycle()` у Compose):

```kotlin
// У Jetpack Compose зчитування відбувається миттєво:
val userState by viewModel.userState.collectAsStateWithLifecycle()
```

---

## 6. `SharedFlow` — Спільний потік подій та відмінності від `StateFlow`

**`SharedFlow`** — це гарячий потік, який транслює значення кільком підписникам. Кожен підписник має власну підписку на спільне джерело. Наприклад, повідомлення про зміну з'єднання можуть слухати одночасно екран та компонент журналювання.

`StateFlow` також підтримує кількох підписників. Ба більше, **`StateFlow` є спеціалізованим різновидом `SharedFlow`** для зберігання поточного стану. Тому різниця між ними полягає передусім у правилах зберігання та передачі значень. [Документація StateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/).

### 6.1. Стан і подія — різні задачі

Уявімо екран завантаження новин:

* **Стан:** зараз триває завантаження, показано певний список або сталася помилка. Після повороту екрана новому UI потрібно одразу отримати актуальний стан і відобразити його. Для цього підходить `StateFlow`.
* **Подія:** користувач натиснув кнопку, і потрібно показати коротке інформаційне повідомлення. Повторне відкриття екрана саме по собі не повинне повторювати вже показане повідомлення. Для таких сповіщень можна використати `SharedFlow` із `replay = 0`, якщо їх допустимо пропустити за відсутності UI.

Різницю добре видно на однакових значеннях:

```kotlin
val state = MutableStateFlow("")

state.value = "Збережено"
state.value = "Збережено" // Значення не змінилося — нового оновлення немає
```

Для стану це природно: поточний текст залишився тим самим. Проте два окремі повідомлення можуть мати однаковий текст:

```kotlin
val events = MutableSharedFlow<String>(replay = 0)

// Усередині корутини, коли вже є активний підписник:
events.emit("Збережено")
events.emit("Збережено") // Окреме надсилання, хоча текст такий самий
```

`SharedFlow` не відфільтровує рівні значення автоматично. У цьому прикладі активний підписник, який продовжує збирати потік, отримує обидва повідомлення. `StateFlow` порівнює значення через `equals` і може пропускати проміжні швидкі оновлення для повільного підписника: його задача — передати актуальний стан. [Поведінка StateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/), [SharedFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/).

### 6.2. Порівняння поведінки

| Властивість | `StateFlow` | `SharedFlow` |
| --- | --- | --- |
| Початкове значення | Обов'язкове для `MutableStateFlow` | Не вимагається |
| Читання без підписки | Поточне значення через `.value` | Властивості `.value` немає |
| Повторне рівне значення | Не створює нового оновлення | Може бути надіслане знову |
| Новий підписник | Отримує поточний стан | Отримує до `replay` попередніх значень, потім нові |
| Немає підписників | Поточний стан зберігається | Зберігаються лише останні `replay` значень; при `replay = 0` попередні події не зберігаються |
| Повільний підписник | Може пропускати проміжні стани | Поведінку визначають буфер та стратегія його переповнення |
| Типове застосування | Стан екрана, налаштування, поточний список | Сповіщення про події, спільна трансляція значень |

### 6.3. `replay`: що побачить новий підписник?

`replay` визначає кількість останніх значень, які потік повторить **кожному новому підписнику**:

| Налаштування | Поведінка під час нової підписки |
| --- | --- |
| `replay = 0` | Попередніх значень немає, підписник чекає наступного надсилання |
| `replay = 1` | Якщо значення вже надсилали, підписник спочатку отримує останнє |
| `replay = 2` | Спочатку отримує до двох останніх значень у порядку їх надсилання |

Наприклад, усередині корутини:

```kotlin
val messages = MutableSharedFlow<String>(replay = 1)

messages.emit("Перше повідомлення")
messages.emit("Друге повідомлення")

// Новий підписник отримає "Друге повідомлення", а потім чекатиме нових:
messages.collect { message -> println(message) }
```

Інший новий підписник також отримає «Друге повідомлення»: читання не вилучає його з replay-кешу. При `replay = 0` обидва надсилання до початку підписки були б пропущені. [Replay-кеш SharedFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/).

**`MutableSharedFlow(replay = 1)` усе ще не є `StateFlow`:** він не вимагає початкового значення, не має `.value` і не відфільтровує дублікати автоматично.

### 6.4. Буфер та різниця між `emit()` і `tryEmit()`

`MutableSharedFlow` має три основні параметри:

Без аргументів використовуються `replay = 0`, `extraBufferCapacity = 0` та `BufferOverflow.SUSPEND`. Для прикладу додамо одне місце в буфері:

```kotlin
val events = MutableSharedFlow<String>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.SUSPEND
)
```

| Параметр | Призначення |
| --- | --- |
| `replay` | Зберігає останні значення для нових підписників |
| `extraBufferCapacity` | Додає місце для значень, які ще не встигли отримати наявні повільні підписники |
| `onBufferOverflow` | Визначає поведінку при переповненні: `SUSPEND` — призупинити надсилання; `DROP_OLDEST` — відкинути найстаріше значення в буфері; `DROP_LATEST` — відкинути нове |

Стратегії `DROP_OLDEST` та `DROP_LATEST` потребують місця в буфері: `replay > 0` або `extraBufferCapacity > 0`. Для `BufferOverflow` використовується імпорт `kotlinx.coroutines.channels.BufferOverflow`. [Параметри MutableSharedFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-mutable-shared-flow.html).

* **`emit(value)`** — `suspend`-функція: за стратегії `SUSPEND` може призупинити корутину, доки з'явиться можливість передати значення. Потік виконання при цьому не блокується. [Документація emit](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-mutable-shared-flow/emit.html).
* **`tryEmit(value)`** — не призупиняє корутину та повертає `Boolean`. `false` означає, що надсилання потребувало б очікування. При цьому `true` не гарантує, що хтось обробив повідомлення. [Документація tryEmit](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-mutable-shared-flow/try-emit.html).

> ⚠️ **Буфер не замінює `replay`.** Коли підписників немає, зберігаються лише значення replay-кешу. Навіть `extraBufferCapacity = 100` не збереже події до появи підписника при `replay = 0`: `emit()` завершиться, а значення буде пропущене. `tryEmit()` у такій ситуації поверне `true`.

### 6.5. Приклад сповіщень із ViewModel

У цьому прикладі повідомлення призначені лише для активного екрана. Якщо екрана немає, пропуск повідомлення є допустимим:

```kotlin
class MessagesViewModel : ViewModel() {

    private val _eventFlow = MutableSharedFlow<String>(
        replay = 0,
        extraBufferCapacity = 1
    )
    val eventFlow: SharedFlow<String> = _eventFlow.asSharedFlow()

    fun onShowMessageClicked() {
        viewModelScope.launch {
            _eventFlow.emit("Інформаційне повідомлення")
        }
    }
}
```

### 6.6. Як підписатися на `SharedFlow` у UI?

Стан зазвичай перетворюємо на `State` Compose та відображаємо. Події обробляємо через `collect` у корутині: отримали повідомлення — викликали `Toast`. Не розміщуйте показ `Toast` безпосередньо в тілі composable, яке може виконуватися повторно під час рекомпозицій.

#### Варіант А: В Android Views (`Activity` / `Fragment`)
Підписка виконується усередині `repeatOnLifecycle`, щоб не показувати Toast чи не робити навігацію, коли екран перебуває у фоні:

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.eventFlow.collect { message ->
            // Обробляємо повідомлення, отримане цією підпискою
            Toast.makeText(this@MainActivity, message, Toast.LENGTH_SHORT).show()
        }
    }
}
```

#### Варіант Б: У Jetpack Compose (через `LaunchedEffect`)
`LaunchedEffect` керує корутиною відповідно до перебування composable у композиції. Щоб додатково зупиняти підписку при переході екрана у фон, використаємо `repeatOnLifecycle`:

```kotlin
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.compose.LocalLifecycleOwner
import androidx.lifecycle.repeatOnLifecycle

@Composable
fun MessagesScreen(viewModel: MessagesViewModel) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    LaunchedEffect(viewModel, lifecycleOwner) {
        lifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.eventFlow.collect { message ->
                Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
            }
        }
    }
}
```

У стані нижче `STARTED` підписка скасовується, а після повернення — створюється знову. При `replay = 0` уже надіслані повідомлення не повторюються новій підписці. Водночас повідомлення, надіслане за відсутності всіх підписників, буде пропущене. [Підписки з урахуванням життєвого циклу](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow).

### 6.7. Чи гарантує SharedFlow обробку події «рівно один раз»?

**Ні. `replay = 0` вимикає повторення попередніх значень новим підписникам, але не підтверджує виконання дії в UI.** Подія може бути надіслана під час відсутності підписки або обробка може перерватися через скасування корутини. А якщо на потік одночасно підписані два компоненти, кожен може виконати свою реакцію на те саме повідомлення.

Для важливого результату, наприклад успішного входу в обліковий запис, зберігайте відповідний **стан**, за яким UI відновить правильний екран. Для повідомлення, яке потрібно обов'язково показати, можна зберігати його в UI-стані та прибирати після підтвердження обробки. `SharedFlow` зі сповіщеннями доречний, коли така гарантія не потрібна. [Рекомендації Android щодо UI-подій](https://developer.android.com/topic/architecture/ui-layer/events).

---

## Підсумок

* **`StateFlow`** — гарячий потік, що завжди зберігає поточний стан UI (сучасний замінник `LiveData`).
* **`SharedFlow`** — спільний гарячий потік із налаштуванням повторення значень через `replay` та буферизації. Підходить для трансляції сповіщень кільком підписникам.
* **`SharedFlow` із `replay = 0`** не повторює попередні значення новим підписникам, але й не зберігає їх за відсутності підписок та не гарантує обробку «рівно один раз».
* **`stateIn(SharingStarted.WhileSubscribed(5000))`** — перетворює `Flow` у `StateFlow` з паузою в 5 секунд при повороті екрану.
* **`repeatOnLifecycle(STARTED)`** — безпечний спосіб підписки у `Activity`.
* **`collectAsStateWithLifecycle()`** — спосіб конвертації `StateFlow` у стан Jetpack Compose.
