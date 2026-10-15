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
