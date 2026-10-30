# Робота з мережею: Retrofit та OkHttp

Майже жоден сучасний мобільний додаток не існує ізольовано. Застосунки стягують новини, курси валют, погоду або відео з Інтернету через API (найчастіше у форматі **JSON**).

В Android стандартом де-факто (Golden Standard) для роботи з мережею є бібліотека **Retrofit** від компанії Square, яка працює в парі з HTTP-клієнтом **OkHttp**.

---

## 1. Як працює мережевий стек в Android?

1. **Retrofit:** Це зручна "обгортка". Ви просто створюєте Kotlin-інтерфейс і позначаєте методи анотаціями (наприклад, `@GET("users")`). Retrofit сам згенерує весь необхідний код для здійснення запиту.
2. **OkHttp:** Це потужний "двигун" під капотом Retrofit, який безпосередньо виконує HTTP-запити, керує з'єднаннями, таймаутами та логуванням.
3. **Конвертер (Kotlinx Serialization / Gson):** Автоматично перетворює сирий текст JSON, який прийшов із сервера, у зручні Kotlin Data-класи (об'єкти).

> ⚠️ **Критичне правило (NetworkOnMainThreadException):** 
> Операційна система Android суворо **забороняє** робити будь-які мережеві запити у Головному потоці (Main Thread). Якщо ви це зробите, додаток миттєво впаде з помилкою. На щастя, Retrofit має вбудовану підтримку `suspend` функцій і автоматично виконує запити у фоні!

---

## 2. Крок 1: Дозволи та Залежності

### Дозвіл на Інтернет (Маніфест)
Перше, про що забувають 90% початківців — це дозвіл на вихід в Інтернет. Відкрийте `AndroidManifest.xml` і додайте цей рядок над тегом `<application>`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### Залежності (Gradle)
Додайте необхідні бібліотеки у файл `build.gradle.kts` вашого модуля:

```kotlin
dependencies {
    // 1. Retrofit (Головна бібліотека)
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    
    // 2. Конвертер JSON (на базі Kotlinx Serialization)
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.11.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
    
    // 3. OkHttp Logging Interceptor (Для перегляду логів запитів у Logcat)
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
}
```
*(Для `kotlinx-serialization` також необхідно підключити однойменний плагін у блоці `plugins {}` нагорі файлу).*

---

## 3. Крок 2: Data-класи (Моделі)

Припустимо, ми звертаємося до відкритого API, яке повертає профіль користувача у форматі JSON:
```json
{
  "id": 1,
  "name": "Leanne Graham",
  "email": "Sincere@april.biz"
}
```

Створюємо відповідний Data-клас у Kotlin та позначаємо його анотацією `@Serializable` (це вказує конвертеру, що цей клас можна розпарсити з JSON):

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName

@Serializable
data class User(
    val id: Int,
    val name: String,
    // Якщо ім'я поля в JSON відрізняється від назви змінної у Kotlin, використовуємо @SerialName
    @SerialName("email") 
    val userEmail: String 
)
```

---

## 4. Крок 3: Опис API Інтерфейсу

Далі ми маємо описати наші мережеві запити. Ми створюємо звичайний інтерфейс і використовуємо анотації Retrofit:

```kotlin
import retrofit2.http.GET
import retrofit2.http.Path
import retrofit2.http.Query

interface ApiService {

    // GET-запит до https://api.example.com/users
    // Повертає список користувачів
    @GET("users")
    suspend fun getUsers(): List<User>

    // Запит із динамічним шляхом (Path). 
    // Якщо userId = 5, URL буде: https://api.example.com/users/5
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") userId: Int): User

    // Запит із параметрами пошуку (Query).
    // URL буде: https://api.example.com/users?sort=desc
    @GET("users")
    suspend fun getSortedUsers(@Query("sort") sortOrder: String): List<User>
}
```
> 💡 Зверніть увагу: ми використовуємо модифікатор **`suspend`**. Завдяки цьому ми зможемо викликати ці функції прямо з Корутин (`viewModelScope`), і Retrofit сам перенесе мережеву роботу на безпечний фоновий потік!

---

## 5. Крок 4: Ініціалізація Retrofit

Тепер нам потрібно створити екземпляр Retrofit та прив'язати його до нашого інтерфейсу. Зазвичай це роблять у синглтоні (через `object` або за допомогою Hilt/Dagger).

```kotlin
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType

object NetworkModule {
    
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"

    // 1. Налаштовуємо OkHttp клієнт (додаємо логер, щоб бачити запити у Logcat)
    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY // Логуємо ВСЕ (заголовки і тіло)
    }

    private val client = OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .build()

    // 2. Налаштовуємо конвертер JSON
    private val json = Json { ignoreUnknownKeys = true } // Ігноруємо зайві поля в JSON
    private val contentType = "application/json".toMediaType()

    // 3. Збираємо Retrofit
    val retrofit: Retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(client)
        .addConverterFactory(json.asConverterFactory(contentType))
        .build()

    // 4. Створюємо реалізацію нашого інтерфейсу
    val apiService: ApiService = retrofit.create(ApiService::class.java)
}
```

---

## 6. Крок 5: Виклик у ViewModel та відображення у Compose

Найкращий патерн для обробки мережевих запитів — це створення Sealed-класу (або інтерфейсу) для відображення стану екрану (Завантаження ➔ Успіх / Помилка).

### 1. State (Стан):
```kotlin
sealed interface UsersUiState {
    object Loading : UsersUiState
    data class Success(val users: List<User>) : UsersUiState
    data class Error(val message: String) : UsersUiState
}
```

### 2. ViewModel:
```kotlin
class UsersViewModel : ViewModel() {

    private val _uiState = MutableStateFlow<UsersUiState>(UsersUiState.Loading)
    val uiState: StateFlow<UsersUiState> = _uiState.asStateFlow()

    init {
        fetchUsers() // Стягуємо дані при відкритті екрану
    }

    fun fetchUsers() {
        viewModelScope.launch {
            _uiState.value = UsersUiState.Loading // Показуємо крутилку
            try {
                // Виконуємо запит через Retrofit (він сам працює у фоні)
                val usersList = NetworkModule.apiService.getUsers()
                
                // Якщо все добре, передаємо дані у State
                _uiState.value = UsersUiState.Success(usersList)
            } catch (e: Exception) {
                // Відловлюємо відсутність інтернету або помилки сервера
                _uiState.value = UsersUiState.Error(e.localizedMessage ?: "Unknown Error")
            }
        }
    }
}
```

### 3. Compose UI:
```kotlin
@Composable
fun UsersScreen(viewModel: UsersViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UsersUiState.Loading -> {
            CircularProgressIndicator() // Показуємо індикатор завантаження
        }
        is UsersUiState.Error -> {
            Text(text = "Помилка: ${state.message}", color = Color.Red)
            Button(onClick = { viewModel.fetchUsers() }) {
                Text("Спробувати знову")
            }
        }
        is UsersUiState.Success -> {
            // Відображаємо реальні дані з мережі у LazyColumn!
            LazyColumn {
                items(state.users) { user ->
                    Text(text = "${user.name} (${user.userEmail})")
                }
            }
        }
    }
}
```

## Підсумок
* **`Retrofit`** — зручний генератор мережевих викликів на основі Kotlin-інтерфейсів.
* **`suspend` функції** у Retrofit позбавляють нас необхідності вручну перемикати потоки (не треба писати `withContext(Dispatchers.IO)`).
* Завжди додавайте дозвіл **`INTERNET`** в маніфест.
* Для зручного дебагу підключайте **`HttpLoggingInterceptor`** (від OkHttp), щоб бачити кожен запит та відповідь у Logcat.
* Огортайте UI стан у **Sealed Class** (`Loading`, `Success`, `Error`), щоб легко керувати відображенням на екрані за допомогою Compose `when`.
