# Вступ до Jetpack Compose: Декларативний UI, Composable та Базові Layouts

Ласкаво просимо до світу **Jetpack Compose** — сучасного декларативного UI-фреймворку від Google для побудови нативних інтерфейсів Android. Compose революціонізував розробку, повністю замінивши застарілий View-підхід (`XML` + `findViewById`/`ViewBinding`).

Цей туторіал створено на основі офіційного курсa від Google: [Jetpack Compose for Android Developers](https://developer.android.com/courses/pathways/jetpack-compose-for-android-developers-1).

---

## 1. Декларативний UI проти Імперативного (XML Views)

### Старий підхід (Імперативний / XML Views):
У застарілому підході ви верстали макет у `XML`, а потім у Kotlin-коді вручну знаходили елементи й змінювали їхній стан:
```kotlin
// Імперативно: Ми самі даємо вказівки "ЯК змінити відмальований View"
binding.tvTitle.text = "Новий заголовок"
binding.btnSubmit.isEnabled = true
```

### Сучасний підхід Jetpack Compose (Декларативний):
У Compose немає жодного `XML`-файлу. Ви описуєте інтерфейс прямо у Kotlin-коді, відповідаючи на запитання **"ЩО має бути на екрані при даному стані"**:

$$\text{UI} = f(\text{State})$$

```kotlin
// Декларативно: UI є прямою функцією від Стану (State)
@Composable
fun HeaderScreen(title: String, isEnabled: Boolean) {
    Column {
        Text(text = title)
        Button(onClick = { /* ... */ }, enabled = isEnabled) {
            Text("Надіслати")
        }
    }
}
```
Якщо змінюється `title` або `isEnabled`, Compose **автоматично перемальовує (`Recomposition`)** потрібні елементи екрану!

---

## 2. `@Composable` функції

Наріжним каменем і головним будівельним блоком Jetpack Compose є **Composable-функції** (функції компонування).

### Головна суть: UI-елементи — це Функції!
У традиційному Android розробники звикли, що кожен елемент інтерфейсу — це окремий об'єкт класу (`TextView`, `Button`, `ConstraintLayout`), а їхня верстка лежить в XML-файлах.

У Jetpack Compose **самий елемент графічного інтерфейсу — це Kotlin-функція**, позначена анотацією **`@Composable`**.

* **Повна відмова від XML:** Ви більше не створюєте XML-макети і не використовуєте `ViewBinding`. Усьому інтерфейсу — від найменшої кнопки до цілого екрану — відповідають Composable-функції.
* **Принцип компонування (Composition):** Складні екрани розробляються шляхом комбінування та вкладення менших Composable-функцій одна в одну (наприклад, ваша функція `UserCard()` викликає всередині стандартні `Row()`, `Image()` та `Text()`).

---

### Роль анотації `@Composable`
Анотація `@Composable` — це підказка для плагіна компілятора Kotlin. Вона трансформує звичайну функцію у спеціальний UI-вузол, здатний автоматично перемальовуватися при зміні даних (подібно до того, як ключове слово `suspend` перетворює функцію для корутин).

---

### Ключові особливості Composable-функцій:

1. **Іменуються з Великої літери як Іменники:** 
   Оскільки Composable-функція описує елемент інтерфейсу (а не дію), вона іменується як іменник з великої літери: `Text()`, `Button()`, `UserCard()`, `HomeScreen()`.
2. **Викликаються тільки з інших Composable-функцій:** 
   За аналогією із `suspend` функціями у корутинах, викликати Composable-функцію можна **тільки з іншої Composable-функції** (або усередині блоку `setContent { ... }` в Activity). Саме так будується дерево інтерфейсу: батьківська функція викликає дочірні.
3. **Приймають параметри для конфігурації UI:** 
   Composable-функції можуть приймати будь-які вхідні параметри (наприклад, текст для виводу, кольори, модифікатори стилю `Modifier` чи обробники подій `onClick: () -> Unit`). Саме через ці параметри передаються дані для налаштування зовнішнього вигляду та поведінки елемента.
4. **Є незмінними (Immutable) та не повертають посилань (`Unit`):** 
   Вони не повертають об'єкти `View` і на них **неможливо зберегти посилання** (на відміну від XML, де можна було зберегти `TextView` і пізніше мутувати його через `view.setText(...)`). Composable-функція є незмінною. Щоб змінити вміст UI-елемента, ви не змінюєте його внутрішні властивості, а передаєте нові дані у параметрах під час рекомпозиції.
5. **Автоматично перезапускаються (Recomposition):** 
   Щойно стан чи параметри, які зчитує Composable-функція, змінюються, Compose автоматично викликає її знову для оновлення відповідного фрагмента екрану.
6. **Можуть виконуватися у довільному порядку та паралельно:** 
   Фреймворк Compose сам вирішує, у якій послідовності викликати Composable-функції для оптимізації швидконості, і може виконувати їх на різних фонових потоках.
7. **Без побічних ефектів (Side-Effect Free):** 
   Composable-функція повинна лише описувати UI на основі вхідних параметрів. Усередині неї не можна писати важкі обчислення, запити в базу чи змінювати зовнішній стан програми.

> 💡 **Сила Kotlin для опису UI (Full Kotlin Control Flow)**
>
> Оскільки у Compose весь інтерфейс описується звичайними Kotlin-функціями, ви можете використовувати **ввесь потенціал і синтаксис мови Kotlin** (цикли `for`, умовні оператори `if`/`when`, інтерполяцію рядків та розширення) безпосередньо усередині верстки:
>
> ```kotlin
> @Composable
> fun UserProfileScreen(user: User?) {
>     Column {
>         // Звичайний 'if-else' Kotlin прямо у верстці без View.GONE чи View.VISIBLE!
>         if (user != null) {
>             Text("Вітаємо, ${user.name}!")
>         } else {
>             Button(onClick = { /* login */ }) { 
>                 Text("Увійти до акаунту") 
>             }
>         }
>     }
> }
> ```

---

## 3. Структура проекту та `MainActivity` у Compose

Коли ви створюєте новий проект у Android Studio, обираючи шаблон **Empty Activity (Compose)**, ви одразу помітите відсутність XML-макетів та файлів ViewBinding.

### 1. `ComponentActivity` та точка входу `setContent { ... }`
У `MainActivity.kt` замість застарілого `setContentView(R.layout.activity_main)` використовується функція **`setContent`**:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Точка входу в Jetpack Compose
        setContent {
            MyApplicationTheme { // 1. Обгортка теми додатка (Material Design 3)
                Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding -> // 2. Базовий каркас екрану
                    Greeting(
                        name = "Android",
                        modifier = Modifier.padding(innerPadding) // 3. Враховуємо системні відступи
                    )
                }
            }
        }
    }
}
```

* **`setContent`** — це міст між Android `Activity` та графічним контекстом Jetpack Compose. Усі UI-елементи вашого застосунку викликаються усередині блоку `setContent`.
* **`MyApplicationTheme`** — тема вашого додатка, яка визначає колірну гаму, шрифти та форми системи Material Design 3.
* **`Scaffold`** — це стандартний "каркас" екрану Material Design. Він автоматично обчислює відступи (`innerPadding`) для системних панелей (Status Bar, Navigation Bar) та надає зручні "слоти" для додавання верхньої панелі (`TopAppBar`), нижньої навігації (`BottomBar`) чи плаваючої кнопки (`FloatingActionButton`).
* **`Surface`** — ще один базовий Material-контейнер, який заповнює фон екрану відповідним кольором (`MaterialTheme.colorScheme.background`) і за потреби додає відтінки чи закруглені кути. Його часто використовують як підкладку для окремих карток або усередині `Scaffold`.

### 2. Шаблон першої Composable-функції Greeting

Нижче у файлі `MainActivity.kt` ви побачите стандартний приклад створення власної Composable-функції:

```kotlin
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}
```

* **`@Composable`** — анотація, що сповіщає плагін компілятора Kotlin про те, що ця функція призначена для емісії UI-вузлів.
* **`Greeting`** — назва вашої Composable-функції (пишеться з Великої літери як іменник).
* **`name: String`** — вхідні дані, які ця функція відображатиме на екрані.
* **`modifier: Modifier = Modifier`** — параметр за замовчуванням для налаштування відступів, розмірів та стилю. За рекомендаціями Google, практично всі кастомні Composable-функції повинні приймати `modifier = Modifier`, щоб зовнішній код міг налаштовувати їхнє позиціонування на екрані.
* **`Text(...)`** — стандартна Composable-функція з бібліотеки Compose, яка виводить текстовий рядок на екран (заміна застарілого `TextView`).

### 3. Попередній перегляд у IDE через анотацію `@Preview`

Щоб бачити вигляд інтерфейсу в Android Studio без запуску емулятора на пристрої, використовують анотацію **`@Preview`**. 

Ви додаєте окрему Composable-функцію з цією анотацією, і Android Studio миттєво відмальовує її на панелі **Design / Split** праворуч:

```kotlin
@Preview(showBackground = true, name = "Світла тема")
@Composable
fun GreetingPreview() {
    MyApplicationTheme {
        Greeting("Android")
    }
}
```

До однієї Composable-функції можна застосовувати **декілька анотацій `@Preview` одночасно** (Multi-preview). Це надзвичайно зручно, оскільки дозволяє на панелі Android Studio миттєво бачити вигляди екрану в різних варіантах — наприклад, одночасно для Світлої та Темної теми (Dark Mode):

```kotlin
@Preview(showBackground = true, name = "Світла тема")
@Preview(
    showBackground = true, 
    name = "Темна тема", 
    uiMode = Configuration.UI_MODE_NIGHT_YES
)
@Composable
fun GreetingPreview() {
    MyApplicationTheme {
        Greeting("Android")
    }
}
```

### 4. Папка `ui.theme` (Налаштування дизайну та теми)

У структурі папок вашого проекту ви знайдете папку **`ui.theme`**. У ній зібрано файли, які визначають зовнішній вигляд та стилі всього застосунку за стандартом **Material Design 3**:

* **`Color.kt`** — визначає палітру кольорів додатка (Primary, Secondary, Background тощо).
* **`Type.kt`** — налаштовує типографіку (стилі шрифтів, розміри заголовок та основного тексту `bodyLarge`, `titleLarge`).
* **`Theme.kt`** — головний файл теми, де створюється функція `MyApplicationTheme { ... }`. Вона об'єднує кольори й шрифти та автоматично перемикає оформлення між **Світлою та Темною темою (Dark Mode)**, а також підтримує адаптивні кольори Android 12+ (Dynamic Color).

### 5. Роль ресурсної папки `res/` у Compose

У Compose-проектах папка **`res/`** позбулася обов'язків верстки, але залишилася головним сховищем контентних ресурсів:

* ❌ **Папка `res/layout/` більше не використовується:** Уся верстка пишеться кодом на Kotlin у Composable-функціях.
* ✅ **Що залишається у `res/`:**
  * **`res/drawable/`** — векторні іконки та зображення (зчитуються через `painterResource(R.drawable.ic_logo)`).
  * **`res/values/strings.xml`** — переклади та локалізація текстів (зчитуються через `stringResource(R.string.app_name)`).
  * **`res/mipmap/`** — іконка самого застосунку на робочому столі пристрою (App Icon).
  * **`res/font/`** — файли кастомних шрифтів (`.ttf` / `.otf`).
  * **`res/raw/`** — фонові медіа-файли, аудіо та JSON-анімації.

---

## 4. Базові Layout-компоненти: `Column`, `Row`, `Box`

У застарілому XML за розміщення відповідали `LinearLayout`, `ConstraintLayout` чи `FrameLayout`. У Compose для цього є 3 прості та потужні компоненти:

```
┌────────────────┐     ┌───────────────────────────┐     ┌───────────────┐
│  Column        │     │ Row (Горизонтально)       │     │ Box (Шар над  │
│  (Вертикально) │     │ ┌──────┐ ┌──────┐ ┌──────┐│     │ шар / Z-ось)  │
│  ┌─────────┐   │     │ │ Item │ │ Item │ │ Item ││     │ ┌───────────┐ │
│  │ Item 1  │   │     │ └──────┘ └──────┘ └──────┘│     │ │ Background│ │
│  ├─────────┤   │     └───────────────────────────┘     │ │ ┌───────┐ │ │
│  │ Item 2  │   │                                       │ │ │ Fore- │ │ │
│  └─────────┘   │                                       │ │ │ ground│ │ │
└────────────────┘                                       └─┴─┴───────┴─┴─┘
```

### 1. `Column` (Вертикальне розміщення)
Розміщує свої дочірні елементи один під одним зверху вниз:
```kotlin
Column {
    Text("Перший рядок")
    Text("Другий рядок")
}
```

### 2. `Row` (Горизонтальне розміщення)
Розміщує елементи поруч один з одним зліва направо:
```kotlin
Row {
    Icon(Icons.Default.Star, contentDescription = null)
    Text("Рейтинг: 4.9")
}
```

### 3. `Box` (Накладання шарів один на одного)
Розміщує елементи один над одним у Z-просторі (як `FrameLayout`). Ідеально для накладання іконок, бейджів чи фонових зображень:
```kotlin
Box {
    Image(painter = painterResource(R.drawable.banner), contentDescription = null)
    Text("Текст поверх картинки", modifier = Modifier.align(Alignment.BottomStart))
}
```

---

## 5. Базові UI-елементи Compose

### `Text` (Виведення тексту):
```kotlin
Text(
    text = "Заголовок статті",
    fontSize = 20.sp,
    fontWeight = FontWeight.Bold,
    color = Color.Blue
)
```

### `Button` (Кнопка):
```kotlin
Button(
    onClick = { Log.d("APP", "Кнопку натиснуто!") },
    enabled = true
) {
    Text("Натисни мене")
}
```

### `OutlinedTextField` / `TextField` (Поле введення тексту):
У Compose поле введення не зберігає текст усередині себе самостійно — воно отримує поточний текст у параметр `value` та сповіщає про кожне нове введення через колбек `onValueChange`:

```kotlin
var textState by remember { mutableStateOf("") }

OutlinedTextField(
    value = textState,
    onValueChange = { newText -> textState = newText },
    label = { Text("Введіть ваше ім'я") },
    placeholder = { Text("Наприклад, Олександр") },
    singleLine = true
)
```

### `Image` (Зображення):
```kotlin
Image(
    painter = painterResource(id = R.drawable.avatar),
    contentDescription = "Аватар користувача",
    contentScale = ContentScale.Crop,
    modifier = Modifier.size(64.dp)
)
```

---

## 6. Налаштування зовнішнього вигляду

### 1. Одиниці вимірювання в Android: `dp` та `sp`

При налаштуванні розмірів елементів та текстів у Compose використовуються спеціальні одиниці вимірювання:

* **`dp` (Density-independent Pixels):** Використовується для вимірювання **розмірів елементів, ширини, висоти та відступів** (`padding`). `dp` автоматично адаптується під щільність пікселів екрану (Screen Density) конкретного смартфона, завдяки чому кнопка розміром `48.dp` виглядатиме однаково за фізичним розміром як на маленькому екрані, так і на 4K-дисплеї.
  ```kotlin
  import androidx.compose.ui.unit.dp

  Modifier.padding(16.dp).size(48.dp)
  ```

  #### 📐 Як фізично розраховується `dp`?
  В основі розрахунку `dp` лежить **щільність пікселів екрану (DPI — Dots Per Inch)**, тобто скільки пікселів поміщається в одному фізичному дюймі (2.54 см) дисплея.

  * **Базовий еталон (Baseline):** За еталон в Android взято екран із щільністю **160 dpi** (так званий `mdpi`). На такому екрані **`1 dp = 1 px`**.
  * **Формула розрахунку пікселів з `dp`:**
    $$\text{px} = \text{dp} \times \left( \frac{\text{DPI екрану}}{160} \right)$$
    де вираз у дужках $\left( \frac{\text{DPI екрану}}{160} \right)$ називається **коефіцієнтом щільності екрану** (Screen Density Scale).

  **Приклад для елемента розміром `16 dp` на різних екранах:**
  * **`mdpi` (160 dpi):** Коефіцієнт $1.0$ ➔ $16 \text{ dp} \times 1.0 = \mathbf{16 \text{ px}}$
  * **`xhdpi` (320 dpi):** Коефіцієнт $2.0$ ➔ $16 \text{ dp} \times 2.0 = \mathbf{32 \text{ px}}$
  * **`xxhdpi` (480 dpi, більшість сучасних смартфонів):** Коефіцієнт $3.0$ ➔ $16 \text{ dp} \times 3.0 = \mathbf{48 \text{ px}}$
  * **`xxxhdpi` (640 dpi, 4K екрани):** Коефіцієнт $4.0$ ➔ $16 \text{ dp} \times 4.0 = \mathbf{64 \text{ px}}$

  Система Android під час відмальовки автоматично множить ваші `dp` на коефіцієнт щільності екрану, щоб на будь-якому смартфоні кнопка мала однаковий розмір у міліметрах.

  #### 🛠️ Перевірка параметрів екрану через команди ADB:
  Ви можете дізнатися точну роздільну здатність та щільність екрану підключеного смартфона чи емулятора за допомогою команд `adb`:

  * **Дізнатися фізичну роздільну здатність екрану (у пікселях `px`):**
    ```bash
    adb shell wm size
    ```
    *Результат:* `Physical size: 1080x2400`

  * **Дізнатися щільність екрану (у `DPI`):**
    ```bash
    adb shell wm density
    ```
    *Результат:* `Physical density: 480` *(у даному випадку коефіцієнт щільності дорівнює $480 / 160 = 3.0$)*
* **`sp` (Scale-independent Pixels):** Використовується виключно для вимірювання **розмірів шрифтів та тексту** (`fontSize`). На відміну від `dp`, значення `sp` масштабується не лише від щільності екрану, але й від **налаштувань розміру шрифту**, які користувач обрав у системних налаштуваннях смартфона (наприклад, для людей із порушенням зору).
  ```kotlin
  import androidx.compose.ui.unit.sp

  Text(text = "Заголовок", fontSize = 20.sp)
  ```
* **`px` (Pixels):** Фізичні пікселі дисплея пристрою. У верстці Compose `px` напряму не використовуються, оскільки один і той самий розмір у `px` матиме різний фізичний вигляд на екранах з різною роздільною здатністю.

---

### 2. Модифікатори (`Modifier`)

**`Modifier`** — це універсальний об'єкт у Compose для налаштування розмірів, відступів (`padding`), фонів, вирівнювання та обробки подій (кліків) для будь-якого Composable-елемента.

Ланцюжок модифікаторів застосовується **послідовно (порядок викликів має значення!)**:

```kotlin
Text(
    text = "Привіт, Compose!",
    modifier = Modifier
        .fillMaxWidth()                  // 1. Зайняти всю ширину
        .padding(16.dp)                  // 2. Встановити відступи
        .background(Color.LightGray)     // 3. Зафарбувати фон сірим кольором
        .clickable { /* обробка кліку */ } // 4. Зробити елемент клікабельним
)
```

> ⚠️ **Важливо:** У Compose немає окремого поняття `margin`! Усі зовнішні та внутрішні відступи робляться комбінацією функції `padding()` та порядку її виклику відносно інших модифікаторів (наприклад, `padding` до `background` працює як margin, а `padding` після `background` — як внутрішній padding).

#### 💡 Рекомендація Google: Передача `modifier` у кастомні компоненти
За правилами розробки у Compose, кожна ваша власна Composable-функція повинна приймати параметр `modifier: Modifier = Modifier` зі значенням за замовчуванням і передавати його своєму найпершому (кореневому) UI-елементу. Це дозволяє зовнішньому коду гнучко налаштовувати відступи та позиціонування вашого компонента у будь-якому місці:

```kotlin
// 1. Кастомна функція приймає modifier у параметри
@Composable
fun UserBadge(name: String, modifier: Modifier = Modifier) {
    Row(
        // Обов'язково застосовуємо отриманий modifier до кореневого макету!
        modifier = modifier
            .background(Color.LightGray)
            .padding(8.dp)
    ) {
        Text(text = name)
    }
}

// 2. Виклик ззовні: тепер ви можете легко задати зовнішні відступи компонента!
UserBadge(
    name = "Олександр",
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
)
```

#### 🛠️ Популярні методи Modifier для налаштування UI:

| Категорія | Метод Modifier | Призначення / Опис | Приклад |
| :--- | :--- | :--- | :--- |
| **Розміри** | `fillMaxWidth()` | Заповнює весь доступний простір по ширині. | `Modifier.fillMaxWidth()` |
| | `fillMaxHeight()` | Заповнює весь доступний простір по висоті. | `Modifier.fillMaxHeight()` |
| | `fillMaxSize()` | Заповнює весь контейнер (і ширину, і висоту). | `Modifier.fillMaxSize()` |
| | `width(dp)` / `height(dp)` | Встановлює точну ширину або висоту. | `Modifier.width(100.dp)` |
| | `size(dp)` | Встановлює однаковий розмір (квадрат). | `Modifier.size(48.dp)` |
| **Відступи** | `padding(dp)` | Встановлює відступи з усіх 4 боків. | `Modifier.padding(16.dp)` |
| | `padding(horizontal, vertical)` | Встановлює відступи окремо по горизонталі та вертикалі. | `Modifier.padding(horizontal = 16.dp, vertical = 8.dp)` |
| **Вирівнювання** | `align(Alignment)` | Вирівнює елемент усередині `Box` чи `Row`/`Column`. | `Modifier.align(Alignment.Center)` |
| | `weight(float)` | Задає пропорційну вагу елемента усередині `Row`/`Column`. | `Modifier.weight(1f)` |
| **Зовнішній вигляд**| `background(color, shape)` | Встановлює колір фону та форму компонента. | `Modifier.background(Color.White)` |
| | `border(width, color, shape)` | Малює рамку / обводку навколо елемента. | `Modifier.border(1.dp, Color.Black)` |
| | `clip(shape)` | Обрізає елемент за формою (`CircleShape`, `RoundedCornerShape(8.dp)`). | `Modifier.clip(CircleShape)` |
| | `shadow(elevation, shape)` | Малює тінь під елементом (ефект підняття). | `Modifier.shadow(4.dp)` |
| **Події** | `clickable { ... }` | Робить елемент клікабельним з анімацією Ripple. | `Modifier.clickable { doSomething() }` |

---

### 3. Встановлення кольорів

У Compose для роботи з кольорами використовується клас `androidx.compose.ui.graphics.Color`. 

Задавати кольори можна двома основними шляхами:

#### A. Стандартні константи (`Color.Green`, `Color.Red`...)
У класі `Color` уже заготовлено базові статичні кольори: `Color.Red`, `Color.Green`, `Color.Blue`, `Color.Yellow`, `Color.Black`, `Color.White`, `Color.Gray`, `Color.Transparent` тощо.

```kotlin
Text(text = "Зелений текст", color = Color.Green)
```

#### B. Власні кольори через Шістнадцятковий код (HEX / ARGB)
Щоб задати власний кастомний колір (наприклад, дизайн з Figma), використовується конструктор `Color(0xFF...)`.

> ⚠️ **Увага:** Конструктор `Color` приймає 8-значний HEX-код у форматі **ARGB** (Alpha + Red + Green + Blue):
> * Перші 2 цифри після `0x` — це **прозорість (Alpha)**: `FF` відповідає за 100% непрозорий колір, `80` — за 50% прозорості.
> * Наступні 6 цифр — це стандартний HEX-код кольору (`RRGGBB`).

```kotlin
// 1. Оранжевий колір (#FF5722) -> додаємо 0xFF попереду:
val myOrange = Color(0xFFFF5722)

// 2. Зелений колір (#4CAF50) -> додаємо 0xFF:
val myGreen = Color(0xFF4CAF50)

// 3. Напівпрозорий червоний (50% прозорості -> 0x80):
val semiTransparentRed = Color(0x80FF0000)
```

#### C. Семантичні кольори теми (`MaterialTheme.colorScheme`) та автоконтраст `Surface`

У реальних Android-проектах кольори рідко кодують жорстко через `Color(0xFF...)`. Замість цього використовують **семантичну палітру теми Material Design 3** — `MaterialTheme.colorScheme`:

* **`MaterialTheme.colorScheme.primary`** — головний фірмовий колір вашого бренду (для активних кнопок та елементів).
* **`MaterialTheme.colorScheme.onPrimary`** — контрастний колір для тексту/іконок, що лежать *поверх* primary.
* **`MaterialTheme.colorScheme.background`** — колір фону всього екрану.
* **`MaterialTheme.colorScheme.surface`** — колір фону карток, діалогів та меню.

> 💡 **Чому це важливо?** Завдяки використанню `MaterialTheme.colorScheme`, при увімкненні **Темної теми (Dark Mode)** у системі ваш додаток **автоматично перемикає всі кольори** на їхні темні варіанти із файлу `ui.theme/Theme.kt` без жодного редагування верстки!

##### Магія автоконтрасту контейнера `Surface`:
Коли ви огортаєте елементи в контейнер `Surface(color = MaterialTheme.colorScheme.primary)`, `Surface` не лише зафарбовує фон, але й **автоматично встановлює колір усього тексту та іконок всередині себе на контрастний** (`onPrimary`):

```kotlin
// Всі елементи Text всередині Surface автоматично стануть контрастно-білими!
Surface(color = MaterialTheme.colorScheme.primary) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Заголовок екрану") // Колір шрифту підбирається автоматично!
        Text("Опис екрану")
    }
}
```

---

#### Де застосовуються кольори у компонентах:

* **Колір ФОНУ встановлюється через `Modifier.background()`:**
  ```kotlin
  // Суцільний кастомний колір фону
  Text("Текст на кастомному тлі", modifier = Modifier.background(Color(0xFFE0E0E0)))

  // Фон із закругленими кутами
  Row(modifier = Modifier.background(color = Color.Green, shape = RoundedCornerShape(8.dp))) { ... }
  ```

* **Колір ТЕКСТУ, ІКОНОК та КНОПОК передається у параметрах самого елемента:**
  ```kotlin
  // Колір тексту передається в параметр color
  Text(text = "Синій заголовок", color = Color.Blue)

  // Колір іконки передається в параметр tint
  Icon(Icons.Default.Favorite, contentDescription = null, tint = Color(0xFFFF1744))

  // Колір кнопки передається у параметр colors
  Button(
      onClick = { /* ... */ }, 
      colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF4CAF50))
  ) {
      Text("Зелена кнопка")
  }
  ```

#### 💡 Чому для кнопок використовується `ButtonDefaults.buttonColors(...)`?

На відміну від тексту чи картинки, кнопка в Material Design є складним елементом, який має **різні стани** (активна кнопка vs вимкнена `enabled = false`) та **різні шари кольорів** (колір самого фону кнопки та колір тексту/іконки всередині неї).

Об'єкт **`ButtonDefaults.buttonColors(...)`** — це спеціальний допоміжний клас Material 3, який дозволяє гнучко налаштувати кольори для всіх станів кнопки:

```kotlin
Button(
    onClick = { /* ... */ },
    enabled = true,
    colors = ButtonDefaults.buttonColors(
        containerColor = Color(0xFF4CAF50),     // Колір фону активної кнопки (зелений)
        contentColor = Color.White,             // Колір тексту/іконки всередині активної кнопки
        disabledContainerColor = Color.LightGray,// Колір фону, коли enabled = false
        disabledContentColor = Color.DarkGray    // Колір тексту, коли enabled = false
    )
) {
    Text("Зберегти")
}
```

* Якщо ви вказуєте тільки `containerColor = Color(...)`, системи Material 3 автоматично підбере контрастний колір для тексту та вимкненого стану за замовчуванням.

---

## Підсумок

Головні принципи філософії Jetpack Compose зводяться до 4 основних правил:

1. **Описуйте "ЩО", а не "ЯК" (Describe what, not how):** Ви декларативно описуєте, яким має бути UI при певних даних, замість того, щоб давати інструкції "як його змінити". Compose сам керує процесом малювання та оновлення.
2. **Елементи UI — це функції (UI elements are functions):** Будь-який елемент інтерфейсу створюється як звичайна Kotlin-функція з анотацією `@Composable` (іменуються з великої літери).
3. **Стан керує UI (State controls UI):** Інтерфейс є прямою функцією від даних: $\text{UI} = f(\text{State})$. Будь-яка зміна стану автоматично запускає перемальовування (рекомпозицію) відповідних UI-елементів.
4. **Події керують Станом (Events control State):** Дії користувача (кліки, введення тексту, свайпи) генерують Події (Events), які змінюють Стан (State), що у свою чергу оновлює UI.

---

### Основні інструменти:
* **`Column`**, **`Row`**, **`Box`** — базові макети для вертикального, горизонтального та Z-накладання елементів.
* **`Modifier`** — універсальний ланцюжок налаштувань розмірів, відступів, кліків, фонів та вирівнювання.
* **`@Preview`** — анотація для миттєвого перегляду дизайну в Android Studio без запуску емулятора.
