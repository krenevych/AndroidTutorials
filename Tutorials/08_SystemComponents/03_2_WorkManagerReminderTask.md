# Завдання: Регулярне нагадування "Випий води!" (Periodic WorkManager)

## Мета завдання
Навчитися працювати з **`PeriodicWorkRequest`** у WorkManager для створення регулярних фонових задач. Опанувати надсилання системних сповіщень (`Notification`) безпосередньо з `CoroutineWorker`, роботу з `SharedPreferences` (або `DataStore`) для збереження лічильника виконань та реалізувати керування фоновими задачами через перемикач у Compose UI.

---

## Опис та вимоги до застосунку

Ваша мета — створити трекер гідратації, який кожні 15 хвилин (або за заданим інтервалом) нагадує користувачеві випити води за допомогою системного сповіщення.

### 1. UI (Jetpack Compose)
На головному екрані (`WaterReminderScreen`) повинні бути наступні елементи:
1. **Перемикач (Switch) "Увімкнути нагадування"**: 
   * Якщо `true` — скасовує попередню задачу і створює нову `PeriodicWorkRequest`.
   * Якщо `false` — повністю скасовує активну задачу (`cancelUniqueWork`).
2. **Текстове поле (OutlinedTextField)**: Для вводу інтервалу нагадувань у хвилинах (наприклад, `15`). *Зверніть увагу: мінімальний системний інтервал становить 15 хвилин!*
3. **Блок статистики (Text)**: Відображає повідомлення *"Нагадувань відправлено: X разів"*.

### 2. Фонова задача (`WaterReminderWorker`)
* Клас успадковується від `CoroutineWorker`.
* Усередині `doWork()` він повинен:
  1. Збільшити лічильник "відправлених нагадувань" на +1 (використовуйте `SharedPreferences` або `DataStore`, оскільки `WorkManager` живе у своєму процесі/фоні і не має прямого доступу до `ViewModel` активності).
  2. Створити та надіслати системне сповіщення (`Notification`) з текстом "Час випити склянку води! 💧".
* **Важливо:** Не забудьте перевірити дозвіл `POST_NOTIFICATIONS` на Android 13+.

---

## Етапи виконання

### Крок 1. Підготовка (Дозволи та Залежності)
1. У `build.gradle.kts` додайте WorkManager: `implementation("androidx.work:work-runtime-ktx:2.9.0")`.
2. У `AndroidManifest.xml` додайте дозвіл `<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />`.

### Крок 2. Реалізація `WaterReminderWorker`
1. Створіть клас `WaterReminderWorker(context, params) : CoroutineWorker(context, params)`.
2. У методі `doWork()` налаштуйте та покажіть `NotificationCompat`:
```kotlin
val manager = applicationContext.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
// ... створення каналу ...
val notification = NotificationCompat.Builder(applicationContext, "water_channel")
    .setContentTitle("Трекер води")
    .setContentText("Час випити склянку води! 💧")
    .setSmallIcon(R.drawable.ic_water_drop)
    .build()
    
manager.notify(System.currentTimeMillis().toInt(), notification)
```
3. *(Опціонально)* Оновіть ваш збережений лічильник у `SharedPreferences`.

### Крок 3. Налаштування ViewModel
У вашій `ViewModel` створіть методи для керування `WorkManager`:

```kotlin
fun startReminder(intervalMinutes: Long) {
    // Враховуємо системне обмеження в 15 хвилин
    val safeInterval = if (intervalMinutes < 15) 15L else intervalMinutes

    val request = PeriodicWorkRequestBuilder<WaterReminderWorker>(
        safeInterval, TimeUnit.MINUTES
    ).build()

    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "water_reminder",
        ExistingPeriodicWorkPolicy.UPDATE, // Або REPLACE, якщо UPDATE недоступний
        request
    )
}

fun stopReminder() {
    WorkManager.getInstance(context).cancelUniqueWork("water_reminder")
}
```

### Крок 4. UI та Тестування
1. Створіть екран Compose із `Switch`. При зміні стану викликайте `viewModel.startReminder()` або `viewModel.stopReminder()`.
2. Додайте дозвіл на сповіщення через `rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission())`.
3. **Тестування:**
   * Запустіть додаток, введіть 15 хвилин, увімкніть Switch.
   * Згорніть додаток.
   * Оскільки чекати 15 хвилин довго, скористайтеся **App Inspection ➔ Background Task Inspector** у Android Studio.
   * Знайдіть вашу задачу `"water_reminder"` і натисніть кнопку **Execute (Запустити зараз)**.
   * Переконайтеся, що на екрані з'явилося сповіщення про воду!

Успіхів! Ви створили корисний і надійний трекер здоров'я на базі Periodic WorkManager!
