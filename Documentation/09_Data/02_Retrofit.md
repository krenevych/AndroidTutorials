# Робота з мережею: Retrofit та OkHttp

Майже жоден сучасний мобільний додаток не існує ізольовано. Застосунки стягують новини, курси валют, погоду або відео з Інтернету через API (найчастіше у форматі **JSON**).

В Android стандартом де-факто (Golden Standard) для роботи з мережею є бібліотека **Retrofit** від компанії Square, яка працює в парі з HTTP-клієнтом **OkHttp**.

---

## 1. Базовий підхід до мережі в Kotlin

Щоб зрозуміти, чому нам потрібні додаткові бібліотеки, варто поглянути, як виглядає завантаження даних "вручну" за допомогою стандартних засобів Kotlin. Ви можете використати клас `URL` і його зручну extension-функцію `readText()`:

```kotlin
// Робимо запит обов'язково у фоновому потоці (Dispatchers.IO)
suspend fun fetchUrlNaively() {
    withContext(Dispatchers.IO) {
        val url = URL("https://itc.ua/ua/novini/")
        
        // Функція readText() робить мережевий запит і викачує всю сторінку як текст
        val text = url.readText(Charset.defaultCharset())
        
        Log.d("App", "Отриманий текст: $text")
    }
}
```

Цей метод ідеально підходить, якщо вам потрібно просто швидко стягнути текстовий файл чи веб-сторінку. **Але для реальних додатків він не годиться, оскільки:**
1. **Відсутність парсингу JSON:** Уявіть, що замість сторінки сайту прийшов довгий рядок JSON-даних. Вам доведеться вручну розбирати його на Kotlin-об'єкти (`{ id: 1, name: 'Andrii' }`).
2. **Жодного керування налаштуваннями:** Ви не можете легко додати заголовки (Headers) для авторизації, вказати таймаути чи налаштувати кешування.

> ⚠️ **Важливо про дозволи:** Незалежно від того, використовуєте ви `URL.readText()` чи `Retrofit`, для будь-якого виходу в мережу ваш додаток **обов'язково** повинен мати дозвіл `INTERNET` у маніфесті. Без нього додаток впаде з `SecurityException`. Детальніше про це у наступному кроці.

---

## 2. Що таке API та які вони бувають?

Перш ніж стягувати дані, варто зрозуміти, куди саме ми будемо робити запити. Мобільні додатки отримують дані від серверів через **API (Application Programming Interface)** — програмний інтерфейс. 

Якщо простими словами: API — це "офіціант" у ресторані. Ваш мобільний додаток (клієнт) робить "замовлення" (надсилає запит), API передає його на кухню (сервер/базу даних), а потім повертає вам готове "замовлення" у вигляді чистого тексту, який зручно читати комп'ютерам (найчастіше це формат **JSON**).

Ось кілька прикладів популярних публічних API, які ви можете використовувати у своїх пет-проектах (домашніх розробках):

* **JSONPlaceholder** (`https://jsonplaceholder.typicode.com/users`) — безкоштовний фейковий API для навчання та тестування додатків. Повертає готові масиви користувачів, постів та коментарів.
* **npoint.io** (`https://www.npoint.io/`) — чудовий безкоштовний сервіс для створення власного фейкового API. Ви просто вставляєте туди свій JSON-файл, і сервіс генерує для вас унікальний URL для тестування.
* **ПриватБанк API** (`https://api.privatbank.ua/p24api/pubinfo?json&exchange&coursid=5`) — відкрите API, яке миттєво повертає актуальний курс валют в Україні.
* **CryptoCompare API** (`https://min-api.cryptocompare.com/data/price?fsym=BTC&tsyms=USD,EUR`) — потужне безкоштовне API для отримання актуальних цін на криптовалюти. Дуже зручно для створення додатків-трекерів курсу Біткоїна!
* **LibreTranslate API** (`https://libretranslate.com/`) — повністю відкрите та безкоштовне API для машинного перекладу тексту (альтернатива платному Google Translate). Ідеально для створення власного словника-перекладача.
* **OpenWeatherMap** (`https://api.openweathermap.org/data/2.5/weather?q=Kyiv&appid={your_key}`) — популярне API для отримання погоди. На відміну від попередніх, вимагає передати унікальний секретний `API_KEY`, щоб сервер міг ідентифікувати вас і не дозволив зробити забагато запитів.

---

## 3. Формат JSON та Моделі даних (POJO)

Як ми вже зазначали, 99% сучасних API повертають дані у форматі **JSON (JavaScript Object Notation)**. Це простий, легкий і зрозумілий як для людини, так і для машини формат обміну даними, який повністю витіснив застарілий XML у веб-розробці.

### Як виглядає JSON?
JSON складається з базових структур, які дуже легко мапляться на об'єкти Kotlin:
```json
{
  "user_id": 101,
  "name": "Андрій",
  "is_active": true,
  "roles": ["admin", "editor"],
  "address": {
    "city": "Kyiv",
    "zipcode": "01001"
  }
}
```

#### Анатомія JSON:
1. **Об'єкт (`{ ... }`):** Позначається фігурними дужками. Це словник, що містить набір пар "ключ-значення". У Kotlin він зазвичай перетворюється на ваш основний `data class`.
2. **Ключ:** Завжди є текстовим рядком (String), який береться у подвійні лапки (наприклад, `"name"` чи `"is_active"`). Це ідентифікатор, тобто унікальне ім'я певного поля/властивості всередині об'єкта. Ключ відокремлюється від значення двокрапкою `:`.
3. **Значення:** Може бути як простим (примітивним) типом, так і складною структурою:
   * **Рядок (String):** Обов'язково береться в подвійні лапки (`"Андрій"`).
   * **Число (Int, Double):** Пишеться без лапок (`101` або `3.14`).
   * **Логічний тип (Boolean):** Пишеться без лапок (`true` або `false`).
   * **Null:** Відсутність значення (пишеться як `null`).
   * **Інший Об'єкт (`{ ... }`):** Значенням може бути вкладений об'єкт (наприклад, `"address": { "city": "Kyiv" }`). Завдяки цьому JSON може описувати багаторівневі ієрархії (у Kotlin це буде вкладений Data-клас).
   * **Масив (`[ ... ]`):** Позначається квадратними дужками. Це впорядкований список будь-яких вищезгаданих значень (наприклад, масив рядків `"roles": ["admin", "editor"]` або масив об'єктів). У Kotlin він перетворюється на `List<T>`.

### Перетворення JSON у Kotlin: Data-класи (POJO)
Щоб працювати з JSON у Kotlin, ми повинні перетворити текст на повноцінні програмні об'єкти. Для цього використовується концепція **POJO (Plain Old Java Object)**, яка у світі Kotlin реалізується через **Data-класи**. Поля цього класу повинні в точності відповідати ключам у JSON.

> Нагадаємо, **Data-клас** — це спеціальний клас, єдиною метою якого, як правило, є зберігання даних (він не містить складної бізнес-логіки). 

#### Приклад: JSON ➔ Data-клас Kotlin
Припустимо, сервер повертає такий JSON-документ:
```json
{
  "id": 42,
  "first_name": "Іван",
  "is_active": true
}
```

Відповідний Kotlin POJO (Data-клас) для цього JSON виглядатиме так:
```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName

@Serializable
data class UserProfile(
    val id: Int,
    
    @SerialName("first_name")
    val firstName: String,
    
    @SerialName("is_active")
    val isActive: Boolean
)
```

**Що тут відбувається:**
1. **Анотація `@Serializable`:** Обов'язково ставиться перед оголошенням класу. Вона слугує маркером для плагіна компілятора `kotlinx.serialization`, який на етапі збірки проекту згенерує весь необхідний код для автоматичного парсингу цього класу з JSON і навпаки.
2. **Анотація `@SerialName`:** У бекенд-розробці для форматів JSON часто використовують зміїний регістр (snake_case), де слова розділяються підкресленнями (наприклад, `"first_name"`). Однак у Kotlin прийнято використовувати верблюжий регістр (CamelCase) для імен змінних (`firstName`). Анотація `@SerialName("first_name")` об'єднує ці два світи. Вона каже конвертеру: *"Коли шукаєш дані в JSON, шукай ключ `first_name`, але у коді Kotlin поклади це значення у змінну `firstName`"*.
3. Якщо ключ у JSON і назва вашої змінної у Kotlin **повністю збігаються** (як у випадку зі змінною `id`), використовувати анотацію `@SerialName` не потрібно — бібліотека зіставить їх автоматично!

### Корисні інструменти для роботи з JSON та POJO:
Під час розробки вам доведеться часто дивитися на "сирі" відповіді від сервера і конвертувати їх у код. Щоб пришвидшити цей процес, розробники використовують спеціальні інструменти:

#### 1. Онлайн-форматери (Web) для читання JSON
* **[JSON Formatter & Validator (curiousconcept.com)](https://jsonformatter.curiousconcept.com/)** — дозволяє вставити "брудний" сирий рядок JSON і миттєво перетворити його на гарне дерево, перевірити на синтаксичні помилки та розгорнути/згорнути вузли.
* **[JSONLint](https://jsonlint.com/)** — ще один дуже популярний валідатор та форматер.

#### 2. Плагіни для браузера (Chrome / Firefox)
* **JSON Viewer (Chrome Extension)** — якщо ви просто відкриєте API URL у браузері, цей плагін автоматично розмалює чорно-білий сирий текст у красиве підсвічене дерево, яке легко читати очима.

#### 3. Онлайн-генератори POJO класів (Web)
* **[JSON to Kotlin Data Class (json2kotlin.com)](https://json2kotlin.com/)** — ви вставляєте JSON, а сайт генерує готовий Kotlin Data-код, який можна скопіювати у проект.

#### 4. Плагіни для Android Studio (IDE)
Замість того, щоб копіювати текст у браузер, ви можете встановити плагіни прямо у ваше середовище розробки:
* **JSON Viewer** — дозволяє виділити будь-який JSON-рядок прямо у логах Logcat і відкрити його у красивому вікні всередині IDE.
* **JSON to Kotlin Class (JSON To Kotlin Class - JsonToKotlinClass)** — магічний плагін для автогенерації! Якщо JSON від сервера величезний (має десятки вкладених об'єктів), не пишіть класи вручну. Просто скопіюйте JSON у буфер обміну, оберіть у меню `New -> Kotlin data class File from JSON`, і плагін миттєво згенерує для вас усе дерево POJO-класів!

---

## 4. Крок 1: Дозволи та Залежності

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

## 6. Крок 2: Створення Data-класів для API

Припустимо, ми звертаємося до API `https://jsonplaceholder.typicode.com/users`, яке повертає масив користувачів у форматі JSON:
```json
{
  "id": 1,
  "name": "Leanne Graham",
  "email": "Sincere@april.biz"
}
```

Створюємо відповідний Data-клас у Kotlin та позначаємо його анотацією **`@Serializable`**. Ця анотація — сигнал для бібліотеки `kotlinx.serialization`, що цей клас можна автоматично парсити з JSON.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName

@Serializable
data class User(
    val id: Int,
    val name: String,
    
    // Якщо ім'я ключа в JSON не збігається з бажаним ім'ям змінної у Kotlin, 
    // використовуємо анотацію @SerialName, щоб підказати конвертеру правильний ключ:
    @SerialName("email") 
    val userEmail: String 
)
```

---

## 5. Крок 2: Опис API Інтерфейсу

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

## 6. Крок 3: Ініціалізація Retrofit

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

## 7. Крок 4: Виклик у ViewModel та відображення у Compose

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
