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

## 5. Intent Filters: Як Android розуміє, який екран відкривати?

Коли ви використовуєте явний інтент, ви прямо кажете системі: *"Відкрий клас SecondActivity"*. Тут все просто.
Але коли ви викликаєте неявний інтент (наприклад, `Intent.ACTION_SEND` для тексту з Прикладу 2), як Android знаходить потрібну програму серед сотень встановлених на телефоні?

Відповідь криється у **Фільтрах Інтентів (Intent Filters)**.

Кожна програма (включно з вашою) може "рекламувати" свої можливості системі через файл `AndroidManifest.xml`. Якщо ви розробляєте власний месенджер і хочете, щоб він з'являвся у системному меню "Поділитися", ви повинні додати `<intent-filter>` всередину тегу вашої Activity.

Ось як це виглядає в `AndroidManifest.xml`:

```xml
<activity 
    android:name=".ShareActivity" 
    android:exported="true">
    
    <!-- Цей фільтр каже системі: "Я вмію обробляти відправку тексту!" -->
    <intent-filter>
        <!-- ДІЯ: Що ми вміємо робити (Відправляти) -->
        <action android:name="android.intent.action.SEND" />
        
        <!-- КАТЕГОРІЯ: Як нас можна викликати (DEFAULT означає стандартний виклик) -->
        <category android:name="android.intent.category.DEFAULT" />
        
        <!-- ДАНІ: З яким типом даних ми працюємо (звичайний текст) -->
        <data android:mimeType="text/plain" />
    </intent-filter>
    
</activity>
```

**Як це працює на практиці:**
1. Користувач натискає кнопку "Поділитися текстом" у якійсь програмі (генерується неявний інтент `ACTION_SEND` з типом `text/plain`).
2. Система Android зупиняється і сканує Маніфести **всіх** встановлених програм на телефоні.
3. Вона шукає збіги (match) між згенерованим інтентом та прописаними `intent-filter`.
4. Всі застосунки (Telegram, Viber, і ваша `ShareActivity`), які мають такий фільтр, потрапляють у гарний список (Chooser), який Android показує користувачу на екрані.

### Основні типи Action та Category
Щоб правильно налаштувати Intent Filter, потрібно знати стандартні константи (Дії та Категорії), які використовує Android:

**Популярні `action` (Дії):**
* `android.intent.action.MAIN` — вказує, що це головна точка входу в програму.
* `android.intent.action.VIEW` — найпоширеніша дія: відобразити дані користувачеві (відкрити посилання в браузері, показати фото в галереї, відкрити адресу на карті).
* `android.intent.action.SEND` — намір надіслати якісь дані (текст, картинку) іншому користувачеві (викликає меню "Поділитися").
* `android.intent.action.DIAL` — відкрити екран набору номера телефону.
* `android.intent.action.IMAGE_CAPTURE` — запит на відкриття камери для створення фото.

**Популярні `category` (Категорії):**
* `android.intent.category.LAUNCHER` — вказує, що іконка цієї Activity має відображатися у списку всіх програм на робочому столі смартфона (Launcher). Зазвичай використовується в парі з дією `android.intent.action.MAIN`.
* `android.intent.category.DEFAULT` — **обов'язкова** категорія для будь-якої Activity, яка хоче приймати неявні інтенти (окрім дії `MAIN`). Якщо ви забудете додати `DEFAULT` у свій фільтр (наприклад, для обробки посилань), система вас просто проігнорує.
* `android.intent.category.BROWSABLE` — дозволяє відкривати цю Activity безпосередньо з веб-браузера при натисканні на посилання.

> **Чи може одна Activity мати кілька Actions або Категорій?**
> Так, звичайно! Одна Activity може мати одразу декілька блоків `<intent-filter>`, або ж містити декілька `<action>` та `<category>` всередині одного фільтра. Якщо в одному фільтрі вказано кілька дій, це означає, що Activity вміє обробляти **будь-яку** з них (спрацьовує правило "АБО"). 
> 
> *Приклад: Ваша програма вміє відправляти повідомлення (`SEND`) і як одиночним отримувачам, так і відправляти їх одразу кільком людям (`SEND_MULTIPLE`). До того ж, вона може робити це як стандартним способом, так і при відкритті через браузер. Ви можете записати це в один фільтр:*
> ```xml
> <intent-filter>
>     <action android:name="android.intent.action.SEND" />
>     <action android:name="android.intent.action.SEND_MULTIPLE" />
>     <category android:name="android.intent.category.DEFAULT" />
>     <category android:name="android.intent.category.BROWSABLE" />
>     <data android:mimeType="text/plain" />
> </intent-filter>
> ```

Отже, **Intent Filter** — це спосіб для Activity заявити операційній системі про те, які неявні інтенти вона здатна обробляти.

## Підсумок
* **`Intent`** — це повідомлення для Android про те, що ви хочете виконати якусь дію.
* **Явні інтенти** використовуються для переходу між вашими власними екранами.
* **Неявні інтенти** використовуються для запуску зовнішніх програм, делегуючи виконання дії ОС.
* **`Intent Filter`** — це блок у Маніфесті, за допомогою якого застосунок "реєструється" в системі як такий, що вміє обробляти певні неявні інтенти (наприклад, відкривати посилання або приймати текст).
* **`putExtra`** / **`getStringExtra`** — дозволяють передавати прості дані між різними Activity. 
* Не забувайте, що будь-яка нова Activity у вашому застосунку обов'язково має бути прописана у файлі `AndroidManifest.xml`!