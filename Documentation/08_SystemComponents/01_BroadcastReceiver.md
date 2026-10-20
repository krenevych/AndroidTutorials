# Системні компоненти: BroadcastReceiver та міжпроцесна взаємодія

Операційна система Android заснована на чотирьох фундаментальних компонентах: **Activity**, **Service**, **BroadcastReceiver** та **ContentProvider**.

У цьому туторіалі ми розглянемо **`BroadcastReceiver`** — системний компонент, який дозволяє додатку слухати та реагувати на широкомовні сповіщення (Broadcast Intents) від самієї системи Android або інших застосунків.

---

## 1. Що таке `BroadcastReceiver`?

**`BroadcastReceiver` (Приймач широкомовних сповіщень)** — це компонент Android, який діє як "слухач подій" (Event Listener) на рівні всієї операційної системи.

Коли в системі відбувається важлива подія (увімкнено режим "У літаку", розрядилася батарея, зникло Wi-Fi з'єднання або пристрій підключили до зарядного пристрою), Android розсилає широкомовне сповіщення (**Broadcast Intent**).

```
[ Android OS / Системні події ]
        │
        ├─── (Режим у літаку) ────► [ BroadcastReceiver ] ───► Оновлення UI / Логіка
        ├─── (Зарядка батареї) ───► [ BroadcastReceiver ]
        └─── (Зміна мережі) ──────► [ BroadcastReceiver ]
```

### Створення `BroadcastReceiver`

Щоб створити приймач сповіщень, необхідно створити клас (або анонімний об'єкт), який успадковується від абстрактного класу **`BroadcastReceiver`**, та перевизначити його єдиний обов'язковий метод **`onReceive(context, intent)`**:

```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {

    // Метод викликається системою Android щоразу, коли надходить Intent,
    // який відповідає налаштованому фільтру цього приймача
    override fun onReceive(context: Context?, intent: Intent?) {
        // 1. Отримуємо Action події
        val action = intent?.action

        // 2. Виконуємо необхідну логіку
        Log.d("BR", "Отримано бродкаст із акшеном: $action")
    }
}
```

> ⚠️ **Критичне правило `onReceive()`:**
> Метод `onReceive()` виконується безпосередньо у **Головному потоці (Main Thread)** вашого застосунку! Усередині нього **категорично заборонено виконувати важкі чи тривалі операції** (запити до мережі, бази даних чи складні обчислення), оскільки це призведе до завісання інтерфейсу та помилки **ANR (Application Not Responding)**. Для тривалих задач викоритовують корутини чи `WorkManager`.

> 💡 **Зверніть увагу:** Сам по собі створений клас `BroadcastReceiver` ще не слухає подій системи. Щоб він почав отримувати повідомлення, його **обов'язково потрібно зареєструвати** в системі (статично у `AndroidManifest.xml` або динамічно у коді). Детально про всі способи реєстрації ми поговоримо далі у **Розділі 4 ("Способи реєстрації: Статична проти Динамічної")**.

---

## 2. Механіка роботи: `Intent`, `Action` та `IntentFilter`

Щоб зрозуміти, як Android точно розпізнає та доправляв сповіщення до потрібного `BroadcastReceiver`, розглянемо три ключові поняття:

```
[ Відправник ] ───► Intent (усередині Action = "ACTION_AIRPLANE_MODE_CHANGED")
                          │
                          ▼ (ОС Android шукає збіги за фільтрами)
                          │
[ IntentFilter (Action = "ACTION_AIRPLANE_MODE_CHANGED") ]
                          │
                          ▼
[ BroadcastReceiver ] ────► метод onReceive()
```

### 1. `Intent` (Намір / Конверт сповіщення)
Це об'єкт-повідомлення, який надсилається в систему. Він виступає "конвертом", що містить текстову назву події (`Action`) та додаткові дані (`Extras`).

### 2. `Action` (Дія / Назва події)
Це **унікальний рядок-ідентифікатор** усередині `Intent`, який описує, *що саме сталося*.
* **Для системних подій:** ці рядки-константи вже заготовлені в ОС Android у класах `Intent` та `ConnectivityManager` (наприклад, `"android.intent.action.BATTERY_LOW"`).
* **Для кастомних подій:** ви придумываєте цей рядок самостійно, дотримуючись унікального пакетирування (наприклад, `"com.example.ACTION_DATA_PROCESSED"`).

### 3. `IntentFilter` (Фільтр подій для Приймача)
Це правило, за допомогою якого ваш `BroadcastReceiver` сповіщає ОС Android: *"Я хочу слухати ТІЛЬКИ ті `Intent`, у яких `Action` дорівнює X та Y"*. Якщо `Action` в `Intent` збігається з вашим `IntentFilter`, система автоматично викликає метод `onReceive()`.

---

## 3. Види широкомовних сповіщень (Broadcasts)

### 1. Системні бродкасти (System Broadcasts)
Надсилаються самою ОС Android при виникненні системних подій чи зміні стану пристрою:

| Системний Action (Intent) | Подія / Опис | Особливість реєстрації | Додаткові дані в Intent (Extras) |
| :--- | :--- | :--- | :--- |
| **`Intent.ACTION_AIRPLANE_MODE_CHANGED`** | Перемикання режиму "У літаку" (Airplane Mode). | Динамічна у коді | `intent.getBooleanExtra("state", false)` — стан режиму (`true` — увімкнено, `false` — вимкнено). |
| **`Intent.ACTION_BATTERY_CHANGED`** | Детальна зміна стану та відсотка заряду батареї. | Динамічна у коді | `BatteryManager.EXTRA_LEVEL` (поточний відсоток), `BatteryManager.EXTRA_STATUS` (статус: Charging/Full), `BatteryManager.EXTRA_PLUGGED` (джерело: USB/AC/Wireless). |
| **`Intent.ACTION_HEADSET_PLUG`** | Підключення / відключення дротових навушників. | Динамічна у коді | `intent.getIntExtra("state", -1)` (`1` — навушники встромлено, `0` — вийнято), `intent.getStringExtra("name")` (назва пристрою). |
| **`ConnectivityManager.CONNECTIVITY_ACTION`** | Зміна стану мережевого з'єднання (Wi-Fi / Mobile Data). | Динамічна у коді | `intent.getBooleanExtra(ConnectivityManager.EXTRA_NO_CONNECTIVITY, false)` (`true` — мережа повністю відсутня). |
| **`Intent.ACTION_POWER_CONNECTED`** | Підключення пристрою до зарядного пристрою. | Динамічна у коді | |
| **`Intent.ACTION_POWER_DISCONNECTED`** | Відключення від зарядного пристрою. | Динамічна у коді | |
| **`Intent.ACTION_BATTERY_LOW`** | Досягнення критично низького рівня заряду батареї. | Динамічна у коді | |
| **`Intent.ACTION_BATTERY_OKAY`** | Відновлення нормального рівня заряду після розрядки. | Динамічна у коді | |
| **`Intent.ACTION_BOOT_COMPLETED`** | Завершення завантаження системи після увімкнення пристрою. | **Статична у `AndroidManifest.xml`** | |
| **`Intent.ACTION_SCREEN_ON`** | Увімкнення дисплею пристрою. | Динамічна у коді | |
| **`Intent.ACTION_SCREEN_OFF`** | Вимкнення / блокування дисплею пристрою. | Динамічна у коді | |
| **`Intent.ACTION_USER_PRESENT`** | Успішне розблокування екрану користувачем. | Динамічна у коді | |
| **`Intent.ACTION_TIME_TICK`** | Хвилиний відлік системи (надсилається щохвилини). | Динамічна у коді | |

### 2. Кастомні бродкасти (Custom Broadcasts)
Окрім системних подій, ви можете надсилати власні широкомовні повідомлення для міжпроцесної взаємодії чи передачі подій між компонентами за допомогою методу **`context.sendBroadcast(intent)`**.

Механіка передачі даних працює за таким принципом:
1. **Створення `Intent` з унікальним Action:** Оголошується унікальний рядок-ідентифікатор події (наприклад, `"com.example.ACTION_DATA_PROCESSED"`).
2. **Пакування даних у `Intent`:** За допомогою методу **`putExtra("key", value)`** усередині `Intent` запаковується корисне навантаження (рядки, числа, прапорці чи об'єкти).
3. **Розпакування на стороні Приймача:** У методі `onReceive(context, intent)` слухач перевіряє `intent.action` і безпечно витягує дані за допомогою методів `intent.getStringExtra("key")` або `intent.getIntExtra("key", defaultValue)`.

#### Приклад надсилання кастомного бродкасту:
```kotlin
// 1. Створюємо Intent з унікальною дією та додаємо корисне навантаження
val intent = Intent("com.example.ACTION_DATA_PROCESSED").apply {
    putExtra("key_message", "Обробку даних успішно завершено!")
    putExtra("key_code", 200)
}

// 2. Відправляємо широкомовне сповіщення
context.sendBroadcast(intent)
```

#### Приклад зчитування даних у `BroadcastReceiver`:
```kotlin
override fun onReceive(context: Context?, intent: Intent?) {
    if (intent?.action == "com.example.ACTION_DATA_PROCESSED") {
        // Витягуємо передані дані з Intent
        val message = intent.getStringExtra("key_message")
        val code = intent.getIntExtra("key_code", -1)
        
        Log.d("BR", "Отримано сповіщення: $message (код $code)")
    }
}
```

---

## 4. Способи реєстрації: Статична проти Динамічної

Існує два способи зареєструвати `BroadcastReceiver`:

### Спосіб 1: Статична реєстрація (у `AndroidManifest.xml`)
Приймач оголошується прямо у файлі маніфесту. Завдяки цьому Android може розбудити ваш додаток для обробки події, навіть якщо додаток зараз закритий.

```xml
<receiver 
    android:name=".MyBootReceiver"
    android:exported="true">
    <intent-filter>
        <!-- Реєструємо подібність завантаження ОС -->
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

> ⚠️ **Обмеження Android 8.0+ (API 26):** З міркувань економії батареї Google заборонив статичну реєстрацію в маніфесті для більшості системних бродкастів (окрім винятків на кшталт `BOOT_COMPLETED`). Для більшості подій потрібно використовувати **динамічну реєстрацію**.

---

### Спосіб 2: Динамічна реєстрація (у коді Kotlin)
Приймач реєструється в коді під час роботи додатка (`Activity`, `Service` чи `ViewModel`) і прив'язується до його життєвого циклу.

#### 🛡️ Вимоги безпеки в Android 14+ (API 34):
Починаючи з Android 14, при реєстрації динамічного бродкасту необхідно **обов'язково вказувати прапорець безпеки**:
* **`RECEIVER_EXPORTED`** — якщо бродкаст може надсилатися з інших зовнішніх додатків.
* **`RECEIVER_NOT_EXPORTED`** — якщо бродкаст надсилається тільки всередині вашого додатка чи системою.

```kotlin
class MainActivity : AppCompatActivity() {

    private val airplaneModeReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            if (intent?.action == Intent.ACTION_AIRPLANE_MODE_CHANGED) {
                val isEnabled = intent.getBooleanExtra("state", false)
                Log.d("BR", "Режим у літаку змінено: $isEnabled")
            }
        }
    }

    override fun onStart() {
        super.onStart()
        val filter = IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED)
        
        // Обов'язкова вказівка прапорця для Android 14+ (API 34)
        val listenFlags = ContextCompat.RECEIVER_NOT_EXPORTED
        ContextCompat.registerReceiver(this, airplaneModeReceiver, filter, listenFlags)
    }

    override fun onStop() {
        super.onStop()
        // Обов'язково відписуємося, щоб уникнути Memory Leak!
        unregisterReceiver(airplaneModeReceiver)
    }
}
```

---

## 5. Сучасний Android-підхід: Перетворення `BroadcastReceiver` у `Flow` через `callbackFlow`

У сучасній архітектурі Android (MVVM + Coroutines + Compose) `BroadcastReceiver` перетворюють на реактивний потік **`Kotlin Flow`** за допомогою білдера **`callbackFlow`**.

Це дозволяє елегантно створити джерело даних, яке автоматично реєструє `BroadcastReceiver` при появі підписника та скасовує його через `unregisterReceiver` при зупинці!

### 1. Створення Flow у репозиторії (`AirplaneModeTracker`):

```kotlin
class AirplaneModeTracker(private val context: Context) {

    // Створюємо асинхронний потік стану режиму у літаку
    val isAirplaneModeEnabled: Flow<Boolean> = callbackFlow {
        
        // 1. Створюємо BroadcastReceiver
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context?, intent: Intent?) {
                if (intent?.action == Intent.ACTION_AIRPLANE_MODE_CHANGED) {
                    val isEnabled = intent.getBooleanExtra("state", false)
                    // Емітимо нове значення у потік Flow
                    trySend(isEnabled)
                }
            }
        }

        // 2. Реєструємо receiver
        val filter = IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED)
        ContextCompat.registerReceiver(
            context,
            receiver,
            filter,
            ContextCompat.RECEIVER_NOT_EXPORTED
        )

        // 3. Відсилаємо початкове значення стану
        val initialStatus = Settings.Global.getInt(
            context.contentResolver,
            Settings.Global.AIRPLANE_MODE_ON, 0
        ) != 0
        trySend(initialStatus)

        // 4. КОЛИ ПОТІК ЗАКРИВАЄТЬСЯ (Activity/ViewModel скасовано) — відписуємо receiver!
        awaitClose {
            context.unregisterReceiver(receiver)
        }
    }
}
```

---

### 2. Підключення у `ViewModel`:

```kotlin
class MainViewModel(airplaneModeTracker: AirplaneModeTracker) : ViewModel() {

    // Перетворюємо холодний callbackFlow у гарячий StateFlow для UI
    val isAirplaneMode: StateFlow<Boolean> = airplaneModeTracker.isAirplaneModeEnabled
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = false
        )
}
```

---

### 3. Відображення у Jetpack Compose:

Тепер у Compose ми можемо легко показати попереджувальний баннер, коли ввімкнено режим "У літаку":

```kotlin
@Composable
fun MainScreen(viewModel: MainViewModel = viewModel()) {
    // Зчитуємо стан BroadcastReceiver як Compose State
    val isAirplaneMode by viewModel.isAirplaneMode.collectAsStateWithLifecycle()

    Column(modifier = Modifier.fillMaxSize()) {
        // Якщо режим у літаку увімкнено — показуємо червоний попереджувальний баннер
        if (isAirplaneMode) {
            Surface(
                color = MaterialTheme.colorScheme.errorContainer,
                modifier = Modifier.fillMaxWidth()
            ) {
                Text(
                    text = "⚠️ Увімкнено режим 'У літаку'. Мережеві запити недоступні.",
                    color = MaterialTheme.colorScheme.onErrorContainer,
                    modifier = Modifier.padding(12.dp)
                )
            }
        }

        // Основний контент екрану...
        Text(
            text = "Головний вміст додатка",
            modifier = Modifier.padding(16.dp)
        )
    }
}
```

---

## Підсумок

* **`BroadcastReceiver`** — системний компонент для слухання міжпроцесних та системних подій Android (Airplane Mode, зарядка, мережа).
* **Статична реєстрація** робиться у `AndroidManifest.xml` (обмежена в Android 8.0+), а **Динамічна** — у коді Kotlin через `registerReceiver()` / `unregisterReceiver()`.
* **Android 14+ (API 34)** вимагає обов'язкової вказівки прапорця `RECEIVER_EXPORTED` або `RECEIVER_NOT_EXPORTED`.
* **`callbackFlow`** — найкращий сучасний спосіб обгортання `BroadcastReceiver` у чистий реактивний `Flow` із гарантованим відписуванням у `awaitClose { ... }`.
