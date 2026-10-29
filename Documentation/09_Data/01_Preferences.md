# Робота з даними: SharedPreferences та DataStore

Кожен додаток потребує зберігання якихось простих даних між запусками: ім'я користувача, токен авторизації, вибрана темна/світла тема чи статус проходження онбордінгу.

Для збереження таких легковажних пар "ключ-значення" (Key-Value) не потрібно створювати складну реляційну базу даних. В Android для цього історично існують два підходи: застарілий **`SharedPreferences`** та його сучасний спадкоємець на базі корутин — **`Preferences DataStore`**.


---

## 1. Застарілий підхід: `SharedPreferences`

`SharedPreferences` (SP) — це найстаріший та колись найпопулярніший механізм Android, який зберігає дані у звичайному простому XML-файлі на диску пристрою. Хоча у нових проектах його більше не використовують, ви **гарантовано зустрінете його** при роботі з існуючими (legacy) додатками, тому важливо розуміти, як він працює.

> 📦 **Початковий проект:** Заготовку проекту (базовий Compose UI) ви знайдете у репозиторії на GitHub: [krenevych/Compose](https://github.com/krenevych/Compose) (гілка **`main`**, модуль **`sharedpref`**).<br>
> 💡 **Приклад розв'язку:** Готовий завершений код із реалізацією `SharedPreferences` знаходиться на гілці **`sharedpref_end`** (модуль **`sharedpref`**).

### Принцип роботи `SharedPreferences`:
Щоб зчитати або записати дані, ви спочатку отримуєте екземпляр SP через `Context`, вказавши ім'я файлу налаштувань (наприклад, `"app_settings"`):

```kotlin
import androidx.core.content.edit

// Отримуємо доступ до файлу app_settings.xml
val sharedPref = context.getSharedPreferences("app_settings", Context.MODE_PRIVATE)

// 1. ЗЧИТУВАННЯ ДАНИХ
// Другий параметр ("Unknown") — це дефолтне значення, якщо такого ключа не існує
val name = sharedPref.getString("key_username", "Unknown")
val hasNotifications = sharedPref.getBoolean("key_notifications", false)

// 2. ЗАПИС ДАНИХ
// Сучасний підхід у Kotlin через Extension-функцію (автоматично викликає .apply())
sharedPref.edit {
    putString("key_username", "Andrii")
    putBoolean("key_notifications", true)
}
```

> 💡 **Що таке `Context.MODE_PRIVATE`?**
> Це режим безпеки, який вказує, що файл налаштувань буде створений виключно для вашого додатка, і жоден інший сторонній додаток на телефоні не зможе його прочитати або змінити. Раніше існували режими на кшталт `MODE_WORLD_READABLE`, які дозволяли іншим додаткам читати ваші налаштування, але Google визнав їх небезпечними і заборонив у сучасних версіях Android.

### Де зберігаються файли SharedPreferences?
Дані записуються у внутрішню захищену пам'ять вашого додатка. Файл створюється автоматично і має назву, яку ви передали у метод `getSharedPreferences()` (плюс розширення `.xml`).

**Фізичний шлях на пристрої:**
`/data/data/<пакет_вашого_додатка>/shared_prefs/app_settings.xml`

Ця папка прихована від звичайних користувачів і доступна лише вашому додатку або операційній системі (з міркувань безпеки). Однак, під час розробки ви можете легко переглянути або змінити цей файл!

#### Як переглянути файл через ADB чи Android Studio:
1. **Device File Explorer в Android Studio:** 
   Відкрийте інструмент *Device File Explorer* (зазвичай у правому нижньому кутку екрану), перейдіть за шляхом `data/data/<ваш_пакет>/shared_prefs/` і просто відкрийте XML-файл подвійним кліком.
2. **Через термінал ADB:**
   Ви можете завантажити файл на комп'ютер за допомогою команди `adb pull`:
   ```bash
   # Завантажуємо файл app_settings.xml на робочий стіл комп'ютера
   adb shell "run-as com.example.myapp cat /data/data/com.example.myapp/shared_prefs/app_settings.xml" > ~/Desktop/app_settings.xml
   ```

### Переваги:
* Легко та швидко налаштувати (не потребує додавання залежностей у Gradle).
* Доступний у будь-якому класі, де є `Context`.

### Недоліки (Чому від нього відмовляються на користь DataStore?):
* **Блокування Main Thread (Синхронність):** Перше зчитування `SharedPreferences` повністю завантажує весь XML-файл у пам'ять, блокуючи той потік, у якому його викликано. Якщо файл великий, а ви читаєте його у Main Thread, інтерфейс підвисне, що може призвести до крашу ANR (Application Not Responding).
* **Відсутність безпеки типів (Type Safety):** Ви можете випадково записати число `putInt("score", 10)`, а інший розробник спробує зчитати його як рядок `getString("score", "")`. Це призведе до падіння додатку (`ClassCastException`).
* **Відсутність реактивності:** SP не має вбудованої підтримки `Flow` або `LiveData`. Якщо дані на диску зміняться (наприклад, у фоновій службі), ваш екран автоматично про це не дізнається. Вам доведеться вішати застарілі колбеки `OnSharedPreferenceChangeListener` та вручну оновлювати UI.

---

## 2. Сучасний підхід: `Preferences DataStore`

У 2020 році Google представив **DataStore** як ідеальну заміну SharedPreferences. Він повністю побудований на **Kotlin Coroutines та Flow**.

### Чому DataStore кращий?
1. **Асинхронність із коробки:** Усі операції зчитування та запису автоматично відбуваються у фонових потоках (`Dispatchers.IO`), ніколи не блокуючи UI.
2. **Безпека типів (Type Safety):** Використовуються строгі ключі з чітко заданим типом.
3. **Реактивність (Flow):** Зчитані дані повертаються у вигляді `Flow`. Це означає, що як тільки значення зміниться на диску, ваш UI (Compose) автоматично перемалюється з новими даними без жодних колбеків!

> 💡 **Де фізично зберігається DataStore?**
> Як і `SharedPreferences`, файли `DataStore` зберігаються у внутрішній закритій пам'яті вашого додатка, але в іншій папці та з іншим форматом.
> Шлях: `/data/data/<пакет_вашого_додатка>/files/datastore/app_settings.preferences_pb`
> Зверніть увагу: DataStore зберігає дані не у звичайному XML, а у спеціальному бінарному форматі Protocol Buffers (`.preferences_pb`), що робить його ще швидшим і надійнішим!
>
> 🛠️ **Як переглянути ці дані під час розробки?**
> Оскільки це бінарний формат, ви **не зможете** просто відкрити і прочитати його як звичайний текст в *Device File Explorer* або вбудованих інспекторах Android Studio. Якщо ви завантажите цей файл, ви побачите "кракозябри", серед яких можна буде розгледіти лише строкові ключі. 
> 
> Тому єдиний ефективний спосіб перевірити, що збережено в `DataStore` під час розробки — це **логування вичитування у вашому `Repository` або `ViewModel`** (наприклад, поставивши `Log.d` всередині `collect` або `map`).

---

## 3. Як працювати з Preferences DataStore

### Крок 1. Залежності
Додайте бібліотеку у `build.gradle.kts` вашого додатка:
```kotlin
implementation("androidx.datastore:datastore-preferences:1.0.0")
```

### Крок 2. Оголошення ключів
Для безпечної роботи у DataStore ми повинні створити строго типізовані "ключі". Для різних типів даних існують свої функції-білдери (`stringPreferencesKey`, `booleanPreferencesKey`, `intPreferencesKey` тощо).

```kotlin
object PreferencesKeys {
    val USER_NAME = stringPreferencesKey("key_username")
    val HAS_NOTIFICATIONS = booleanPreferencesKey("key_notifications")
}
```

### Крок 3. Створення екземпляра DataStore (на рівні Context)
Найкраща практика — створити `DataStore` один раз як розширення (Extension) для `Context` на самому верхньому рівні файлу (поза межами класів). Це зробить його єдиним (Singleton) для всього додатка.

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "app_settings")
```

### Крок 4. Клас-Репозиторій для роботи з даними
Щоб не розмазувати логіку по всьому проекту, створимо `SettingsRepository`, який буде інкапсулювати читання та запис.

```kotlin
class SettingsRepository(private val context: Context) {

    // 1. ЧИТАННЯ ДАНИХ (Повертає реактивний Flow)
    val userNameFlow: Flow<String> = context.dataStore.data
        .map { preferences ->
            // Якщо ключа немає, повертаємо дефолтне значення "Гість"
            preferences[PreferencesKeys.USER_NAME] ?: "Гість"
        }

    val hasNotificationsFlow: Flow<Boolean> = context.dataStore.data
        .map { preferences ->
            preferences[PreferencesKeys.HAS_NOTIFICATIONS] ?: false
        }

    // 2. ЗАПИС ДАНИХ (Асинхронна suspend функція)
    suspend fun saveUserName(name: String) {
        context.dataStore.edit { preferences ->
            preferences[PreferencesKeys.USER_NAME] = name
        }
    }

    suspend fun toggleNotifications(isEnabled: Boolean) {
        context.dataStore.edit { preferences ->
            preferences[PreferencesKeys.HAS_NOTIFICATIONS] = isEnabled
        }
    }
}
```

---

## 4. Інтеграція DataStore із ViewModel та Jetpack Compose

Тепер ми можемо елегантно прив'язати наш репозиторій до UI. ViewModel зчитає холодний `Flow` із DataStore і перетворить його у гарячий `StateFlow`, а Compose автоматично відреагує на нього.

### ViewModel:
```kotlin
class SettingsViewModel(application: Application) : AndroidViewModel(application) {
    
    private val repository = SettingsRepository(application.applicationContext)

    // Перетворюємо потік даних з DataStore у гарячий StateFlow для Compose
    val userName: StateFlow<String> = repository.userNameFlow
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), "Гість")

    val hasNotifications: StateFlow<Boolean> = repository.hasNotificationsFlow
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), false)

    // Метод для виклику з UI
    fun updateName(newName: String) {
        viewModelScope.launch {
            repository.saveUserName(newName) // Зберігаємо у DataStore
        }
    }
}
```

### Compose UI:
```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel) {
    // Підписуємося на стан із DataStore
    val userName by viewModel.userName.collectAsStateWithLifecycle()
    val hasNotifications by viewModel.hasNotifications.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        Text("Поточний користувач: $userName")
        Text("Сповіщення: ${if (hasNotifications) "Увімкнено" else "Вимкнено"}")

        Spacer(modifier = Modifier.height(16.dp))

        Button(onClick = { viewModel.updateName("Андрій") }) {
            Text("Змінити ім'я на 'Андрій'")
        }
    }
}
```

> 💡 **Магія DataStore у дії:** Коли ви натиснете кнопку "Змінити ім'я", `ViewModel` запише його в `DataStore`. Щойно дані збережуться на диск (у фоновому потоці), `DataStore` автоматично надішле нове значення у свій `Flow`. `ViewModel` отримає його, оновить свій `StateFlow`, а **Compose миттєво і самостійно перемалює текст** на екрані без жодних ваших додаткових команд!

---

## Підсумок
* **Не використовуйте `SharedPreferences`** у нових проектах через його синхронність та небезпеку блокування Main Thread.
* **Використовуйте `Preferences DataStore`** для збереження токенів, налаштувань UI та іншої легкої інформації.
* `DataStore` повністю асинхронний і працює у зв'язці з **`Kotlin Flow`**, що робить його ідеальним джерелом даних для Jetpack Compose.
