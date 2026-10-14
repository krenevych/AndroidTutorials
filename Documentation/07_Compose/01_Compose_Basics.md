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

## 2. Що таке `@Composable` функції?

Наріжним каменем Jetpack Compose є **Composable-функції** (функції компонування), позначені анотацією **`@Composable`**.

### Правила написання Composable-функцій:
1. Позначені анотацією `@Composable`.
2. Іменуються з **великої літери** (наприклад, `GreetingScreen()`, `UserCard()`, `PrimaryButton()`).
3. Нічого не повертають (`Unit`) — вони описують UI-вузли, які додаються в дерево інтерфейсу.
4. Є швидкими та idempotent (однаковий `State` на вході ➔ однаковий `UI` на виході).

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
