# Використання View Binding замість findViewById

У попередньому туторіалі ми створили простий застосунок, використовуючи метод `findViewById` для зв'язку між XML-розміткою та кодом Kotlin. Хоча цей метод є класичним і перевіреним часом, він має свої недоліки: код стає громіздким (багато рядків пошуку елементів), є ризик помилок з типами (Type Safety) та помилок через те, що елемент не знайдено (краш через NullPointerException).

Сучасний підхід в Android-розробці — це використання інструменту **View Binding**. 
View Binding автоматично генерує спеціальний клас для кожного вашого XML-файлу розмітки, дозволяючи звертатися до UI-компонентів напряму та безпечно.

Давайте переробимо наш попередній застосунок під View Binding, щоб ви могли побачити різницю.

## 1. Увімкнення View Binding у проекті

View Binding за замовчуванням вимкнений. Щоб його увімкнути:
1. У лівій панелі Android Studio (вікно Project) знайдіть і відкрийте файл `build.gradle.kts` (або `build.gradle`), який відповідає за модуль вашого застосунку (зазвичай має приписку `Module :app`).
2. Додайте блок `buildFeatures` всередину блоку `android`:

```kotlin
android {
    namespace = "com.example.simpleviewsapp"
    compileSdk = 34

    // ... інші налаштування ...

    // Додаємо цей блок:
    buildFeatures {
        viewBinding = true
    }
}
```
3. У верхньому правому куті редактора з'явиться банер із кнопкою **Sync Now**. Обов'язково натисніть її. Після завершення синхронізації Gradle, Android Studio автоматично згенерує класи прив'язки для всіх ваших XML-файлів.

## 2. Інтерфейс користувача (UI)

Ми використовуємо **абсолютно ту саму розмітку**, що й у попередньому туторіалі. Нічого змінювати в `activity_main.xml` не потрібно!

*Нагадаємо, що там є 3 елементи:*
* `TextView` з ID `@+id/resultTextView`
* `EditText` з ID `@+id/inputEditText`
* `Button` з ID `@+id/actionButton`

Коли View Binding увімкнено, для файлу `activity_main.xml` автоматично генерується клас-обгортка з назвою `ActivityMainBinding` (назва формується з імені XML-файлу: `activity_main` -> `ActivityMainBinding`).

## 3. Зміни в MainActivity.kt

Тепер змінимо логіку. Ми більше не будемо використовувати `findViewById`. Замість цього ми створимо об'єкт "прив'язки" (binding) і будемо звертатися до всіх елементів через нього.

Відкрийте `MainActivity.kt` і оновіть його наступним чином:

```kotlin
package com.example.simpleviewsapp

import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
// Обов'язково імпортуємо згенерований клас для нашої розмітки
import com.example.simpleviewsapp.databinding.ActivityMainBinding 

class MainActivity : AppCompatActivity() {

    companion object {
        private const val TAG = "MainActivity"
    }

    // 1. Оголошуємо змінну для binding
    // "lateinit" означає, що ми ініціалізуємо її пізніше (в методі onCreate)
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 2. Ініціалізуємо binding, "надуваючи" (inflating) XML-розмітку
        binding = ActivityMainBinding.inflate(layoutInflater)
        
        // 3. Передаємо кореневий елемент (root) розмітки для відображення на екрані
        // Це замінює класичний setContentView(R.layout.activity_main)
        setContentView(binding.root)

        // 4. Тепер ми можемо звертатися до всіх елементів через об'єкт binding!
        // Зверніть увагу: ID елементів з XML (наприклад, resultTextView) 
        // автоматично стають доступними як властивості об'єкта binding.
        
        binding.actionButton.setOnClickListener {
            val inputText = binding.inputEditText.text.toString()

            Log.d(TAG, "Користувач ввів текст: $inputText")

            if (inputText.isNotBlank()) {
                binding.resultTextView.text = "Привіт, $inputText!"
            } else {
                Log.w(TAG, "Введено порожній текст!")
                binding.resultTextView.text = "Будь ласка, введіть ім'я."
            }
        }
    }
}
```

## 4. Чому View Binding кращий за findViewById?

Як ви могли помітити, код став значно чистішим. Ось головні переваги використання View Binding:

1. **Type Safety (Безпека типів):** View Binding автоматично знає, що `binding.resultTextView` — це `TextView`, а `binding.actionButton` — це `Button`. Вам не потрібно явно вказувати типи, і ви не зможете випадково викликати метод кнопки у текстового поля.
2. **Null Safety (Захист від null):** Якщо елемент існує в XML, він гарантовано буде в об'єкті `binding`. Ви не отримаєте помилку `NullPointerException` (краш програми), що часто трапляється при помилці в `findViewById` (наприклад, якщо випадково вказати ID від іншого екрану).
3. **Менше коду:** Вам не потрібно писати окремий рядок `val button = findViewById<Button>(R.id.button)` для кожного елемента на екрані. Усі компоненти миттєво доступні всередині єдиного об'єкта `binding`.

## Підсумок
Ви навчилися підключати та використовувати View Binding — сучасний стандарт для роботи з класичним UI в Android. Перехід від `findViewById` до `ViewBinding` робить ваш код більш надійним, коротким та захищеним від помилок. Якщо ви розробляєте проект не на Jetpack Compose, використання View Binding є суворою рекомендацією (Best Practice).
