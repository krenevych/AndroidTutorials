# Створення простого застосунку (View Approach)

У цьому туторіалі ми створимо найпростіший Android-застосунок, використовуючи класичний підхід з XML-розміткою (View Approach).
Наш застосунок буде містити текстове поле (TextView), поле для вводу тексту (EditText) та кнопку (Button). При натисканні на кнопку, текст, введений користувачем, буде відображатися у текстовому полі.

## 1. Створення проекту

1. Відкрийте Android Studio.
2. Натисніть **New Project**.
3. Виберіть шаблон **Empty Views Activity** (цей шаблон одразу налаштує проект для використання класичних Views, а не Jetpack Compose).
4. Назвіть проект, наприклад, `SimpleViewsApp`.
5. Переконайтеся, що вибрана мова **Kotlin**.
6. Натисніть **Finish** та дочекайтеся завершення синхронізації проекту Gradle.

## 2. Створення інтерфейсу користувача (UI)

Інтерфейс у View-підході створюється у спеціальних XML-файлах. 
Відкрийте файл розмітки екрану `res/layout/activity_main.xml`. Перейдіть у режим відображення **Code** (кнопка у правому верхньому куті редактора) і замініть його вміст на наступний код. Він використовує `LinearLayout` для вертикального розміщення елементів один під одним:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="24dp"
    android:gravity="center"
    tools:context=".MainActivity">

    <!-- Текстове поле для відображення результату -->
    <TextView
        android:id="@+id/resultTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Привіт!"
        android:textSize="24sp"
        android:layout_marginBottom="24dp" />

    <!-- Поле для вводу тексту користувачем -->
    <EditText
        android:id="@+id/inputEditText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Введіть ваше ім'я"
        android:inputType="textPersonName"
        android:layout_marginBottom="24dp" />

    <!-- Кнопка для виконання дії -->
    <Button
        android:id="@+id/actionButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Привітатися" />

</LinearLayout>
```

### Основні UI-компоненти (Views):
* **TextView** — використовується для відображення статичного тексту на екрані. Ми задали йому унікальний ідентифікатор `@+id/resultTextView`.
* **EditText** — поле, де користувач може вводити текст (клавіатура з'явиться автоматично при натисканні). Ідентифікатор — `@+id/inputEditText`. Атрибут `android:hint` показує підказку сірим кольором, поки поле порожнє.
* **Button** — звичайна кнопка, на яку можна натиснути. Ідентифікатор — `@+id/actionButton`.

## 3. Написання логіки в Activity (MainActivity.kt)

Тепер додамо логіку роботи: зробимо так, щоб кнопка реагувала на натискання, зчитувала текст з `EditText` та виводила його у `TextView`.
Відкрийте файл `MainActivity.kt` (знаходиться у папці `java/<назва_вашого_пакету>/`). 

Оновіть метод `onCreate` наступним чином:

```kotlin
package com.example.simpleviewsapp // У вас тут буде назва вашого пакету

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 1. Встановлюємо XML-розмітку для цього екрану (Activity)
        setContentView(R.layout.activity_main)

        // 2. Знаходимо елементи інтерфейсу за їхніми ID з XML файлу
        val resultTextView = findViewById<TextView>(R.id.resultTextView)
        val inputEditText = findViewById<EditText>(R.id.inputEditText)
        val actionButton = findViewById<Button>(R.id.actionButton)

        // 3. Встановлюємо слухач натискань (click listener) на кнопку
        actionButton.setOnClickListener {
            // Отримуємо текст, який ввів користувач в EditText
            val inputText = inputEditText.text.toString()

            // Перевіряємо, чи введений текст не порожній
            if (inputText.isNotBlank()) {
                // Змінюємо властивість .text у TextView
                resultTextView.text = "Привіт, $inputText!"
                
                // (Опційно) Очистити поле після натискання
                // inputEditText.text.clear()
            } else {
                resultTextView.text = "Будь ласка, введіть ім'я."
            }
        }
    }
}
```

### Розбір коду:
1. `setContentView(R.layout.activity_main)` — ця команда повідомляє Android, який саме XML-файл потрібно "намалювати" на екрані при запуску `MainActivity`.
2. `findViewById<Тип>(R.id.назва_id)` — класичний (і основний для View-підходу) метод пошуку елемента інтерфейсу з XML-розмітки для подальшої взаємодії з ним у коді Kotlin.
3. `setOnClickListener { ... }` — блок коду, який буде виконуватись щоразу, коли користувач натискає на кнопку `actionButton`.

## 4. Запуск застосунку

1. Підключіть реальний Android-пристрій через кабель (не забудьте увімкнути "Відлагодження по USB" в налаштуваннях розробника) або запустіть віртуальний емулятор (Android Virtual Device).
2. Натисніть кнопку **Run** (зелений трикутник) на панелі інструментів зверху або комбінацію `Shift + F10`.
3. Коли застосунок збереться і запуститься, ви побачите поле вводу та кнопку. Введіть туди текст і натисніть кнопку — привітання на екрані повинно оновитись!

## 5. Дебаг (відлагодження) за допомогою Log

### Що таке логування і для чого воно потрібно?
Мобільний застосунок працює на телефоні, який часто не підключений до компʼютера. Якщо в програмі стається помилка або вона поводить себе дивно, ви не завжди можете одразу побачити, що саме пішло не так. 

**Логування** — це процес запису текстових повідомлень про хід виконання програми у спеціальний системний журнал пристрою (log). 

Це не такий "прямий" та інтерактивний спосіб пошуку помилок, як використання класичного Дебагера (Debugger, який дозволяє "заморозити" програму на певному рядку та подивитись стан пам'яті). Однак логування є надзвичайно важливим інструментом, тому що:
* Ви можете бачити історію роботи програми: які екрани відкривались, які дані вводив користувач, що приходило з інтернету.
* Журнал записується безперервно. Якщо застосунок раптово впав (сраш), ви можете переглянути логи і зрозуміти, що саме передувало помилці.
* Це найшвидший спосіб перевірити, чи викликається певний шматок коду (наприклад, чи спрацьовує обробник натискання кнопки).

Для запису таких повідомлень в Android використовується клас `Log`.

### Синтаксис `Log`
Методи класу `Log` зазвичай приймають два основних параметри. Розглянемо синтаксис на прикладі методу `Log.d`:

```kotlin
Log.d(TAG, "Текст вашого повідомлення")
```
* **`TAG` (Тег)** — це рядок (`String`), який допомагає ідентифікувати джерело повідомлення (щоб потім легше було знайти його в консолі). Зазвичай як тег використовують назву поточного класу (наприклад, `"MainActivity"`).
* **Повідомлення** — це, власне, сам текст (`String`), який ви хочете вивести в лог.

Давайте додамо логування у наш обробник натискання кнопки, щоб бачити, який саме текст вводить користувач. Для зручності ми також створимо константу `TAG` в класі `MainActivity`:

```kotlin
import android.util.Log // Не забудьте додати цей імпорт
// ...

class MainActivity : AppCompatActivity() {

    // Створюємо константу для тегу логів
    companion object {
        private const val TAG = "MainActivity"
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        // ...
        
        actionButton.setOnClickListener {
            val inputText = inputEditText.text.toString()

            // Виводимо повідомлення в лог з тегом TAG ("MainActivity")
            Log.d(TAG, "Користувач ввів текст: $inputText")

            if (inputText.isNotBlank()) {
                resultTextView.text = "Привіт, $inputText!"
            } else {
                // Повідомлення рівня Warning (Попередження), якщо поле порожнє
                Log.w(TAG, "Введено порожній текст!")
                resultTextView.text = "Будь ласка, введіть ім'я."
            }
        }
    }
}
```

### Як переглядати логи (Logcat)
1. Внизу вікна Android Studio знайдіть і відкрийте вкладку **Logcat** (або натисніть `Alt + 6` на Windows / `Cmd + 6` на Mac).
2. Запустіть застосунок на пристрої.
3. Коли ви натиснете на кнопку, в панелі Logcat з'явиться ваше повідомлення.
4. Ви можете використовувати рядок пошуку в Logcat, ввівши туди ваш тег (наприклад, `tag:MainActivity`), щоб відфільтрувати лише ваші повідомлення і не бачити системні логи.

*Основні рівні логування:*
* `Log.d` (Debug) — для відлагодження, щоб бачити інформацію, яка корисна під час розробки.
* `Log.i` (Info) — для інформаційних повідомлень про успішні операції.
* `Log.w` (Warning) — для попереджень, коли щось пішло не так, але програма не впала.
* `Log.e` (Error) — для повідомлень про серйозні помилки.

## Підсумок
Вітаємо! Ви щойно створили базовий застосунок, який використовує класичні View-компоненти: `TextView`, `EditText` та `Button`. Ви навчилися розміщувати їх в екрані за допомогою `LinearLayout` та оживляти їх через Kotlin-код за допомогою `findViewById` та обробників подій. А також дізналися, як використовувати `Log` для відлагодження програми.
