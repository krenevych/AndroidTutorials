# Управління станом у Compose: State, Recomposition та State Hoisting

У цьому туторіалі ми зануримося у найважливіший концепт Jetpack Compose — **Управління станом (State Management)**. Ви дізнаєтеся, як Compose оновлює графічний інтерфейс, як правильно зберігати дані при повороті екрану та як проектувати гнучкі й піддатливі тестуванню Composable-компоненти.

---

## 1. Що таке Стан (State) та Рекомпозиція (Recomposition)?

**Стан (State)** у Compose — це будь-яке значення, яке може змінюватися з часом (наприклад, текст у полі введення, статус завантаження, відкрита/закрита картка чи рахунок у грі).

### Що таке Рекомпозиція (Recomposition)?
У застарілому View-підході ви вручну змінювали текст чи колір кнопки (`binding.tv.text = "..."`). У Compose UI є декларативним:

$$\text{UI} = f(\text{State})$$

Коли стан змінюється, Compose автоматично повторно викликає ті Composable-функції, які залежать від цього стану. Цей процес називається **Рекомпозицією (Recomposition)**.

> 💡 **Оптимізація Compose:** Compose дуже розумний — під час рекомпозиції він **не перемальовує весь екран**, а оновлює **тільки ті елементи**, дані для яких дійсно змінилися (Smart Recomposition).

---

## 2. Створення стану: `mutableStateOf` та `remember`

Щоб Compose міг відстежувати зміни змінної, вона повинна бути загорнута у спеціальний об'єкт `State<T>`.

### 1. `mutableStateOf`
Функція `mutableStateOf(value)` створює спостережуваний об'єкт `MutableState<T>`. Коли ви змінюєте його значення, Compose сповіщається і запускає рекомпозицію.

### 2. Ключове слово `remember`
Якщо просто написати `val count = mutableStateOf(0)`, то при кожній рекомпозиції ця змінна буде **створюватися заново з початковим 0**!

Щоб зберігати стан між рекомпозиціями, використовується функція **`remember`**:

```kotlin
// Зберігає стан у пам'яті під час рекомпозиції
val countState = remember { mutableStateOf(0) }
```

### 2 основні синтаксичні варіанти роботи зі станом у Kotlin:

```kotlin
// Варіант 1: Через властивість .value (Тип змінної: MutableState<Int>)
val countState = remember { mutableStateOf(0) }
countState.value++ // Зміна значення через .value

// Варіант 2: Через делегат 'by' (Найзручніший та найпопулярніший спосіб!)
var count by remember { mutableStateOf(0) }
count++ // Працюємо як зі звичайною змінною!
```

> ⚠️ **Важливо:** Для використання синтаксису `var count by remember` потрібні імпорти:
> `import androidx.compose.runtime.getValue`
> `import androidx.compose.runtime.setValue`

---

## 3. Збереження стану при повороті екрану: `rememberSaveable`

Функція `remember` зберігає стан під час рекомпозиції, **АЛЕ втрачає його при зміні конфігурації екрану** (поворот пристрою, зміна теми чи мови).

Щоб стан виживав під час повороту екрану та тимчасового знищення процесу системою, використовують **`rememberSaveable`**:

```kotlin
// Зберігає стан навіть при повороті екрану!
var isExpanded by rememberSaveable { mutableStateOf(false) }
```

`rememberSaveable` автоматично зберігає значення у системному `Bundle` (працює для багатьох базових типів: `Boolean`, `Int`, `String` тощо).

---

## 4. Підйом стану (State Hoisting): Stateful vs Stateless

У Compose прийнято ділити компоненти на два типи:
* **Stateful (Зі станом)** — компонент, який сам керує своїм внутрішнім станом через `remember`.
* **Stateless (Без стану)** — компонент, який не має внутрішнього стану, а отримує значення та колбек-функцію через параметри.

### Що таке State Hoisting (Підйом стану)?
**State Hoisting** — це патерн переміщення стану вище за дерево компонентів (до батьківської функції).

Замість того, щоб робити компонент *Stateful*, його стан "піднімають" нагору, передаючи у параметрах:
1. `value: T` — поточне значення для відображення.
2. `onValueChange: (T) -> Unit` — подія (лямбда), яка сповіщає про прохання змінити значення.

#### Приклад State Hoisting:

```kotlin
// 1. Stateless компонент (Чистий UI, ідеальний для перевикористання та тестування)
@Composable
fun CounterContent(
    count: Int,
    onIncrement: () -> Unit
) {
    Button(onClick = onIncrement) {
        Text("Натиснуто: $count разів")
    }
}

// 2. Stateful екран (Керує станом і передає його у Stateless компонент)
@Composable
fun CounterScreen() {
    var count by rememberSaveable { mutableStateOf(0) }

    CounterContent(
        count = count,
        onIncrement = { count++ }
    )
}
```

---

## 5. Практичний приклад: Розгортається Картка (Expandable Card)

Побудуємо інтерактивний елемент інтерфейсу — картку, яка розгортається/згортається при натисканні на кнопку "Show More / Show Less":

```kotlin
@Composable
fun ExpandableCard(
    title: String,
    description: String,
    modifier: Modifier = Modifier
) {
    // Стан розгортання картки (виживає при повороті екрану)
    var isExpanded by rememberSaveable { mutableStateOf(false) }

    Card(
        modifier = modifier
            .fillMaxWidth()
            .padding(8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Column(
            modifier = Modifier
                .padding(16.dp)
        ) {
            Row(
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(
                    text = title,
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.weight(1f)
                )

                // Кнопка для перемикання стану
                IconButton(onClick = { isExpanded = !isExpanded }) {
                    Icon(
                        imageVector = if (isExpanded) Icons.Default.ExpandLess else Icons.Default.ExpandMore,
                        contentDescription = if (isExpanded) "Згорнути" else "Розгорнути"
                    )
                }
            }

            // Умовний рендеринг: опис показується тільки якщо isExpanded == true
            if (isExpanded) {
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = description,
                    style = MaterialTheme.typography.bodyMedium
                )
            }
        }
    }
}
```

---

## Підсумок

* **State (Стан)** — це будь-яке значення, яке змінюється з часом.
* **Recomposition (Рекомпозиція)** — процес автоматичного перемальовування UI при зміні стану.
* **`remember { mutableStateOf(...) }`** — створює стан і зберігає його між рекомпозиціями.
* **`rememberSaveable`** — зберігає стан при змінах конфігурації (повороті екрану).
* **State Hoisting (Підйом стану)** — винесення стану у параметри функції (`value`, `onValueChange`) для створення гнучких *Stateless* компонентів.
