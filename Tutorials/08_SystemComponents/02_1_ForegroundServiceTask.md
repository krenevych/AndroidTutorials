# Завдання: Створення Foreground Service та керування дозволами в Android 14+

## Мета завдання
Опанувати процес створення безперервної фонової служби (**`Foreground Service`**), навчитися створювати системне сповіщення (`Notification`) із каналом, налаштовувати дозволи відповідно до сучасних вимог Android 14+ та запускати/зупиняти службу з UI-інтерфейсу (через Jetpack Compose).

---

## Вимоги до застосунку

Вам потрібно реалізувати фоновий таймер-секундомір (Tracker), який продовжує відраховувати час навіть коли користувач згорнув додаток.

### 1. UI (Compose)
На екрані повинні бути дві кнопки:
* **"Старт Таймер"** — запускає `Foreground Service`.
* **"Стоп Таймер"** — зупиняє роботу служби.

### 2. Маніфест та Дозволи
Служба має виконувати роботу, що підпадає під категорію `specialUse` (оскільки це просто таймер). Відповідно, ви повинні:
* Зареєструвати `Service` в маніфесті та вказати `android:foregroundServiceType="specialUse"`.
* Запросити базовий дозвіл `FOREGROUND_SERVICE`.
* Запросити специфічний дозвіл `FOREGROUND_SERVICE_SPECIAL_USE`.
* Вказати причину `specialUse` через тег `<property>` у маніфесті (обов'язково для цього типу).
* Запитати Runtime-дозвіл на показ сповіщень **`POST_NOTIFICATIONS`** (для Android 13+).

### 3. Клас Служби (`TimerForegroundService`)
* Під час старту (`onStartCommand`) служба повинна створити `NotificationChannel` та `Notification`, а потім негайно викликати `startForeground()`.
* Після виклику `startForeground()` запускається Корутина (наприклад, через `CoroutineScope`), яка раз на секунду `delay(1000)` робить `Log.d("Timer", "Минуло $seconds секунд")`.
* При виклику `stopSelf()` або кліку "Стоп", корутина має бути скасована.

---

## Етапи виконання

### Крок 1. Налаштування `AndroidManifest.xml`
1. Додайте необхідні 3 дозволи (`POST_NOTIFICATIONS`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_SPECIAL_USE`).
2. Додайте реєстрацію сервісу:
```xml
<service
    android:name=".TimerForegroundService"
    android:foregroundServiceType="specialUse">
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="Running a background timer for the user" />
</service>
```

### Крок 2. Реалізація `TimerForegroundService`
1. Створіть клас `TimerForegroundService : Service()`.
2. Поверніть `null` у `onBind` (нам не потрібна прив'язана служба).
3. Перевизначте `onCreate()`, де створіть `NotificationChannel`.
4. У `onStartCommand()` створіть сповіщення `NotificationCompat.Builder` і викличте `startForeground(1, notification)`.
5. Додайте `CoroutineScope` з диспатчером `Dispatchers.Default` і запустіть безкінечний цикл із `delay(1000)` та логуванням.
6. У `onDestroy()` обов'язково зробіть `scope.cancel()`.

### Крок 3. Налаштування екрану та запит дозволу
1. У вашій `Activity` / Compose-екрані запитайте дозвіл на сповіщення `POST_NOTIFICATIONS` за допомогою стандартного ланцюжка `rememberLauncherForActivityResult`.
2. Реалізуйте кнопки запуску і зупинки, використовуючи команди:
```kotlin
val intent = Intent(context, TimerForegroundService::class.java)

// Старт
ContextCompat.startForegroundService(context, intent)

// Зупинка
context.stopService(intent)
```

### Крок 4. Тестування
1. Запустіть додаток, надайте дозвіл на сповіщення.
2. Натисніть "Старт Таймер". 
3. Переконайтеся, що в шторці телефону з'явилося сповіщення, а в Logcat пішов відлік секунд.
4. **Згорніть додаток (кнопка Home).** Переконайтеся, що відлік у Logcat продовжується!
5. Поверніться в додаток, натисніть "Стоп" і переконайтеся, що сповіщення зникло, а логи зупинилися.

Успіхів у підкоренні фонових служб Android!
