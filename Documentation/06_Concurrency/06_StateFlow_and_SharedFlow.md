# Гарячі реактивні потоки: StateFlow та SharedFlow

У той час як звичайний `Flow` є холодним і заново запускається з нуля для кожного нового підписника, **Гарячі потоки (Hot Streams)** — `StateFlow` та `SharedFlow` — використовуються для збереження та передачі будь-якого активного стану системи (стан UI екрану, активна сесія користувача, статус з'єднання з мережею, бізнес-події) між різними компонентами та шарами застосунку.

Вони зберігають значення в оперативній пам'яті незалежно від наявності підписників, що робить їх ідеальними для використання у `ViewModel` та з **Jetpack Compose**.

---

## 1. Що таке Гарячі потоки (Hot Streams)?

* **Холодний потік (`Flow`)** — як запис фільму на диску: не грає, поки ви не натиснете Play (`collect`), і грає з нуля для кожного нового глядача.
* **Гарячий потік (`StateFlow` / `SharedFlow`)** — як пряма трансія по ТБ: вона іде постійно, зберігає свій стан у RAM, а підписники просто підключаються до неї та отримують поточну картинку.

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

## 5. `SharedFlow` — Для одноразових подій (Single Events)

**`SharedFlow`** — це гарячий потік **без початкового значення**. Він використовується для надсилання **одноразових подій UI** (Events), які не повинні повторюватися при повороті екрану:
* Показ `Toast` чи `Snackbar`.
* Одноразова навігація (перехід на новий екран).

```kotlin
class LoginViewModel : ViewModel() {

    private val _eventFlow = MutableSharedFlow<String>()
    val eventFlow: SharedFlow<String> = _eventFlow.asSharedFlow()

    fun onLoginClicked() {
        viewModelScope.launch {
            // Емітимо одноразову подію
            _eventFlow.emit("Авторизація успішна!")
        }
    }
}
```

### Як підписатися на `SharedFlow` у UI?

#### Варіант А: В Android Views (`Activity` / `Fragment`)
Підписка виконується усередині `repeatOnLifecycle`, щоб не показувати Toast чи не робити навігацію, коли екран перебуває у фоні:

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.eventFlow.collect { message ->
            // Подія обробляється і показується РІВНО один раз!
            Toast.makeText(this@MainActivity, message, Toast.LENGTH_SHORT).show()
        }
    }
}
```

#### Варіант Б: У Jetpack Compose (через `LaunchedEffect`)
У Compose одноразові події з `SharedFlow` слухають усередині ефекту **`LaunchedEffect`**:

```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel) {
    val context = LocalContext.current

    // LaunchedEffect запускає корутину підписки для обробки подій
    LaunchedEffect(Unit) {
        viewModel.eventFlow.collect { message ->
            Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
        }
    }
}
```

---

## 6. Золотий стандарт ViewModel: Оператор `stateIn`

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

## Підсумок

* **`StateFlow`** — гарячий потік, що завжди зберігає поточний стан UI (сучасний замінник `LiveData`).
* **`SharedFlow`** — гарячий потік для одноразових подій (Toast, Навігація).
* **`stateIn(SharingStarted.WhileSubscribed(5000))`** — перетворює `Flow` у `StateFlow` з паузою в 5 секунд при повороті екрану.
* **`repeatOnLifecycle(STARTED)`** — безпечний спосіб підписки у `Activity`.
* **`collectAsStateWithLifecycle()`** — спосіб конвертації `StateFlow` у стан Jetpack Compose.
