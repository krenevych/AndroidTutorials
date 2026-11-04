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

### З чого складається HTTP-запит до API?
Щоб зробити правильне "замовлення" до сервера, ваш додаток формує мережевий запит (HTTP Request). У ньому є 3 ключові складові:
1. **URL (Адреса):** Вказує, до якого саме сервера і до якого конкретного ресурсу ми звертаємося (наприклад, `https://api.example.com/users`).
2. **Метод (Тип дії):**
   * **`GET`** — просто "дай мені дані" (найчастіший запит, наприклад, стягнути список новин).
   * **`POST`** — "візьми мої дані і збережи" (наприклад, реєстрація користувача, відправка коментаря або фотографії).
   * **`PUT` / `PATCH`** — "онови існуючі дані".
   * **`DELETE`** — "видали ці дані на сервері".
3. **Заголовки та Тіло (Headers & Body):** 
   * У *Заголовки* зазвичай кладуть секретні токени авторизації (щоб сервер знав, хто ви) або вказують формат даних (`Content-Type: application/json`).
   * У *Тіло* (Body) кладуть самі дані, які ви відправляєте на сервер (наприклад, при `POST` запиті ви кладете туди логін і пароль).

### З чого складається Відповідь від API?
Коли сервер обробить ваш запит, він повертає відповідь (HTTP Response), яка містить:
1. **Статус-код (Status Code):** Числовий код успіху або провалу.
   * `200 OK` — усе чудово, дані знайдено!
   * `400 Bad Request` — ви відправили неправильні дані.
   * `401 Unauthorized` — ви забули додати токен авторизації.
   * `404 Not Found` — такого URL на сервері не існує.
   * `500 Internal Server Error` — сервер "впав" (помилка на боці бекенду).
2. **Тіло відповіді (Body):** Власне сам текст у форматі JSON, який містить те, що ви просили (наприклад, список користувачів).

Ось кілька прикладів популярних публічних API, які ви можете використовувати у своїх пет-проектах (домашніх розробках):

* **JSONPlaceholder** (`https://jsonplaceholder.typicode.com/users`) — безкоштовний фейковий API для навчання та тестування додатків. Повертає готові масиви користувачів, постів та коментарів.
* **npoint.io** (`https://www.npoint.io/`) — чудовий безкоштовний сервіс для створення власного фейкового API. Ви просто вставляєте туди свій JSON-файл, і сервіс генерує для вас унікальний URL для тестування.
* **ПриватБанк API** (`https://api.privatbank.ua/`) — відкрите API, яке миттєво повертає актуальний курс валют в Україні.
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
Щоб працювати з JSON у Kotlin, ми повинні перетворити текст на повноцінні програмні об'єкти. Для цього використовується концепція **POJO (Plain Old Java Object)**, яка у світі Kotlin реалізується через **Data-класи**. Назви властивостей зіставляються з ключами JSON; якщо вони відрізняються, можна вказати відповідність анотацією.

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
import com.google.gson.annotations.SerializedName

data class UserProfile(
    val id: Int,
    
    @SerializedName("first_name")
    val firstName: String,
    
    @SerializedName("is_active")
    val isActive: Boolean
)
```

**Що тут відбувається:**
1. **Gson** перетворює JSON на об'єкти цього класу під час роботи застосунку. Для цього не потрібен окремий плагін компілятора чи анотація перед класом.
2. **Анотація `@SerializedName`:** У JSON часто використовують snake_case (`"first_name"`), а для властивостей Kotlin — lowerCamelCase (`firstName`). `@SerializedName("first_name")` указує Gson, який ключ JSON відповідає властивості `firstName`.
3. Якщо ключ JSON і назва властивості Kotlin **збігаються** (як у випадку з `id`), анотація `@SerializedName` не потрібна — Gson зіставить їх автоматично.

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

## 4. Сучасна архітектура: Retrofit + OkHttp

Щоб розв'язати проблему ручного мережевого запиту (описаного в Розділі 1) та необхідності вручну парсити JSON, Android-розробники використовують золотий стандарт — потужну зв'язку бібліотек:

1. **Retrofit:** Це зручна "обгортка" (від компанії Square). Ви просто створюєте Kotlin-інтерфейс і позначаєте його методи спеціальними анотаціями (наприклад, `@GET("users")`). Retrofit сам генерує весь необхідний код під капотом для здійснення реального запиту.
2. **OkHttp:** Це потужний "двигун" під капотом Retrofit, який безпосередньо виконує HTTP-запити. Саме він керує з'єднаннями, таймаутами та логуванням.
3. **Конвертер Gson:** Це бібліотека, яку підключаємо до Retrofit для автоматичного перетворення JSON із сервера на Kotlin Data-класи (POJO об'єкти), з якими працюємо в коді.

Першоджерела для вивчення Retrofit:

* **[Офіційна документація Retrofit](https://lysine.dev/retrofit/)** — опис анотацій, налаштування та приклади використання.
* **[Репозиторій Retrofit на GitHub](https://github.com/lysine-dev/retrofit)** — вихідний код, README та історія змін.

> ⚠️ **Критичне правило (NetworkOnMainThreadException):**
> Операційна система Android забороняє виконувати блокувальні мережеві операції у головному потоці (Main Thread). Наприклад, `url.readText()` потрібно викликати у фоновому потоці. У нашому прикладі Retrofit запускатиме запит асинхронно через `Call.enqueue()`: мережеву роботу виконає OkHttp, а результат отримаємо через callback.

---

## 5. Практичний приклад: поточні курси валют ПриватБанку

Створимо екран, який завантажує поточні **готівкові курси валют ПриватБанку** до гривні та відображає купівлю й продаж у Compose. Запит виконуватиметься при створенні ViewModel та після натискання кнопки оновлення. Пройдемо всі кроки: від дозволу на Інтернет і залежностей до обробки станів завантаження, успіху та помилки.

* **[Документація API ПриватБанку](https://api.privatbank.ua/#/p24/exchange)** — опис запиту та полів відповіді.
* **[Поточний готівковий курс у JSON](https://api.privatbank.ua/p24api/pubinfo?json&exchange&coursid=5)** — адреса, яку можна відкрити у браузері для перевірки відповіді. Для цього публічного запиту не потрібні API-ключ чи авторизація.

Параметр `coursid=5` обирає готівковий курс, а `coursid=11` — безготівковий (для конвертації за картками та в Приват24). У нашому прикладі використовуватимемо `5`.

### 5.1. Дозволи та залежності

#### Дозвіл на Інтернет (Маніфест)
Відкрийте `AndroidManifest.xml` і додайте дозвіл на вихід в Інтернет над тегом `<application>`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

#### Залежності (Gradle)

Для роботи з API додайте дві залежності у `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:3.0.0")
    implementation("com.squareup.retrofit2:converter-gson:3.0.0")
}
```

Перша підключає Retrofit, друга — конвертер Gson для перетворення JSON на Kotlin-об'єкти. OkHttp та Gson підтягнуться автоматично як транзитивні залежності. Додаткові Gradle-плагіни для цього не потрібні.

**Версії `retrofit` та `converter-gson` мають збігатися:** у цьому прикладі обидві — `3.0.0`. Під час оновлення змінюйте версію в обох залежностях разом.

Виконайте **Sync Project with Gradle Files**. Приклад орієнтований на сучасний Compose-шаблон Android Studio. Стандартні імпорти Android, Compose та корутин у прикладах опущено — їх можна додати підказками Android Studio.

---

### 5.2. Створення Data-класів для API

API повертає **масив об'єктів**, кожен із яких містить курс однієї валюти. Приклад структури відповіді (значення курсів наведені для ілюстрації та змінюються з часом):

```json
[
  {
    "ccy": "EUR",
    "base_ccy": "UAH",
    "buy": "50.70000",
    "sale": "51.70000"
  },
  {
    "ccy": "USD",
    "base_ccy": "UAH",
    "buy": "45.50000",
    "sale": "46.10000"
  }
]
```

Зіставимо поля JSON із властивостями Kotlin:

| Поле JSON | Властивість Kotlin | Значення |
| --- | --- | --- |
| `ccy` | `currency` | Код валюти, наприклад `USD` або `EUR` |
| `base_ccy` | `baseCurrency` | Базова валюта — `UAH` |
| `buy` | `buy` | Скільки гривень банк платить за купівлю однієї одиниці валюти |
| `sale` | `sale` | Скільки гривень банк бере за продаж однієї одиниці валюти |

**Купівля та продаж вказані з погляду банку:** якщо ви здаєте долари банку, застосовується `buy`; якщо купуєте долари в банку — `sale`.

Створюємо файл `CurrencyItem.kt`. За допомогою `@SerializedName` зіставляємо назви полів API зі зрозумілими назвами властивостей Kotlin:

```kotlin
import com.google.gson.annotations.SerializedName

data class CurrencyItem(
    @SerializedName("ccy")
    val currency: String,

    @SerializedName("base_ccy")
    val baseCurrency: String,

    val buy: String,
    val sale: String
)
```

Зверніть увагу: API передає `buy` і `sale` **як рядки в лапках**, тому у моделі це `String`. Для відображення збережемо значення як є. Якщо потрібно обчислювати суми, їх можна перетворити через `toBigDecimal()` і визначити правила округлення; `Double` для точних грошових розрахунків не підходить.

---

### 5.3. Опис API Інтерфейсу

#### Як влаштований GET-запит?

**GET** — це HTTP-метод для отримання даних із сервера. У нашому випадку ми просимо банк повернути курси валют. Метод визначає дію, а URL — адресу ресурсу та параметри вибірки:

```text
GET https://api.privatbank.ua/p24api/pubinfo?json&exchange&coursid=5
```

Розберімо цю адресу на частини:

| Частина URL | Що означає |
| --- | --- |
| `https://` | Схема з'єднання: HTTP із шифруванням TLS |
| `api.privatbank.ua` | Домен сервера, до якого звертаємося |
| `/p24api/pubinfo` | Шлях до ресурсу на сервері |
| `?` | Початок параметрів запиту (query parameters) |
| `json` | Параметр формату відповіді JSON у цьому API |
| `exchange` | Параметр запиту курсів валют у цьому API |
| `&` | Роздільник між параметрами |
| `coursid=5` | Параметр `coursid` зі значенням `5`: готівковий курс |

Параметри часто мають вигляд `ім'я=значення`. У цьому API `json` та `exchange` записані без `=значення` — як прапорці. Назви параметрів та їхнє значення визначає документація конкретного API: вони не є стандартними командами HTTP.

**У нашого запиту немає тіла (Body):** усі параметри передаємо в URL після `?`. Заголовки — окрема частина HTTP-запиту. Наприклад, у спрощеному вигляді для HTTP/1.1 запит можна записати так:

```http
GET /p24api/pubinfo?json&exchange&coursid=5 HTTP/1.1
Host: api.privatbank.ua
Accept: application/json

```

Тут `Host` указує сервер, а `Accept` повідомляє, що клієнт очікує JSON. `Content-Type` описував би формат тіла, яке ми надсилаємо; у цьому GET-запиті тіла немає, тому цей заголовок не потрібен. Формат відповіді визначає сервер відповідно до свого API. Заголовок авторизації для цього публічного запиту також не потрібен.

#### Як описати цей запит у Retrofit?

Retrofit дозволяє описати запит через Kotlin-інтерфейс. Розділимо повну адресу на базову частину, шлях і параметри:

```text
Базова адреса:   https://api.privatbank.ua/p24api/
Відносний шлях:  pubinfo
Сталі параметри: json&exchange
Змінний параметр: coursid=5
```

Базову адресу налаштуємо через `.baseUrl(...)` у підрозділі 5.4. Зараз створюємо файл `PrivatBankApi.kt`: у `@GET` записуємо шлях зі сталими параметрами, а через `@Query` передаємо змінний параметр `coursid`:

```kotlin
import retrofit2.Call
import retrofit2.http.GET
import retrofit2.http.Query

interface PrivatBankApi {

    // GET https://api.privatbank.ua/p24api/pubinfo?json&exchange&coursid=5
    @GET("pubinfo?json&exchange")
    fun getCurrencyExchange(
        @Query("coursid") courseId: Int = 5
    ): Call<List<CurrencyItem>?>?
}
```

* **`@GET("pubinfo?json&exchange")`** задає HTTP-метод, відносний шлях та сталі параметри запиту.
* **`@Query("coursid")`** додає до URL параметр із серверною назвою `coursid`. Kotlin-параметр `courseId` може мати іншу назву; `= 5` задає його значення за замовчуванням.
* **`Call<List<CurrencyItem>?>?`** описує об'єкт мережевого виклику. Gson перетворить масив `[...]` із JSON на список `CurrencyItem`, який отримаємо через відповідь. Внутрішній `?` дозволяє списку бути `null`, а зовнішній — самому об'єкту `Call`.

Виклик `getCurrencyExchange()` створить `Call` для запиту з `coursid=5`, а `getCurrencyExchange(courseId = 11)` — із `coursid=11`. **Сам виклик методу ще не відправляє запит:** для асинхронного виконання потрібно викликати `enqueue()`. Retrofit та OkHttp самі зберуть HTTP-запит; вручну писати рядки `GET ... HTTP/1.1` та `Host` у Kotlin не потрібно. Заголовок `Accept` вище показаний для пояснення будови HTTP-запиту; наш приклад працює без його явного додавання.

> 💡 `enqueue()` приймає `Callback` із двома методами: `onResponse()` для отриманої HTTP-відповіді та `onFailure()` для помилки виконання запиту. HTTP-коди `400`, `404` чи `500` теж потрапляють у `onResponse()`, тому статус потрібно перевіряти окремо.

---

### 5.4. Ініціалізація Retrofit

У файлі `PrivatBankNetworkModule.kt` створимо клас, який налаштовує Retrofit та надає реалізацію `PrivatBankApi` через властивість `service`. Запит викликатимемо у ViewModel:

```kotlin
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

class PrivatBankNetworkModule {

    private val BASE_URL = "https://api.privatbank.ua/p24api/"

    private val retrofit: Retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    val service: PrivatBankApi = retrofit.create(PrivatBankApi::class.java)
}
```

Базова адреса Retrofit має завершуватися `/`. Разом із шляхом `pubinfo` та параметрами вона утворює повний URL із попереднього підрозділу. `GsonConverterFactory.create()` підключає перетворення JSON на моделі даних. Retrofit створить OkHttp-клієнт автоматично.

Властивість `service` доступна з ViewModel: через неї викличемо `getCurrencyExchange(5)`, а отриманий `Call` запустимо через `enqueue()`.

---

### 5.5. Виклик у ViewModel та відображення у Compose

Опишемо стан екрана через sealed-інтерфейс: завантаження, успішно отримані курси або помилка. Кожен наступний блок розміщуємо в окремому Kotlin-файлі з відповідною назвою в тому самому пакеті, що й попередні класи прикладу.

#### 1. State (Стан):
```kotlin
sealed interface ExchangeRatesUiState {
    object Loading : ExchangeRatesUiState
    data class Success(val rates: List<CurrencyItem>) : ExchangeRatesUiState
    data class Error(val message: String) : ExchangeRatesUiState
}
```

#### 2. ViewModel:

Створимо екземпляр `PrivatBankNetworkModule`. У ViewModel викличемо `networkModule.service.getCurrencyExchange(5)`, запустимо запит через `enqueue()` та оновимо стан екрана у callback:

```kotlin
import retrofit2.Call
import retrofit2.Callback
import retrofit2.Response

class ExchangeRatesViewModel : ViewModel() {

    private val networkModule = PrivatBankNetworkModule()

    private val _uiState = MutableStateFlow<ExchangeRatesUiState>(ExchangeRatesUiState.Loading)
    val uiState: StateFlow<ExchangeRatesUiState> = _uiState.asStateFlow()

    private var currentCall: Call<List<CurrencyItem>?>? = null

    init {
        fetchRates()
    }

    fun fetchRates() {
        currentCall?.cancel()
        _uiState.value = ExchangeRatesUiState.Loading

        val call = networkModule.service.getCurrencyExchange(5)
        currentCall = call

        if (call == null) {
            _uiState.value = ExchangeRatesUiState.Error("Не вдалося створити запит.")
            return
        }

        call.enqueue(object : Callback<List<CurrencyItem>?> {
            override fun onResponse(
                call: Call<List<CurrencyItem>?>,
                response: Response<List<CurrencyItem>?>
            ) {
                if (call !== currentCall || call.isCanceled) return

                val rates = response.body()
                _uiState.value = when {
                    !response.isSuccessful -> ExchangeRatesUiState.Error(
                        "Сервер повернув HTTP ${response.code()}. Спробуйте пізніше."
                    )
                    rates == null -> ExchangeRatesUiState.Error(
                        "Банк повернув відповідь без даних."
                    )
                    else -> ExchangeRatesUiState.Success(rates)
                }
            }

            override fun onFailure(call: Call<List<CurrencyItem>?>, throwable: Throwable) {
                if (call !== currentCall || call.isCanceled) return

                _uiState.value = ExchangeRatesUiState.Error(
                    "Не вдалося отримати курси. Спробуйте ще раз."
                )
            }
        })
    }

    override fun onCleared() {
        currentCall?.cancel()
        super.onCleared()
    }
}
```

У `onResponse()` перевіряємо `response.isSuccessful` (HTTP-статус `2xx`) та отримуємо список через `response.body()`. Тіло відповіді може бути `null`; порожній список `[]` при цьому є коректним результатом, який окремо відобразить UI. У `onFailure()` обробляємо помилки з'єднання, читання відповіді або перетворення JSON.

`currentCall` зберігає поточний виклик. Перед оновленням скасовуємо попередній, а в `onCleared()` — незавершений запит при знищенні ViewModel. Callback ігнорує скасовані та застарілі виклики. Кожне оновлення створює новий `Call`: один екземпляр можна виконати лише один раз.

#### 3. Compose UI:
```kotlin
@Composable
fun ExchangeRatesScreen(viewModel: ExchangeRatesViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Text(
            text = "Готівковий курс ПриватБанку",
            style = MaterialTheme.typography.titleLarge
        )

        when (val state = uiState) {
            ExchangeRatesUiState.Loading -> {
                CircularProgressIndicator()
            }
            is ExchangeRatesUiState.Error -> {
                Text(text = state.message, color = MaterialTheme.colorScheme.error)
                Button(onClick = { viewModel.fetchRates() }) {
                    Text("Спробувати знову")
                }
            }
            is ExchangeRatesUiState.Success -> {
                Button(onClick = { viewModel.fetchRates() }) {
                    Text("Оновити курси")
                }

                if (state.rates.isEmpty()) {
                    Text("Банк не повернув курсів валют.")
                } else {
                    LazyColumn(
                        modifier = Modifier.fillMaxWidth().weight(1f),
                        verticalArrangement = Arrangement.spacedBy(16.dp)
                    ) {
                        items(state.rates) { rate ->
                            Column {
                                Text(
                                    text = "${rate.currency} / ${rate.baseCurrency}",
                                    style = MaterialTheme.typography.titleMedium
                                )
                                Text("Банк купує: ${rate.buy} ${rate.baseCurrency}")
                                Text("Банк продає: ${rate.sale} ${rate.baseCurrency}")
                            }
                        }
                    }
                }
            }
        }
    }
}
```

Викличте `ExchangeRatesScreen()` у Compose-вмісті вашого екрана всередині теми застосунку. `viewModel()` отримає або створить ViewModel для поточного власника, а `collectAsStateWithLifecycle()` збиратиме її стан з урахуванням життєвого циклу. Звичайна рекомпозиція не створює новий мережевий запит: початкове завантаження запускається в `init` ViewModel, а наступні — кнопками. Це одноразові запити за поточним курсом, без автоматичного опитування сервера.

## Підсумок
* **`Retrofit`** — зручний генератор мережевих викликів на основі Kotlin-інтерфейсів.
* **`Call.enqueue()`** виконує запит асинхронно та повертає результат через `Callback`; HTTP-статус перевіряємо в `onResponse()`.
* Завжди додавайте дозвіл **`INTERNET`** в маніфест.
* **`GsonConverterFactory`** перетворює JSON на моделі даних, а **`@SerializedName`** зіставляє різні назви полів JSON та властивостей Kotlin.
* Огортайте UI стан у **Sealed Class** (`Loading`, `Success`, `Error`), щоб легко керувати відображенням на екрані за допомогою Compose `when`.
