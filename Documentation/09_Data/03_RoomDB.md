# Локальна база даних: Room Database

У попередніх уроках ми навчилися зберігати прості налаштування (через `DataStore`) та викачувати дані з Інтернету (через `Retrofit`). Але що робити, якщо додатку потрібно зберегти велику кількість структурованих даних (наприклад, тисячі контактів, історію повідомлень або скачані офлайн-статті), щоб вони були доступні без Інтернету?

Для цього операційна система Android має вбудовану легковажну реляційну базу даних — **SQLite**.

Проте писати сирі SQL-запити вручну та конвертувати таблиці в об'єкти Kotlin дуже важко і небезпечно. Тому Google створив бібліотеку **Room** — потужну обгортку над SQLite, яка перевіряє SQL-код ще на етапі компіляції і автоматично перетворює рядки з бази на Kotlin-об'єкти.

> 📖 **Поглиблене читання:** Детальна офіційна документація по роботі з Room від Google знаходиться за посиланням: [Save data in a local database using Room](https://developer.android.com/training/data-storage/room).

---

## 1. Архітектура Room Database

Щоб налаштувати базу даних Room, вам потрібно створити три обов'язкові компоненти:

1. **Entity (Сутність / Модель):** Data-клас, який описує структуру однієї таблиці в базі даних. Кожен об'єкт цього класу — це один рядок у таблиці.
2. **DAO (Data Access Object):** Інтерфейс, у якому ви оголошуєте методи доступу до бази (`Insert`, `Delete`, `Update`, `Query`). Room сам згенерує їхній код.
3. **Database (База даних):** Абстрактний клас, який об'єднує всі Entity та DAO разом і слугує головною точкою доступу до фізичного файлу бази на диску.

**Як ці компоненти взаємодіють між собою:**
Клас бази даних (`Database`) надає вашому додатку екземпляри об'єктів доступу до даних (`DAO`), пов'язані з цією базою. Своєю чергою, додаток використовує `DAO` для отримання даних з бази у вигляді екземплярів об'єктів сутностей (`Entity`). Додаток також може використовувати створені сутності `Entity` для оновлення відповідних рядків у таблицях бази або для створення нових рядків (вставки). Цю взаємодію наочно проілюстровано на схемі нижче:

![Архітектура Room](res/room_architecture.png)

---

## 2. Залежності та плагін KSP

Room використовує **KSP (Kotlin Symbol Processing)** для генерації реалізацій DAO та класу бази даних.

**1. Кореневий файл (Project) `build.gradle.kts`:**
Спочатку оголосіть версію плагіна KSP:
```kotlin
plugins {
    // ... інші плагіни ...
    id("com.google.devtools.ksp") version "2.3.10" apply false
}
```

**2. Файл модуля `build.gradle.kts`:**
Застосуйте плагін KSP у модулі, вже без вказівки версії:
```kotlin
plugins {
    // ... інші плагіни ...
    id("com.google.devtools.ksp") // Підключаємо плагін KSP
}
```

Потім додайте бібліотеки Room у блок `dependencies`:
```kotlin
dependencies {
    val room_version = "3.0.3"
    
    // Основна бібліотека Room
    implementation("androidx.room3:room3-runtime:$room_version")
    
    // Компілятор Room (обробник анотацій через KSP)
    ksp("androidx.room3:room3-compiler:$room_version")
}
```

> 💡 **Що змінилося в Room 3.x?** Версія 3 використовує пакет `androidx.room3`, вимагає KSP для генерації коду та явного налаштування SQLite-драйвера під час створення бази. Підтримка **KMP (Kotlin Multiplatform)** та об'єднання `room-ktx` із runtime з'явилися ще в Room 2.7. У Room 3 підтримка корутин і `Flow` також входить до `room3-runtime`, тому окремий `-ktx` не потрібний. [Room 3](https://developer.android.com/jetpack/androidx/releases/room3#3.0.0), [Room 2.7](https://developer.android.com/jetpack/androidx/releases/room#2.7.0).

Після змін виконайте **Sync Project with Gradle Files**. Версію Room можна звірити з [офіційною документацією](https://developer.android.com/jetpack/androidx/releases/room3), а підключення плагіна — з [інструкцією KSP](https://kotlinlang.org/docs/ksp-quickstart.html).

---

## 3. Створення та використання бази даних Room

> 📦 **Початковий проект:** Заготовку проекту (базовий Compose UI) ви знайдете у репозиторії на GitHub: [krenevych/Compose](https://github.com/krenevych/Compose) (гілка **`main`**, модуль **`roomdb`**).<br>
> 💡 **Приклад розв'язку:** Готовий завершений код із реалізацією `Room` знаходиться на гілці **`roomdb`** (модуль **`roomdb`**).

### Крок 1: Створення Entity (Таблиці)

Створимо таблицю для збереження карток. Використовуємо анотацію **`@Entity`** перед звичайним Data-класом.

```kotlin
import androidx.room3.ColumnInfo
import androidx.room3.Entity
import androidx.room3.PrimaryKey

// Назва таблиці в базі буде "card_table"
@Entity(tableName = "card_table")
data class CardData(
    
    // Кожна таблиця зобов'язана мати первинний ключ.
    // autoGenerate = true доручає SQLite генерувати унікальний ID
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,

    // За замовчуванням колонка в базі назветься як і змінна, 
    // але ви можете змінити це через @ColumnInfo
    @ColumnInfo(name = "Title")
    val title: String = "Заголовок картки",
    
    @ColumnInfo(name = "Description")
    val description: String = "Це дуже детальний опис...",
    
    @ColumnInfo(name = "CreationTime")
    val creationTime: String = "2023-10-27 15:30"
)
```

#### Чому нова картка має `id = 0`?

Для цілочисельного ключа з `autoGenerate = true` значення `0` під час вставки означає «ID ще не задано». SQLite згенерує ключ для нового рядка, тому кілька нових об'єктів із `id = 0` не замінюють один одного. Не варто очікувати, що номери завжди йтимуть без пропусків.

Початковий Kotlin-об'єкт із `val id` після вставки сам не зміниться. У цьому прикладі ми отримуємо згенеровані ID, коли Room читає збережені картки через `getAllCards()`. Для оновлення або видалення потрібно використовувати саме ID збереженого рядка. [Документація `@PrimaryKey`](https://developer.android.com/reference/kotlin/androidx/room3/PrimaryKey).

---

### Крок 2: Створення DAO (Data Access Object)

Тут ми описуємо, ЯК саме ми хочемо працювати з таблицею. Це звичайний `interface` з анотацією **`@Dao`**.

```kotlin
import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.OnConflictStrategy
import androidx.room3.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface CardDao {

    // 1. ЗАПИС ДАНИХ (Room виконує операцію асинхронно)
    // REPLACE: у разі конфлікту ключа старий рядок видаляється, а новий вставляється
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertCard(card: CardData)

    // 2. ВИДАЛЕННЯ ДАНИХ
    @Query("DELETE FROM card_table WHERE id = :id")
    suspend fun deleteCardById(id: Int)

    // 3. ЗЧИТУВАННЯ ДАНИХ (Повертає реактивний Flow!)
    // Зверніть увагу: ми НЕ пишемо suspend для функцій, що повертають Flow!
    @Query("SELECT * FROM card_table ORDER BY Title ASC")
    fun getAllCards(): Flow<List<CardData>>
}
```

> 💡 **Як працює Flow у Room:** `getAllCards()` повертає холодний `Flow`. Коли на нього підписуються, Room виконує запит і починає відстежувати зміни таблиці. Після змін Room повторно виконує запит та передає його результат у потік. У кроці 4 підключимо цей потік до стану Compose, щоб список оновлювався без ручного перезавантаження даних.

#### Основні операції та запити до БД

| Операція | Анотація або запит у DAO | Що робить |
| --- | --- | --- |
| Додавання | `@Insert` | Додає переданий об'єкт `CardData` як новий рядок таблиці. |
| Оновлення | `@Update` | Оновлює рядок за первинним ключем переданого об'єкта `CardData`. Якщо рядка немає, нічого не змінює. |
| Додавання або оновлення | `@Upsert` | Вставляє новий рядок або оновлює наявний за первинним ключем переданого об'єкта `CardData`. |
| Видалення об'єкта | `@Delete` | Видаляє рядок за первинним ключем переданого об'єкта `CardData`. |
| Отримання всіх карток | `@Query("SELECT * FROM card_table")` | Повертає всі рядки таблиці. |
| Пошук за ID | `@Query("SELECT * FROM card_table WHERE id = :id")` | Повертає картку із заданим ID. Для одноразового читання використовуйте тип результату `CardData?`, адже картки може не бути. |
| Сортування | `@Query("SELECT * FROM card_table ORDER BY Title ASC")` | Повертає картки за зростанням заголовка; `DESC` задає зворотний порядок. |
| Оновлення окремого поля | `@Query("UPDATE card_table SET Title = :title WHERE id = :id")` | Змінює лише заголовок картки із заданим ID. |
| Видалення за ID | `@Query("DELETE FROM card_table WHERE id = :id")` | Видаляє картку без потреби передавати цілий об'єкт. |
| Підрахунок карток | `@Query("SELECT COUNT(*) FROM card_table")` | Повертає кількість рядків, наприклад як `Int`. |

Позначення `:id` та `:title` — це параметри методу DAO з відповідними іменами. Для одноразового читання та зміни даних використовуйте `suspend`; для спостереження за результатами читання — `Flow` без `suspend`, як у прикладі вище.

#### Чим `REPLACE` відрізняється від оновлення?

`@Insert(onConflict = OnConflictStrategy.REPLACE)` задає поведінку вставки при конфлікті. Наприклад, якщо первинний ключ уже зайнятий, SQLite **видаляє попередній рядок і вставляє новий**. Це може мати значення для пов'язаних таблиць і тригерів. [Поведінка `REPLACE` у SQLite](https://www.sqlite.org/lang_conflict.html).

Для зміни наявної картки використовуйте `@Update`. Якщо потрібна поведінка «додати, коли ID відсутній, або оновити, коли він уже є», використовуйте `@Upsert`. У цих методах передавайте об'єкт зі збереженим ID, коли хочете змінити конкретну картку. Для анотацій із таблиці додавайте відповідні імпорти з `androidx.room3`, наприклад `import androidx.room3.Upsert`.

> 📖 **Детальніше:** [Робота з даними через DAO в офіційній документації Room](https://developer.android.com/training/data-storage/room/accessing-data).

---

### Крок 3: Створення Database (Бази даних)

Створюємо абстрактний клас бази даних, який наслідується від `RoomDatabase`. 

```kotlin
import android.content.Context
import androidx.room3.Database
import androidx.room3.Room
import androidx.room3.RoomDatabase
import androidx.sqlite.driver.AndroidSQLiteDriver

// Вказуємо список усіх таблиць (entities) та версію бази даних
@Database(entities = [CardData::class], version = 1, exportSchema = false)
abstract class CardDatabase : RoomDatabase() {

    // Метод, через який ми отримуватимемо доступ до нашого DAO
    abstract fun cardDao(): CardDao

    companion object {
        // Запобігаємо створенню кількох екземплярів бази в пам'яті (Singleton)
        @Volatile
        private var INSTANCE: CardDatabase? = null

        fun getDatabase(context: Context): CardDatabase {
            return INSTANCE ?: synchronized(this) {
                // Подвійна перевірка (Double-checked locking) для потокобезпеки
                INSTANCE?.let { return it }
                
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    CardDatabase::class.java,
                    "Cards" // Назва фізичного файлу бази на диску
                )
                    .setDriver(AndroidSQLiteDriver())
                    .build()
                
                INSTANCE = instance
                instance
            }
        }
    }
}
```

`AndroidSQLiteDriver` використовує SQLite, вбудовану в Android. У Room 3 потрібно явно передати драйвер через `.setDriver(...)`, інакше побудувати базу не вдасться. Тут драйвер доступний через залежності Room; додаткова бібліотека з власною SQLite для цього прикладу не потрібна. [Створення бази Room](https://developer.android.com/training/data-storage/room#usage).

Ми передаємо `context.applicationContext`, оскільки спільний екземпляр бази живе довше за окрему Activity. Так він не утримуватиме контекст знищеної Activity після повороту екрана чи переходу на інший екран.

#### Навіщо потрібна подвійна перевірка `INSTANCE`?

Ми хочемо створити **один спільний екземпляр `CardDatabase`** для додатка. Але `getDatabase()` можуть одночасно викликати різні потоки. Перевірка «якщо бази ще немає — створи її» складається з кількох дій, між якими інший потік також може почати створення бази.

У коді є дві перевірки, і кожна має свою роль:

* **Перша — `INSTANCE ?: synchronized(this) { ... }`.** Якщо екземпляр уже існує, повертаємо його одразу, без блокування. Якщо `INSTANCE == null`, намагаємося увійти в `synchronized`.
* **Друга — `INSTANCE?.let { return it }`.** Після входу в `synchronized` перевіряємо значення повторно: поки потік чекав на блокування, інший потік міг уже створити екземпляр. Якщо він є, `return it` повертає його з `getDatabase()`.

Розгляньмо ситуацію з двома потоками:

1. Потоки **А** та **Б** майже одночасно виконують першу перевірку. Обидва бачать `INSTANCE == null`.
2. Потік **А** першим входить у `synchronized(this)`. Потік **Б** чекає: блок із тим самим об'єктом блокування може виконувати лише один потік одночасно. Тут `this` — спільний `companion object`.
3. Потік **А** повторно перевіряє `INSTANCE`, створює базу, записує її в `INSTANCE` та звільняє блокування.
4. Потік **Б** входить у блок. **Без другої перевірки він створив би ще один екземпляр**, адже першу перевірку вже пройшов раніше. Завдяки `INSTANCE?.let { return it }` він повертає екземпляр, створений потоком **А**.

Отже, перша перевірка дозволяє швидко отримувати вже створену базу, а друга запобігає повторному створенню після очікування блокування. Це і є **Double-checked locking**.

**А навіщо `@Volatile`?** Перша перевірка читає `INSTANCE` поза `synchronized`. Анотація гарантує видимість записаного значення для інших потоків: потік, який прочитає посилання на базу, також побачить результати її ініціалізації. Самої `@Volatile` недостатньо: вона не робить усю послідовність «перевірити → створити → присвоїти» неподільною, тому блокування та повторна перевірка залишаються потрібними.

> 📖 **Детальніше:** [`synchronized`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/synchronized.html) та [`@Volatile`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.jvm/-volatile/) в офіційній документації Kotlin.

#### Що означає `version = 1` і коли потрібна міграція?

Це **версія схеми бази даних**, а не версія застосунку або бібліотеки Room. Схема описує таблиці, колонки, їхні типи та обмеження. Наприклад, якщо після випуску застосунку додати поле до `CardData`, потрібно оновити структуру вже збереженої бази на пристроях користувачів.

Для такої зміни підвищують версію схеми, наприклад із `1` до `2`, та визначають **міграцію** — спосіб переходу між версіями зі збереженням даних. Одного збільшення числа недостатньо: потрібно описати перехід через `Migration` і додати його до налаштувань бази. Якщо необхідного переходу немає, за стандартних налаштувань відкриття старої бази завершиться помилкою.

У цьому прикладі `exportSchema = false` вимикає створення JSON-описів схеми, щоб не налаштовувати їхній експорт. **Дані карток при цьому зберігаються у базі як звичайно.** Автоматичні міграції потребують експорту схем; у цьому уроці ми їх не використовуємо.

У нашому прикладі залишаємо версію `1`, адже створюємо базу вперше.

> 📖 **Детальніше:** [Міграції та експорт схем Room](https://developer.android.com/training/data-storage/room/migrating-db-versions).

---

### Крок 4: Використання у ViewModel та Compose

З'єднаємо реактивний DAO з інтерфейсом користувача через ViewModel. Екран отримуватиме список карток і передаватиме у ViewModel події додавання та видалення.

> 💡 `viewModel()` та `collectAsStateWithLifecycle()` належать до інтеграції Lifecycle з Compose. Вони потрібні для наведеного UI-прикладу, але не для роботи Room 3. Якщо вони вже доступні у вашій заготовці, додаткових залежностей для них не потрібно.

#### ViewModel

```kotlin
class CardsViewModel(application: Application) : AndroidViewModel(application) {
    
    // Отримуємо екземпляр DAO
    private val cardDao = CardDatabase.getDatabase(application).cardDao()

    // 1. Отримуємо холодний Flow з Room і перетворюємо його у гарячий StateFlow для Compose
    val allCards: StateFlow<List<CardData>> = cardDao.getAllCards()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(stopTimeoutMillis = 5_000),
            initialValue = emptyList() // Початкове значення до першого результату Room
        )

    // 2. Метод для додавання нової картки
    fun addCard(title: String, description: String) {
        viewModelScope.launch {
            val newCard = CardData(title = title, description = description)
            cardDao.insertCard(newCard) // Room виконає операцію поза головним потоком
        }
    }
    
    // 3. Метод для видалення
    fun removeCard(id: Int) {
        viewModelScope.launch {
            cardDao.deleteCardById(id)
        }
    }
}
```

#### Як працюють `stateIn` та `WhileSubscribed`?

`getAllCards()` повертає холодний `Flow`: спостереження починається під час його збирання. `stateIn(...)` перетворює його на спільний `StateFlow`, що зберігає останнє значення для підписників. Аргументи визначають поведінку цього перетворення:

| Аргумент | Для чого потрібний |
| --- | --- |
| `scope = viewModelScope` | Прив'язує корутину збирання до ViewModel. Коли ViewModel знищується, її корутини скасовуються. |
| `SharingStarted.WhileSubscribed(stopTimeoutMillis = 5_000)` | Починає збирати Flow з Room із появою першого підписника. Через п'ять секунд після зникнення останнього підписника зупиняє збирання. Затримка допомагає пережити коротку перерву, наприклад під час повороту екрана. |
| `initialValue = emptyList()` | Задає значення до першого результату Room. Порожній список сам по собі не дозволяє відрізнити завантаження від справді порожньої бази. |

**П'ять секунд — це затримка зупинки спостереження, а не період опитування бази.** Після зупинки останній список залишається у `StateFlow` за стандартних налаштувань. Коли підписник з'явиться знову, збирання відновиться. Ця поведінка стосується читання списку; методи додавання та видалення запускають окремі корутини.

`viewModelScope.launch` за замовчуванням запускається у головному потоці, а `suspend` сам по собі не означає «працювати у фоні». У цьому прикладі саме **Room** забезпечує виконання операцій DAO поза головним потоком, тому вручну додавати `withContext(Dispatchers.IO)` навколо цих викликів не потрібно.

> 📖 **Детальніше:** [`stateIn`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html), [`WhileSubscribed`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-sharing-started/-companion/-while-subscribed.html) та [асинхронні запити Room](https://developer.android.com/training/data-storage/room/async-queries).

#### Compose UI

Для простоти кнопка додає картки із заданими заголовком та описом. У кожної буде власний ID, а час створення залишаємо демонстраційним рядком із `CardData`. Пізніше ви можете замінити заголовок та опис введенням із `TextField`.

```kotlin
@Composable
fun CardsScreen(viewModel: CardsViewModel) {
    // Безпечно підписуємося на реактивний потік з бази даних!
    val cardsList by viewModel.allCards.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        
        Button(onClick = { viewModel.addCard("Нова картка", "Опис...") }) {
            Text("Додати картку")
        }

        Spacer(modifier = Modifier.height(16.dp))

        // Відображаємо список
        LazyColumn {
            items(
                items = cardsList,
                key = { card -> card.id }
            ) { card ->
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(vertical = 4.dp)
                ) {
                    Column(modifier = Modifier.padding(16.dp)) {
                        Text(text = "ID: ${card.id} | ${card.title}")
                        Text(text = card.description)
                        Text(text = "Створено: ${card.creationTime}")
                        TextButton(onClick = { viewModel.removeCard(card.id) }) {
                            Text("Видалити")
                        }
                    }
                }
            }
        }
    }
}
```

`collectAsStateWithLifecycle()` перетворює `StateFlow` на стан Compose та збирає його, коли життєвий цикл екрана перебуває щонайменше у стані `STARTED`. Коли екран переходить у фон і стає нижче `STARTED`, збирання припиняється; якщо інших підписників немає, через п'ять секунд `WhileSubscribed` зупинить і спостереження за Room. Після повернення підписка відновиться. [Документація про стан у Compose](https://developer.android.com/develop/ui/compose/state#other-supported-types-of-state).

#### Звідки екран отримує ViewModel?

У `MainActivity` всередині `onCreate()` використайте `viewModel()` у блоці `setContent`. Збережіть тему Compose із заготовки навколо вмісту екрана:

```kotlin
import androidx.activity.compose.setContent
import androidx.lifecycle.viewmodel.compose.viewModel

// Усередині onCreate(), після super.onCreate(savedInstanceState):
setContent {
    // Цей вміст розмістіть усередині теми вашого застосунку
    val cardsViewModel: CardsViewModel = viewModel()
    CardsScreen(viewModel = cardsViewModel)
}
```

`viewModel()` отримує екземпляр із власника ViewModel; у цьому прикладі ним є Activity. Після звичайної рекомпозиції або повороту екрана використовується та сама ViewModel. Стандартна фабрика Activity передає `Application` у конструктор нашого `AndroidViewModel`. Не створюйте `CardsViewModel(...)` вручну в тілі composable. [Отримання ViewModel у Compose](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-apis).

Після натискання «Додати картку» Room записує дані у таблицю `card_table` у файлі `Cards`. Зміна таблиці запускає повторне виконання запиту активного Flow, і новий `List<CardData>` потрапляє через `StateFlow` у стан Compose. Екран відображає оновлений список. Кнопка «Видалити» проходить тим самим шляхом, тому вручну змінювати список у UI не потрібно.

---

### Крок 5: Перевірка збереження даних

1. Запустіть застосунок і додайте кілька карток. Переконайтеся, що вони мають різні ID та відображають задані дані.
2. Повністю закрийте й повторно запустіть застосунок, не очищаючи його дані. Картки мають залишитися: вони збережені у файлі бази, а не лише у ViewModel.
3. Видаліть одну картку кнопкою «Видалити». Список має оновитися автоматично.
4. Повторно перезапустіть застосунок. Видалена картка не повинна з'явитися знову.

#### Як подивитися таблицю в Android Studio?

Запустіть застосунок на емуляторі або пристрої з **API 26 чи новішим**. Додайте хоча б одну картку, щоб база вже була відкрита, та перейдіть у **View → Tool Windows → App Inspection → Database Inspector**. Виберіть процес застосунку, розгорніть базу `Cards` та відкрийте таблицю `card_table`.

Увімкніть **Live updates**, щоб бачити додавання та видалення рядків під час роботи з UI. Для перевірки можна також виконати запит:

```sql
SELECT * FROM card_table ORDER BY id;
```

У колонці `CreationTime` ви побачите той самий текстовий час, що й на екрані. Database Inspector працює з вбудованою в Android SQLite, яку ми використовуємо через `AndroidSQLiteDriver`. [Документація Database Inspector](https://developer.android.com/studio/inspect/database).

---

## Підсумок

* **`Room`** спрощує роботу з локальними базами даних SQLite в Android.
* Складається з трьох рівнів: **`Entity`** (модель/таблиця), **`Dao`** (інтерфейс запитів), **`Database`** (файл бази та зв'язка).
* Room перевіряє SQL-запити в анотаціях `@Query` **під час компіляції**.
* **`Flow` → `StateFlow` → стан Compose** пов'язують зміни в таблиці з оновленням UI.
* Версія схеми та міграції дозволяють змінювати структуру бази зі збереженням даних користувача.
