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

## 3. Базові Layout-компоненти: `Column`, `Row`, `Box`

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

## 4. Базові UI-елементи Compose

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

## 5. Модифікатори (`Modifier`): Налаштування зовнішнього вигляду

**`Modifier`** — це універсальний об'єкт для налаштування розмірів, відступів (padding), фонів, обробок кліків та вирівнювання будь-якого Composable-елемента.

Ланцюжок модифікаторів застосовується **послідовно (порядок має значення!)**:

```kotlin
Text(
    text = "Привіт, Compose!",
    modifier = Modifier
        .fillMaxWidth()                  // Зайняти всю ширину
        .padding(16.dp)                  // Внутрішній/зовнішній відступ
        .background(Color.LightGray)     // Колір фону
        .clickable { /* обробка кліку */ } // Клікабельність
)
```

> ⚠️ **Важливо:** У Compose немає окремого `margin`! Відступи робляться комбінацією `padding()` та порядку виклику модифікаторів.

---

## 6. Попередній перегляд у IDE (`@Preview`)

Щоб бачити вигляд інтерфейсу в Android Studio без запуску емулятора, використовують анотацію **`@Preview`**:

```kotlin
@Preview(showBackground = true, name = "Світла тема")
@Composable
fun UserCardPreview() {
    UserCard(name = "Олександр", role = "Android Developer")
}
```

---

## Повний приклад картки користувача:

```kotlin
@Composable
fun UserCard(name: String, role: String) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Image(
                painter = painterResource(id = R.drawable.ic_launcher_foreground),
                contentDescription = null,
                modifier = Modifier
                    .size(48.dp)
                    .clip(CircleShape)
            )
            Spacer(modifier = Modifier.width(16.dp))
            Column {
                Text(text = name, fontWeight = FontWeight.Bold, fontSize = 18.sp)
                Text(text = role, color = Color.Gray, fontSize = 14.sp)
            }
        }
    }
}
```

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
