# Гарячі реактивні потоки: StateFlow та SharedFlow

У той час як звичайний `Flow` є холодним і запускається з нуля для кожного підписника, для зберігання стану екрану в Android використовуються **Гарячі потоки (Hot Streams)**: `StateFlow` та `SharedFlow`.

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

## 3. Золотий стандарт ViewModel: Оператор `stateIn`

Найчастіше у репозиторії ви отримуєте звичайний холодний `Flow` (наприклад, з бази даних Room). Щоб перетворити його на гарячий `StateFlow` усередині `ViewModel`, використовується оператор **`stateIn`**:

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

---

## 4. Безпечна підписка на StateFlow у View та Jetpack Compose

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
            _eventFlow.emit("Авторизація успішна!")
        }
    }
}
```

---

## Порівняння: LiveData vs StateFlow

| Характеристика | `LiveData` | `StateFlow` |
| :--- | :--- | :--- |
| **Залежність** | Залежить від Android (`androidx.lifecycle`) | **Чистий Kotlin** (без прив'язки до Android) |
| **Початкове значення** | Необов'язкове | **Обов'язкове** |
| **Підтримка Compose** | Потребує конвертації | **Першокласна нативна підтримка** |
| **Оператори** | Обмежені (`map`, `switchMap`) | Величезний набір операторів Kotlin Flow |

---

## Підсумок

* **`StateFlow`** — гарячий потік, що завжди зберігає поточний стан UI (сучасний замінник `LiveData`).
* **`SharedFlow`** — гарячий потік для одноразових подій (Toast, Навігація).
* **`stateIn(SharingStarted.WhileSubscribed(5000))`** — перетворює `Flow` у `StateFlow` з паузою в 5 секунд при повороті екрану.
* **`repeatOnLifecycle(STARTED)`** — безпечний спосіб підписки у `Activity`.
* **`collectAsStateWithLifecycle()`** — спосіб конвертації `StateFlow` у стан Jetpack Compose.
