# Завдання: Гарантоване фонове виконання задач за допомогою WorkManager

## Мета завдання
Опанувати роботу з **`WorkManager`** — сучасним системним рішенням для виконання відкладених та періодичних фонових задач. Навчитися створювати робочі класи (`CoroutineWorker`), налаштовувати умови запуску (`Constraints`), передавати та отримувати дані, а також запобігати дублюванню періодичних задач.

---

## Вимоги до застосунку

Уявіть, що ви розробляєте новинний застосунок, який повинен **щоденно вночі синхронізувати останні новини**, але робити це так, щоб **не саджати батарею користувача**.

Крім цього, користувач має кнопку "Синхронізувати зараз" для миттєвого ручного завантаження.

### 1. UI (Compose)
На екрані повинні бути:
* Кнопка **"Синхронізувати зараз"** (одноразова задача).
* Кнопка **"Увімкнути нічну синхронізацію"** (періодична задача).
* **`Text`**, який відображає статус виконання одноразової задачі (наприклад, *"Очікує..."*, *"Виконується"*, *"Успіх: завантажено 15 новин"*).

### 2. Умови запуску (`Constraints`)
* **Періодична задача** повинна запускатися ТІЛЬКИ тоді, коли телефон підключено до Wi-Fi (`UNMETERED`) та він лежить на зарядці (`RequiresCharging`).
* **Одноразова задача** повинна запускатися при наявності будь-якого інтернету (`CONNECTED`).

### 3. Клас `NewsSyncWorker` (`CoroutineWorker`)
* Робочий клас має приймати на вхід строку-параметр `user_token`.
* Усередині методу `doWork()` зімітуйте мережевий запит за допомогою `delay(3000)`.
* Після успішного "завантаження" Воркер має повернути `Result.success(outputData)`, передавши повідомлення (наприклад, *"Синхронізовано"*).

---

## Етапи виконання

### Крок 1. Залежності
Додайте у `build.gradle.kts` бібліотеку WorkManager:
```kotlin
implementation("androidx.work:work-runtime-ktx:2.9.0")
```

### Крок 2. Реалізація `NewsSyncWorker`
1. Створіть клас `NewsSyncWorker`, успадкуйте його від `CoroutineWorker`.
2. У `doWork()` витягніть токен: `val token = inputData.getString("user_token")`.
3. Зробіть `delay(3000)`, імітуючи скачування.
4. Запакуйте результат: `val out = Data.Builder().putString("result", "Успіх!").build()`.
5. Поверніть `Result.success(out)`.

### Крок 3. Створення та запуск одноразової задачі (UI)
1. У `MainActivity` по кліку на кнопку "Синхронізувати зараз" створіть `Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build()`.
2. Створіть `OneTimeWorkRequestBuilder<NewsSyncWorker>()`, додавши `Constraints` та `setInputData(Data.Builder().putString("user_token", "abc123").build())`.
3. Викличте `WorkManager.getInstance(context).enqueue(request)`.

### Крок 4. Отримання статусу виконання
1. Підпишіться на стан щойно створеної одноразової задачі:
```kotlin
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(request.id)
    .observe(this) { workInfo ->
        if (workInfo != null) {
            val status = workInfo.state.name
            // Оновіть StateFlow для Compose, щоб показати статус ("ENQUEUED", "RUNNING", "SUCCEEDED")
            if (workInfo.state == WorkInfo.State.SUCCEEDED) {
                val message = workInfo.outputData.getString("result")
                // Покажіть результат
            }
        }
    }
```

### Крок 5. Створення періодичної задачі
1. По кліку на другу кнопку створіть жорсткі `Constraints` (Wi-Fi + Зарядка).
2. Створіть `PeriodicWorkRequestBuilder<NewsSyncWorker>(15, TimeUnit.MINUTES)` (оскільки 15 хв — це системний мінімум).
3. Використайте метод **`enqueueUniquePeriodicWork`** з політикою `ExistingPeriodicWorkPolicy.KEEP` та ім'ям `"night_news_sync"`, щоб не наплодити дублікатів.

### Крок 6. Тестування та Симуляція
Для швидкого тестування WorkManager не потрібно чекати 15 хвилин чи шукати кабель для зарядки! 
* Натисніть кнопку "Увімкнути нічну синхронізацію".
* Скористайтеся інструментом Android Studio **App Inspection ➔ Background Task Inspector**.
* Там ви побачите вашу задачу зі статусом *Enqueued*. Ви зможете натиснути кнопку **"Execute" (Запустити примусово)**, і система миттєво виконає її, ігноруючи `Constraints` та інтервали!

Успіхів у налаштуванні ідеальної фонової роботи додатків!
