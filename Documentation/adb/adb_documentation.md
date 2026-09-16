# Повний довідник з ADB та ADB Shell для Android-розробника

**ADB (Android Debug Bridge)** — це консольний інструмент для взаємодії між комп'ютером розробника та Android-пристроєм чи емулятором.

---

## ⚡ Швидкий старт: ТОП-10 команд для щоденної роботи

1. Показати список усіх підключених пристроїв та емуляторів:
   ```bash
   adb devices
   ```
2. Встановити APK із заміною існуючої версії (`-r`) та авто-видачею дозволів (`-g`):
   ```bash
   adb install -r -g app-debug.apk
   ```
3. Скинути дані та кеш додатка до початкового стану (ніби щойно встановили):
   ```bash
   adb shell pm clear com.example.app
   ```
4. Фільтрувати логи лише за обраним тегом:
   ```bash
   adb logcat -s "MyTag"
   ```
5. Примусово завершити роботу застосунку (правильний спосіб закрити додаток):
   ```bash
   adb shell am force-stop com.example.app
   ```
6. Надіслати текст у поточне активне поле вводу (зручно для логінів):
   ```bash
   adb shell input text "test@gmail.com"
   ```
7. Симулювати відкриття Deep Link:
   ```bash
   adb shell am start -a android.intent.action.VIEW -d "https://myapp.com/profile"
   ```
8. Зробити скріншот та зберегти на комп'ютер:
   ```bash
   adb exec-out screencap -p > screen.png
   ```
9. Дозволити додатку звертатися до `localhost:8080` вашого комп'ютера (критично для тестування локального бекенду):
   ```bash
   adb reverse tcp:8080 tcp:8080
   ```
10. Швидко увімкнути темну тему:
    ```bash
    adb shell cmd uimode night yes
    ```

---

## Зміст

* <a href="#section1">1. Підключення та вибір пристрою</a>
* <a href="#section2">2. Керування пакетами (pm - Package Manager)</a>
* <a href="#section3">3. Керування застосунками (am - Activity Manager)</a>
* <a href="#section4">4. Системні логи (logcat)</a>
* <a href="#section5">5. Обмін файлами та файлова система</a>
* <a href="#section6">6. Симуляція дій користувача (input)</a>
* <a href="#section7">7. Корисні налаштування розробника</a>
* <a href="#section8">8. Аналіз процесів та ресурсів (Linux рівень)</a>
* <a href="#section9">9. Системна інформація, мережа та налаштування</a>
* <a href="#section10">10. Практичні сценарії налагодження</a>

---

<a name="section1"></a>
## 1. Підключення та вибір пристрою

Якщо підключено кілька девайсів, команди треба спрямовувати на конкретний пристрій. Замість таблиці ось список селекторів:

* Показати всі доступні пристрої (та їхні ID), підключені до комп'ютера:
  ```bash
  adb devices
  ```
* Спрямувати команду на єдиний фізичний USB-пристрій:
  ```bash
  adb -d <команда>
  ```
  _Приклад:_
  ```bash
  adb -d shell
  ```
* Спрямувати команду на єдиний запущений емулятор:
  ```bash
  adb -e <команда>
  ```
  _Приклад:_
  ```bash
  adb -e logcat
  ```
* Виконати команду на конкретному пристрої за його ідентифікатором:
  ```bash
  adb -s <ID> <команда>
  ```
  _Приклад:_
  ```bash
  adb -s emulator-5554 shell
  ```
* Перезавантажити підключений пристрій:
  ```bash
  adb reboot
  ```
* Перенаправити мережевий порт комп'ютера на порт пристрою:
  ```bash
  adb forward <локальний_порт> <віддалений_порт>
  ```
  _Приклад:_
  ```bash
  adb forward tcp:8080 tcp:8080
  ```


### Бездротове підключення (Wi-Fi)
* **Android 11+ (через Pairing):**
  ```bash
  adb pair 192.168.1.105:37123
  adb connect 192.168.1.105:42155
  ```

---

<a name="section2"></a>
## 2. Керування пакетами (`pm` - Package Manager)

Утиліта **`pm` (Package Manager)** відповідає за управління всіма встановленими пакетами (додатками) на пристрої. Вона виконує функції, аналогічні меню "Додатки" в налаштуваннях Android, але через консоль.
**Основні зони відповідальності:** інсталяція та деінсталяція APK, керування дозволами (permissions), очищення даних додатків та надання детальної інформації про встановлені застосунки.

### Встановлення та деінсталяція

* Встановити APK-файл на пристрій:
  ```bash
  adb install <шлях_до_apk>
  ```
  _Приклад:_
  ```bash
  adb install app.apk
  ```
* Встановити з перезаписом (`-r`) та дозволом понизити версію (`-d`):
  ```bash
  adb install -r -d <шлях_до_apk>
  ```
  _Приклад:_
  ```bash
  adb install -r -d app.apk
  ```
* Повністю видалити додаток із пристрою:
  ```bash
  adb uninstall <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb uninstall com.example.app
  ```
* Видалити застосунок зі збереженням кешу та даних:
  ```bash
  adb uninstall -k <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb uninstall -k com.example.app
  ```
* Встановити APK, розташований у файловій системі пристрою:
  ```bash
  adb shell pm install <шлях_на_пристрої>
  ```
  _Приклад:_
  ```bash
  adb shell pm install /sdcard/app.apk
  ```
* Виконати деінсталяцію пакета через середовище shell:
  ```bash
  adb shell pm uninstall <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb shell pm uninstall com.example.app
  ```

### Робота з дозволами та перевірка пакетів

* Вивести повний список встановлених пакетів:
  ```bash
  adb shell pm list packages
  ```
* Вивести список тільки сторонніх додатків:
  ```bash
  adb shell pm list packages -3
  ```
* Знайти пакет за ключовим словом:
  ```bash
  adb shell pm list packages | grep <слово>
  ```
  _Приклад:_
  ```bash
  adb shell pm list packages | grep example
  ```
* Надати застосунку вказаний дозвіл:
  ```bash
  adb shell pm grant <пакет> <дозвіл>
  ```
  _Приклад:_
  ```bash
  adb shell pm grant com.example.app android.permission.CAMERA
  ```
* Відкликати дозвіл у застосунку:
  ```bash
  adb shell pm revoke <пакет> <дозвіл>
  ```
  _Приклад:_
  ```bash
  adb shell pm revoke com.example.app android.permission.CAMERA
  ```
* Повністю очистити локальне сховище та кеш додатка:
  ```bash
  adb shell pm clear <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb shell pm clear com.example.app
  ```
* Дізнатися абсолютний шлях до APK-файлу на пристрої:
  ```bash
  adb shell pm path <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb shell pm path com.example.app
  ```

### Як витягнути APK з телефону на комп'ютер:
1. Знаходимо шлях: `adb shell pm path com.example.app`
2. Завантажуємо: `adb pull /data/app/.../base.apk ./myapp.apk`

---

<a name="section3"></a>
## 3. Керування застосунками (`am` - Activity Manager)

Утиліта **`am` (Activity Manager)** є ключовим інструментом для керування життєвим циклом Android-компонентів. Вона взаємодіє безпосередньо з ядром системи, яке контролює поведінку додатків.
**Основні зони відповідальності:** запуск екранів (Activities) та фонових служб (Services), надсилання широкомовних повідомлень (Broadcasts), передача даних через Intents, симуляція системних подій (наприклад, завантаження пристрою) та примусове завершення процесів додатків.

* `adb shell am start -n com.example.app/.MainActivity` — запустити конкретну Activity.
* `adb shell am start -S -n com.example.app/.MainActivity` — зупинити процес перед запуском (`-S`).
* `adb shell am force-stop com.example.app` — повністю вбити процес додатка (краще, ніж просто змахнути з недавніх).
* `adb shell am broadcast -a android.intent.action.BOOT_COMPLETED` — симулювати перезавантаження пристрою.

### Передача параметрів (Extras) при запуску
```bash
adb shell am start -n com.example.app/.DetailsActivity \
  --es "user_name" "Alex" \
  --ei "user_id" 1042 \
  --ez "is_admin" true
```

---

<a name="section4"></a>
## 4. Системні логи (`logcat`)

* `adb logcat` — відкрити потік логів.
* `adb logcat -c` — очистити буфер логів (видалити старі записи).
* `adb logcat *:E` — показувати **тільки помилки** (Error та Fatal).
* `adb logcat -s MyTag:D *:S` — показати логи тільки за тегом `MyTag` (рівень Debug і вище).
* `adb logcat --pid=$(adb shell pidof -s com.example.app) — фільтрувати вивід лише для PID конкретного додатка.`

---

<a name="section5"></a>
## 5. Обмін файлами та файлова система

### Обмін файлами з комп'ютером
* `adb push notes.txt /sdcard/notes.txt` — скопіювати файл із комп'ютера на девайс.
* `adb pull /sdcard/log.txt ./log.txt` — завантажити файл із девайса на робочу станцію.

### Внутрішні команди оболонки
* `adb shell ls /sdcard/` — переглянути вміст директорії.
* `adb shell cd /data/data/` — перейти до іншого каталогу.
* `adb shell pwd` — вивести поточний шлях.
* `adb shell mkdir /sdcard/myfolder` — створити новий каталог.
* `adb shell rm /sdcard/old.txt` — видалити файл.
* `adb shell rm -r /sdcard/photos` — видалити директорію рекурсивно.
* `adb shell cp /sdcard/a.txt /sdcard/b.txt` — скопіювати файл.
* `adb shell mv /sdcard/old.txt /sdcard/new.txt` — перемістити або перейменувати файл.
* `adb shell cat /proc/cpuinfo` — переглянути текстовий вміст файлу.
* `adb shell "echo Hello > /sdcard/hello.txt"` — записати рядок у файл.
* `adb shell chmod 777 /sdcard/script.sh` — встановити права доступу.
* `adb shell chown shell:shell /sdcard/file.txt` — змінити власника файла.
* `adb shell exit` — завершити сесію оболонки.

---

<a name="section6"></a>
## 6. Симуляція дій користувача (`input`)

Дуже корисно для тестування без мишки/пальця:
* `adb shell input tap 500 500` — клік по екрану за координатами (X, Y).
* `adb shell input text "my_password"` — швидко вставити текст у поле (увага: не підтримує пробіли напряму).
* `adb shell input keyevent 4` — натиснути апаратну кнопку **"Назад"**.
* `adb shell input keyevent 3` — натиснути апаратну кнопку **"Home"**.
* `adb shell input keyevent 82` — розблокувати екран.

---

<a name="section7"></a>
## 7. Корисні налаштування розробника

* `adb shell cmd uimode night yes` — **Увімкнути темну тему**.
* `adb shell cmd uimode night no` — **Увімкнути світлу тему**.
* `adb shell settings put global transition_animation_scale 0` — **Вимкнути анімації** (швидше працюють автотести).
* `adb shell dumpsys meminfo com.example.app` — подивитись скільки оперативної пам'яті їсть ваш додаток.
* `adb shell screenrecord /sdcard/video.mp4` — записати відео роботи екрана (зупинити: Ctrl+C).

---

<a name="section8"></a>
## 8. Аналіз процесів та ресурсів (Linux рівень)

Оскільки Android базується на ядрі Linux, ви можете використовувати стандартні Unix-утиліти для моніторингу процесів на низькому рівні. Це корисно для глибокого дебагінгу.

* `adb shell ps` або `adb shell ps -A` — відобразити список усіх активних процесів у системі.
* `adb shell pidof com.example.app` — дізнатися числовий ідентифікатор (PID) конкретного пакета (додатка).
* `adb shell top -n 1` — переглянути поточне завантаження процесора (CPU) різними процесами.
* `adb shell kill <PID>` — м'яко завершити процес за його PID (відправити SIGTERM).
* `adb shell kill -9 <PID>` — примусово знищити процес сигналом SIGKILL (жорстке завершення).
* `adb shell cat /proc/<PID>/status` — вивести детальний системний статус процесу.
* `adb shell dumpsys gfxinfo com.example.app framestats` — переглянути статистику швидкості рендерингу кадрів (корисно для пошуку UI-лагів).

---

<a name="section9"></a>
## 9. Системна інформація, мережа та налаштування

* `adb shell getprop ro.build.version.release` — переглянути версію операційної системи Android.
* `adb shell df -h` — відобразити зайняте та вільне місце на дискових розділах.
* `adb shell free` — показати статистику використання оперативної пам'яті.
* `adb shell uname -a` — вивести системну інформацію про версію ядра Linux.
* `adb shell ifconfig` або `adb shell ip addr` — переглянути конфігурацію мережевих інтерфейсів та IP-адреси.
* `adb shell ping google.com` — перевірити доступність мережевого вузла.
* `adb shell netstat -an` — показати перелік відкритих портів та з'єднань.
* `adb shell ss -tulnp` — вивести статистику використання мережевих сокетів.
* `adb shell settings get system screen_brightness` — отримати поточне системне значення параметра.
* `adb shell settings put system screen_brightness 150` — записати нове значення системного налаштування.
* `adb shell screencap /sdcard/screen.png` — зробити знімок екрана та зберегти на девайсі.
* `adb shell screenrecord /sdcard/video.mp4` — записати відео роботи інтерфейсу в файл.

---


<a name="section10"></a>
## 10. Практичні сценарії налагодження

### Сценарій 1: Пошук, перевірка та примусове закриття процесу
```bash
adb shell ps | grep com.example.myapp
adb shell pidof com.example.myapp
adb shell cat /proc/$(adb shell pidof com.example.myapp)/status
adb shell kill -9 $(adb shell pidof com.example.myapp)
```

### Сценарій 2: Моніторинг використання процесора конкретним додатком
```bash
adb shell top -n 1 | grep com.example.myapp
```

### Сценарій 3: Швидка перевірка стану процесу (живий чи зупинений)
```bash
adb shell pidof com.example.myapp && echo "Running" || echo "Stopped"
```

### Сценарій 4: Перегляд усіх запущених потоків додатка
```bash
adb shell ls /proc/$(adb shell pidof com.example.myapp)/task
```

### Сценарій 5: Діагностика витоків пам'яті (Memory Leaks)
```bash
adb shell dumpsys meminfo com.example.myapp
```

