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

## 2. Залежності та Плагін

Room використовує генерацію коду (KSP — Kotlin Symbol Processing) для створення реалізацій ваших інтерфейсів. 

Відкрийте файл `build.gradle.kts` вашого модуля і додайте плагін KSP нагорі:
```kotlin
plugins {
    // ...
    id("com.google.devtools.ksp") // Плагін для кодогенерації Room
}
```

Потім додайте самі бібліотеки Room у блок `dependencies`:
```kotlin
dependencies {
    val room_version = "3.0.3"
    
    // Основна бібліотека Room
    implementation("androidx.room3:room3-runtime:$room_version")
    
    // Компілятор Room (обробник анотацій через KSP)
    ksp("androidx.room3:room3-compiler:$room_version")
}
```

> 💡 **Новий Room 3.x:** Починаючи з версії 3.0, Room підтримує **KMP (Kotlin Multiplatform)** (зверніть увагу на зміну префіксу пакетів на `androidx.room3`). Окрім того, підтримка `Coroutines` та `Flow` (яка раніше вимагала окремого пакету `room-ktx`) тепер **вбудована безпосередньо** у базовий пакет `room3-runtime`. Тому окремої залежності для `-ktx` більше не потрібно!

> ⚠️ *Примітка:* Щоб KSP запрацював, вам також потрібно додати його версію (яка відповідає вашій версії Kotlin) у файл `build.gradle.kts` рівня проекту!

---

## 3. Крок 2: Створення Entity (Таблиці)

Створимо таблицю для збереження користувачів. Використовуємо анотацію **`@Entity`** перед звичайним Data-класом.

```kotlin
import androidx.room.Entity
import androidx.room.PrimaryKey
import androidx.room.ColumnInfo

// Назва таблиці в базі буде "users_table"
@Entity(tableName = "users_table")
data class UserEntity(
    
    // Кожна таблиця зобов'язана мати первинний ключ.
    // autoGenerate = true означає, що SQLite сама буде нумерувати (1, 2, 3...)
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    
    // За замовчуванням колонка в базі назветься "name", 
    // але ви можете змінити це через @ColumnInfo
    @ColumnInfo(name = "user_name")
    val name: String,
    
    val email: String
)
```

---

## 4. Крок 3: Створення DAO (Data Access Object)

Тут ми описуємо, ЯК саме ми хочемо працювати з таблицею. Це звичайний `interface` з анотацією **`@Dao`**.

```kotlin
import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface UserDao {

    // 1. ЗАПИС ДАНИХ (Виконується асинхронно через suspend)
    // OnConflictStrategy.REPLACE - якщо такий ID вже є, він оновиться
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: UserEntity)

    // 2. ВИДАЛЕННЯ ДАНИХ
    @Query("DELETE FROM users_table WHERE id = :userId")
    suspend fun deleteUserById(userId: Int)

    // 3. ЗЧИТУВАННЯ ДАНИХ (Повертає реактивний Flow!)
    // Зверніть увагу: ми НЕ пишемо suspend для функцій, що повертають Flow!
    @Query("SELECT * FROM users_table ORDER BY user_name ASC")
    fun getAllUsers(): Flow<List<UserEntity>>
}
```

> 💡 **Магія Flow у Room:** Зверніть увагу на функцію `getAllUsers()`. Оскільки вона повертає `Flow`, Room налаштує **автоматичний наглядач (Observer)** за цією таблицею. 
> Щоразу, коли ви викличете `insertUser` або `deleteUserById`, таблиця зміниться, і Room **АВТОМАТИЧНО згенерує новий список у Flow**. Ваш екран у Compose перемалюється миттєво без потреби викликати "оновити дані" вручну!

---

## 5. Крок 4: Створення Database (Бази даних)

Створюємо абстрактний клас бази даних, який наслідується від `RoomDatabase`. 

```kotlin
import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

// Вказуємо список усіх таблиць (entities) та версію бази даних
@Database(entities = [UserEntity::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {

    // Метод, через який ми отримуватимемо доступ до нашого DAO
    abstract fun userDao(): UserDao

    companion object {
        // Запобігаємо створенню кількох екземплярів бази в пам'яті (Singleton)
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database.db" // Назва фізичного файлу бази на диску
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

---

## 6. Крок 5: Використання у ViewModel та Compose

Залишилося лише з'єднати наш чудовий реактивний `Dao` з інтерфейсом користувача через ViewModel.

### ViewModel:
```kotlin
class UsersViewModel(application: Application) : AndroidViewModel(application) {
    
    // Отримуємо екземпляр DAO
    private val userDao = AppDatabase.getDatabase(application).userDao()

    // 1. Отримуємо холодний Flow з Room і перетворюємо його у гарячий StateFlow для Compose
    val allUsers: StateFlow<List<UserEntity>> = userDao.getAllUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList() // Поки база вантажиться, список порожній
        )

    // 2. Метод для додавання нового користувача по кліку
    fun addUser(name: String, email: String) {
        viewModelScope.launch {
            val newUser = UserEntity(name = name, email = email)
            userDao.insertUser(newUser) // Зберігаємо у фоновому потоці
        }
    }
}
```

### Compose UI:
```kotlin
@Composable
fun UsersScreen(viewModel: UsersViewModel) {
    // Безпечно підписуємося на реактивний потік з бази даних!
    val usersList by viewModel.allUsers.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        
        Button(onClick = { viewModel.addUser("Олексій", "alex@mail.com") }) {
            Text("Додати Олексія")
        }

        Spacer(modifier = Modifier.height(16.dp))

        // Відображаємо список
        LazyColumn {
            items(usersList) { user ->
                Card(modifier = Modifier.fillMaxWidth().padding(4.dp)) {
                    Text(
                        text = "ID: ${user.id} | ${user.name}",
                        modifier = Modifier.padding(16.dp)
                    )
                }
            }
        }
    }
}
```

Коли ви натискаєте на кнопку "Додати", дані записуються у файл `app_database.db`. Room бачить зміну у файлі і автоматично виштовхує новий `List<UserEntity>` у ваш `StateFlow`. UI самостійно малює новий рядок!

---

## Підсумок
* **`Room`** — це найбезпечніший і найзручніший спосіб роботи з локальними базами даних SQLite в Android.
* Складається з трьох рівнів: **`Entity`** (модель/таблиця), **`Dao`** (інтерфейс запитів), **`Database`** (файл бази та зв'язка).
* Запити з модифікатором `@Query` перевіряються системою на правильність SQL-коду **ще під час компіляції** (ви ніколи не впадете з помилкою через синтаксис під час роботи додатка).
* Використовуйте функції, що повертають **`Flow`** у `Dao`, щоб мати повністю реактивний UI, який оновлюється сам при будь-яких змінах у таблиці!
