# Списки у Compose: LazyColumn та LazyRow (Альтернатива RecyclerView)

У традиційному Android-підході для відображення списків використовувався `RecyclerView`. Він вимагав написання величезної кількості шаблонного коду: створення `Adapter`, `ViewHolder`, `DiffUtil` та XML-макетів для кожного типу елемента.

У Jetpack Compose списки створюються за допомогою компонентів **`LazyColumn`** (вертикальний список) та **`LazyRow`** (горизонтальний список). Вони автоматично перевіряють видимість елементів на екрані та створюють тільки ті вузли, які відображаються прямо зараз.

У цьому уроці ми вивчимо списки на основі картки **`ExpandableCard`**, яку ми розробили у попередньому туторіалі.

---

## 1. Чому не можна використовувати звичайний `Column` для великих списків?

Звичайний `Column` створює і відмальовує у пам'яті **абсолютно всі елементи одразу**. Якщо у вашому списку 1000 елементів, `Column` спробує завантажити всі 1000 елементів у RAM при відкритті екрану, що призведе до завісання або `OutOfMemoryError`.

```
┌──────────────────────────────────────────────────────────┐
│  Column (Погано для списків)                             │
│  Створює ВСІ 10 000 елементів у пам'яті одразу           │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  LazyColumn (Правильно!)                                 │
│  Створює ТІЛЬКИ ті 5 елементів, які видно на екрані      │
└──────────────────────────────────────────────────────────┘
```

**`LazyColumn`** відмальовує тільки ті елементи, які потрапляють у видиму область екрану (плюс невеличкий буфер для плавного скролінгу), заощаджуючи процесорний час та оперативну пам'ять.

---

## 2. Базовий синтаксис: `items()` та `item()`

Усередині блоку `LazyColumn` (чи `LazyRow`) ви використовуєте спеціальні функції DSL:

* **`items(list) { item -> ... }`** — приймає список даних і генерує Composable-елемент для кожного пункту.
* **`item { ... }`** — додає один поодинокий елемент (наприклад, заголовок списку чи рекламний банер).

### Приклад вертикального списку з розгортаними картками:

```kotlin
data class TopicItem(val id: Int, val title: String, val description: String)

@Composable
fun TopicCardList(topics: List<TopicItem>) {
    LazyColumn {
        // 1. Поодинокий заголовок списку
        item {
            Text(
                text = "Навчальні теми",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.padding(16.dp)
            )
        }

        // 2. Динамічний список ExpandableCard
        items(topics) { topic ->
            ExpandableCard(
                title = topic.title,
                description = topic.description
            )
        }
    }
}
```

---

## 3. Горизонтальні списки (`LazyRow`)

Якщо вам потрібен горизонтальний список, який можна гортати вбік (наприклад, категорій товарів чи фільтрів), просто замініть `LazyColumn` на **`LazyRow`**:

```kotlin
@Composable
fun CategoriesRow(categories: List<String>) {
    LazyRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(horizontal = 16.dp)
    ) {
        items(categories) { category ->
            Button(onClick = { /* ... */ }) {
                Text(text = category)
            }
        }
    }
}
```

---

## 4. Оптимізація та відступи у Lazy-списках

### 1. Відступи між елементами без `Spacer`
Вам більше не потрібно вставляти `Spacer` між кожним елементом списку. Замість цього використовуйте параметр **`verticalArrangement = Arrangement.spacedBy(...)`**:

```kotlin
LazyColumn(
    verticalArrangement = Arrangement.spacedBy(8.dp) // Відступ 8dp між усіма картками
) { ... }
```

### 2. Внутрішній padding для всього списку (`contentPadding`)
Якщо задати відступи через `Modifier.padding()`, контент списку обрізатиметься при скролінгу. Щоб список гарно скролився під системні панелі, використовуйте параметр **`contentPadding`**:

```kotlin
LazyColumn(
    contentPadding = PaddingValues(16.dp) // Зовнішні відступи для всієї області скролінгу
) { ... }
```

### 3. Ключі оптимізації `key` (Аналог DiffUtil)
За замовчуванням позиція елемента у списку визначається за його індексом. Якщо ви видалите або перемішаєте елементи, Compose може втратити їхній внутрішній стан (наприклад, стан розкриття конкретної `ExpandableCard`).

Щоб Compose точно знав, яка `ExpandableCard` відповідає якому об'єкту, завжди передавайте унікальний ключ **`key`**:

```kotlin
items(
    items = topicList,
    key = { topic -> topic.id } // ⚡ Унікальний ID для збереження стану та плавних анімацій!
) { topic ->
    ExpandableCard(
        title = topic.title,
        description = topic.description
    )
}
```

---

## 5. Динамічне додавання та видалення елементів списку

У застарілому `RecyclerView` для додавання чи видалення елементів доводилося викликати ручні сповіщення адаптера: `adapter.notifyItemInserted()` або `adapter.notifyItemRemoved()`.

У Jetpack Compose **немає жодних адаптерів чи методів сповіщення**! Зміни у списку відбуваються суто декларативно через **Стан (State)**:

1. Список елементів зберігається у списку стану (наприклад, `mutableStateListOf()` або `StateFlow<List<T>>`).
2. Ви просто додаєте або видаляєте об'єкт зі списку даних (`list.add()` або `list.remove()`).
3. Compose автоматично виявляє зміну стану і миттєво виконує **рекомпозицію**, оновлюючи екран!

### Приклад: Список із можливістю додавання та видалення карток

```kotlin
data class TaskItem(val id: Int, val title: String, val description: String)

@Composable
fun DynamicCardList() {
    // 1. Створюємо динамічний список стану за допомогою mutableStateListOf
    val tasks = remember {
        mutableStateListOf(
            TaskItem(1, "Тема 1", "Вступ до Jetpack Compose"),
            TaskItem(2, "Тема 2", "Управління станом та Recomposition")
        )
    }

    Column(modifier = Modifier.padding(16.dp)) {
        // Кнопка додавання нового елемента у список
        Button(
            onClick = {
                val nextId = (tasks.maxOfOrNull { it.id } ?: 0) + 1
                tasks.add(TaskItem(nextId, "Тема $nextId", "Опис нової теми $nextId"))
            },
            modifier = Modifier.fillMaxWidth().padding(bottom = 12.dp)
        ) {
            Text("Додати нову картку")
        }

        // LazyColumn, який автоматично відмальовує зміни у tasks
        LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
            items(
                items = tasks,
                key = { task -> task.id } // ⚡ Унікальний ключ обов'язковий!
            ) { task ->
                Row(verticalAlignment = Alignment.CenterVertically) {
                    ExpandableCard(
                        title = task.title,
                        description = task.description,
                        modifier = Modifier.weight(1f)
                    )
                    // Кнопка видалення елемента зі списку
                    IconButton(onClick = { tasks.remove(task) }) {
                        Icon(Icons.Default.Delete, contentDescription = "Видалити")
                    }
                }
            }
        }
    }
}
```

---

### Анімація додавання та видалення елементів: `Modifier.animateItem()`

Щоб елементи списку додавалися, видалялися та зміщувалися з плавними анімаціями (замість миттєвого зникнення), просто додайте модифікатор **`Modifier.animateItem()`** до дочірнього елемента всередині `LazyColumn`:

```kotlin
items(
    items = tasks,
    key = { task -> task.id } // ⚡ Параметр key НЕОБХІДНИЙ для роботи анімацій!
) { task ->
    ExpandableCard(
        title = task.title,
        description = task.description,
        modifier = Modifier.animateItem() // ⚡ Анімація появи, видалення та зміщення!
    )
}
```

---

## Повний практичний приклад: Список розгортаних карток курсу

У цьому прикладі кожна картка списку `ExpandableCard` зберігає свій власний стан розгортання (`isExpanded`) під час скролінгу завдяки `rememberSaveable` та ключу `key`:

```kotlin
data class CourseModule(
    val id: Int,
    val title: String,
    val description: String
)

@Composable
fun CourseModuleList(modules: List<CourseModule>) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        // Поодинокий заголовок списку
        item {
            Text(
                text = "Модулі курсу Android 2026",
                style = MaterialTheme.typography.titleLarge,
                modifier = Modifier.padding(bottom = 8.dp)
            )
        }

        // Список розгортаних карток
        items(
            items = modules,
            key = { module -> module.id } // Зберігає стан роз розкриття картки при скролі!
        ) { module ->
            ExpandableCard(
                title = module.title,
                description = module.description
            )
        }
    }
}
```

---

## Підсумок

* **`LazyColumn`** та **`LazyRow`** — це Compose-аналоги `RecyclerView`, які завантажують у пам'ять тільки видимі на екрані елементи.
* **`items(list)`** відповідає за генерацію елементів зі списку, а **`item { ... }`** — за поодинокі елементи (заголовки, банери).
* **`contentPadding`** додає відступи для всього списку без обрізання скролінгу.
* **`verticalArrangement = Arrangement.spacedBy(...)`** автоматично робить однаковий відступ між елементами.
* **`key = { ... }`** — унікальний ідентифікатор елемента, завдяки якому кожна `ExpandableCard` зберігає свій власний стан розгортання при скролінгу (аналог `DiffUtil`).
