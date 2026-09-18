# Навігація між екранами: Запуск іншої Activity

Більшість повноцінних застосунків складаються з кількох екранів. У класичному підході (View Approach) кожен окремий екран найчастіше представлений класом `Activity`. 

Для того, щоб перейти з однієї Activity на іншу, в Android використовується спеціальний клас — **`Intent`** (Намір).

У цьому короткому туторіалі ми навчимося відкривати новий екран при натисканні на кнопку, а також передавати на нього дані.

## 1. Створення другого екрану (SecondActivity)

Перш ніж кудись переходити, нам потрібен сам екран, куди ми будемо переходити.
1. В Android Studio в панелі **Project** (зліва) натисніть правою кнопкою миші на папку вашого пакету (наприклад, `com.example.myapp`).
2. Виберіть **New -> Activity -> Empty Views Activity**.
3. Введіть назву, наприклад, `SecondActivity` і натисніть **Finish**.

*Що при цьому відбулося?*
* Android Studio створила файл `SecondActivity.kt`.
* Створила файл розмітки `activity_second.xml` у папці `res/layout`.
* **Найважливіше:** автоматично додала запис `<activity android:name=".SecondActivity" />` у ваш `AndroidManifest.xml`. (Без цього програма впала б при спробі відкрити екран!).

## 2. Відкриття SecondActivity за допомогою Intent

Припустимо, у вашій `MainActivity` є кнопка (з ID `btnOpenSecond`). Давайте додамо логіку її натискання, щоб відкрити `SecondActivity`.

Відкрийте `MainActivity.kt` і додайте наступний код:

```kotlin
import android.content.Intent // Не забудьте імпортувати Intent
import android.os.Bundle
import android.widget.Button
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val btnOpenSecond = findViewById<Button>(R.id.btnOpenSecond)

        btnOpenSecond.setOnClickListener {
            // 1. Створюємо Intent. 
            // Параметри: звідки (this - поточна Activity) та куди (SecondActivity::class.java)
            val intent = Intent(this, SecondActivity::class.java)
            
            // 2. Запускаємо нову Activity
            startActivity(intent)
        }
    }
}
```

Цей тип Intent-у називається **явним (Explicit Intent)**, тому що ми чітко (явно) вказуємо клас екрану (`SecondActivity`), який хочемо відкрити.

## 3. Передача даних між екранами

Часто нам потрібно не просто відкрити новий екран, а й передати туди якусь інформацію (наприклад, текст, який користувач ввів у попередньому екрані). Для цього Intent має спеціальний механізм "додатків" (Extras), який працює як словник (ключ-значення).

### Як відправити дані (з MainActivity):
```kotlin
btnOpenSecond.setOnClickListener {
    val intent = Intent(this, SecondActivity::class.java)
    
    // Додаємо дані в Intent. 
    // "EXTRA_USER_NAME" - це ключ (назва комірки), а "Олександр" - значення.
    intent.putExtra("EXTRA_USER_NAME", "Олександр")
    intent.putExtra("EXTRA_USER_AGE", 25) // Можна передавати числа та інші прості типи
    
    startActivity(intent)
}
```

### Як отримати дані (в SecondActivity):
Тепер відкриємо `SecondActivity.kt` і зчитаємо ці дані у методі `onCreate`:

```kotlin
import android.os.Bundle
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class SecondActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_second)

        // Зчитуємо дані з Intent-у, який відкрив цю Activity
        // Використовуємо ті самі ключі!
        val userName = intent.getStringExtra("EXTRA_USER_NAME")
        
        // Для чисел (Int) потрібно вказати значення за замовчуванням (defaultValue), 
        // на випадок, якщо таких даних не було передано
        val userAge = intent.getIntExtra("EXTRA_USER_AGE", 0) 

        // Тепер ці дані можна вивести на екран
        val textView = findViewById<TextView>(R.id.textViewResult)
        textView.text = "Привіт, $userName! Тобі $userAge років."
    }
}
```

## 4. Неявні Інтенти (Implicit Intents)

До цього ми розглядали **явні (Explicit)** інтенти, коли ми точно знали, який клас хочемо відкрити (`SecondActivity::class.java`). Але що, якщо ми хочемо виконати дію, для якої у нас немає власного екрану? Наприклад: відкрити веб-сайт, подзвонити за номером телефону, поділитися текстом або відкрити карту.

Для цього використовуються **неявні (Implicit) інтенти**. Ви кажете системі Android *"Що потрібно зробити"*, а система сама шукає застосунок на телефоні користувача, який вміє це робити (наприклад, відкриває браузер Chrome для посилання або Google Maps для адреси).

### Приклад 1: Відкриття веб-сторінки
```kotlin
btnOpenWeb.setOnClickListener {
    // Вказуємо дію ACTION_VIEW (перегляд) і дані у вигляді URI (посилання)
    val webpage: Uri = Uri.parse("https://www.google.com")
    val intent = Intent(Intent.ACTION_VIEW, webpage)
    
    // Перевіряємо, чи є на телефоні хоча б одна програма, здатна це відкрити
    if (intent.resolveActivity(packageManager) != null) {
        startActivity(intent)
    }
}
```

### Приклад 2: Відправка тексту в інші програми (Share)
Цей інтент викличе стандартне системне меню "Поділитися" (де користувач зможе вибрати Telegram, Viber, пошту тощо).

```kotlin
btnShareText.setOnClickListener {
    val sendIntent = Intent().apply {
        action = Intent.ACTION_SEND
        putExtra(Intent.EXTRA_TEXT, "Привіт! Подивись, який крутий застосунок я написав!")
        type = "text/plain" // Вказуємо тип даних (звичайний текст)
    }
    
    // Створюємо гарне меню "Поділитися через..."
    val shareIntent = Intent.createChooser(sendIntent, "Поділитися через...")
    startActivity(shareIntent)
}
```

### Приклад 3: Написання Email-листа
Якщо ви хочете, щоб користувач міг відправити вам лист до підтримки, ви можете використати інтент, який відкриє поштовий клієнт (наприклад, Gmail) із вже заповненими полями "Кому" та "Тема".

```kotlin
btnSendEmail.setOnClickListener {
    val emailIntent = Intent(Intent.ACTION_SENDTO).apply {
        // "mailto:" означає, що інтент повинні перехопити лише поштові програми
        data = Uri.parse("mailto:") 
        
        // Масив адрес, куди відправляємо лист
        putExtra(Intent.EXTRA_EMAIL, arrayOf("support@example.com")) 
        putExtra(Intent.EXTRA_SUBJECT, "Відгук про застосунок")
        putExtra(Intent.EXTRA_TEXT, "Привіт, розробники! Я хотів би...")
    }

    if (emailIntent.resolveActivity(packageManager) != null) {
        startActivity(emailIntent)
    }
}
```

### Приклад 4: Відкриття додатку телефону (Dialer)
Цей інтент відкриває стандартну програму телефону і вводить номер, але **не здійснює** сам дзвінок автоматично (користувач повинен сам натиснути зелену кнопку виклику). Це найбезпечніший спосіб, який не вимагає спеціальних дозволів.

```kotlin
btnCall.setOnClickListener {
    // Вказуємо дію DIAL і передаємо номер телефону через префікс "tel:"
    val number = Uri.parse("tel:+380123456789")
    val callIntent = Intent(Intent.ACTION_DIAL, number)
    
    if (callIntent.resolveActivity(packageManager) != null) {
        startActivity(callIntent)
    }
}
```

> **Важливо:** Якщо б ви хотіли здійснити дзвінок *одразу* без участі користувача, потрібно було б використати `Intent.ACTION_CALL`. Але для цього вашому застосунку обов'язково знадобиться спеціальний дозвіл `android.permission.CALL_PHONE` у файлі `AndroidManifest.xml`, а також запит цього дозволу у користувача під час роботи програми. Для відкриття номеронабирача (`ACTION_DIAL`), відкриття браузера чи Email спеціальні дозволи не потрібні.

## Підсумок
* **`Intent`** — це повідомлення для Android про те, що ви хочете виконати якусь дію.
* **Явні інтенти** використовуються для переходу між вашими власними екранами.
* **Неявні інтенти** використовуються для запуску зовнішніх програм (браузер, карти, шеринг), делегуючи виконання дії операційній системі.
* **`putExtra`** / **`getStringExtra`** — дозволяють передавати прості дані (текст, числа, булеві значення) між різними Activity. 
* Не забувайте, що будь-яка нова Activity у вашому застосунку обов'язково має бути прописана у файлі `AndroidManifest.xml`!