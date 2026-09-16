# Довідник з ADB та ADB Shell для Android-розробника

**ADB (Android Debug Bridge)** — це консольний інструмент для взаємодії між комп'ютером розробника та Android-пристроєм чи емулятором.

### ADB Shell

Операційна система Android побудована на базі ядра Linux. У той час як базові команди (такі як `adb devices`, `adb install`, `adb push`) є програмами вашого комп'ютера, що лише взаємодіють з телефоном, команда **`adb shell`** відкриває віддалений доступ до внутрішнього термінала самого Android-пристрою. 

Додаючи слово `shell` (`adb shell <команда>`), ви змушуєте Android виконати цю команду "всередині" себе. Це дозволяє:
* Взаємодіяти з ядром та системними сервісами (наприклад, управління пакетами `pm` або вікнами `am`).
* Керувати файловою системою Linux на пристрої (`ls`, `rm`, `cat`, `mkdir`).
* Симулювати дотики до екрана, змінювати системні налаштування та працювати з процесами (PID).

Ви також можете просто написати `adb shell` та натиснути Enter — це відкриє інтерактивну сесію, де ви зможете вводити команди так, ніби працюєте безпосередньо у терміналі на самому телефоні.

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

Якщо підключено кілька девайсів, команди треба спрямовувати на конкретний пристрій.

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

* Запустити конкретну Activity:
  ```bash
  adb shell am start -n <пакет>/<повний_шлях_до_класу>
  ```
  _Приклад:_
  ```bash
  adb shell am start -n com.example.app/.MainActivity
  ```
* Зупинити процес перед запуском (`-S`):
  ```bash
  adb shell am start -S -n <пакет>/<повний_шлях_до_класу>
  ```
  _Приклад:_
  ```bash
  adb shell am start -S -n com.example.app/.MainActivity
  ```
* Повністю вбити процес додатка (краще, ніж просто змахнути з недавніх):
  ```bash
  adb shell am force-stop <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb shell am force-stop com.example.app
  ```
* Симулювати перезавантаження пристрою:
  ```bash
  adb shell am broadcast -a android.intent.action.BOOT_COMPLETED
  ```

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

* Відкрити потік логів:
  ```bash
  adb logcat
  ```
* Очистити буфер логів (видалити старі записи):
  ```bash
  adb logcat -c
  ```
* Показувати **тільки помилки** (Error та Fatal):
  ```bash
  adb logcat *:E
  ```
* Показати логи тільки за тегом `MyTag` (рівень Debug і вище):
  ```bash
  adb logcat -s <Тег>:<Рівень> *:<S>
  ```
  _Приклад:_
  ```bash
  adb logcat -s MyTag:D *:S
  ```
* Фільтрувати вивід лише для PID конкретного додатка:
  ```bash
  adb logcat --pid=$(adb shell pidof -s <назва_пакета>)
  ```
  _Приклад:_
  ```bash
  adb logcat --pid=$(adb shell pidof -s com.example.app)
  ```

---

<a name="section5"></a>
## 5. Обмін файлами та файлова система

### Обмін файлами з комп'ютером

* Скопіювати файл із комп'ютера на девайс:
  ```bash
  adb push <локальний_шлях> <віддалений_шлях>
  ```
  _Приклад:_
  ```bash
  adb push notes.txt /sdcard/notes.txt
  ```
* Завантажити файл із девайса на робочу станцію:
  ```bash
  adb pull <віддалений_шлях> <локальний_шлях>
  ```
  _Приклад:_
  ```bash
  adb pull /sdcard/log.txt ./log.txt
  ```

### Внутрішні команди оболонки

* Переглянути вміст директорії:
  ```bash
  adb shell ls <шлях>
  ```
  _Приклад:_
  ```bash
  adb shell ls /sdcard/
  ```
* Перейти до іншого каталогу:
  ```bash
  adb shell cd <шлях>
  ```
  _Приклад:_
  ```bash
  adb shell cd /data/data/
  ```
* Вивести поточний шлях:
  ```bash
  adb shell pwd
  ```
* Створити новий каталог:
  ```bash
  adb shell mkdir <шлях>
  ```
  _Приклад:_
  ```bash
  adb shell mkdir /sdcard/myfolder
  ```
* Видалити файл:
  ```bash
  adb shell rm <шлях_до_файлу>
  ```
  _Приклад:_
  ```bash
  adb shell rm /sdcard/old.txt
  ```
* Видалити директорію рекурсивно:
  ```bash
  adb shell rm -r <шлях_до_каталогу>
  ```
  _Приклад:_
  ```bash
  adb shell rm -r /sdcard/photos
  ```
* Скопіювати файл:
  ```bash
  adb shell cp <джерело> <призначення>
  ```
  _Приклад:_
  ```bash
  adb shell cp /sdcard/a.txt /sdcard/b.txt
  ```
* Перемістити або перейменувати файл:
  ```bash
  adb shell mv <старий_шлях> <новий_шлях>
  ```
  _Приклад:_
  ```bash
  adb shell mv /sdcard/old.txt /sdcard/new.txt
  ```
* Переглянути текстовий вміст файлу:
  ```bash
  adb shell cat <шлях_до_файлу>
  ```
  _Приклад:_
  ```bash
  adb shell cat /proc/cpuinfo
  ```
* Записати рядок у файл:
  ```bash
  adb shell "echo <текст> > <файл>"
  ```
  _Приклад:_
  ```bash
  adb shell "echo Hello > /sdcard/hello.txt"
  ```
* Встановити права доступу:
  ```bash
  adb shell chmod <права> <файл>
  ```
  _Приклад:_
  ```bash
  adb shell chmod 777 /sdcard/script.sh
  ```
* Змінити власника файла:
  ```bash
  adb shell chown <власник>:<група> <файл>
  ```
  _Приклад:_
  ```bash
  adb shell chown shell:shell /sdcard/file.txt
  ```
* Завершити сесію оболонки:
  ```bash
  adb shell exit
  ```

---

<a name="section6"></a>
## 6. Симуляція дій користувача (`input`)

Дуже корисно для тестування без мишки/пальця:
* Клік по екрану за координатами (X, Y):
  ```bash
  adb shell input tap <X> <Y>
  ```
  _Приклад:_
  ```bash
  adb shell input tap 500 500
  ```
* Швидко вставити текст у поле (увага: не підтримує пробіли напряму):
  ```bash
  adb shell input text "<текст>"
  ```
  _Приклад:_
  ```bash
  adb shell input text "my_password"
  ```
* Натиснути апаратну кнопку **"Назад"**:
  ```bash
  adb shell input keyevent 4
  ```
* Натиснути апаратну кнопку **"Home"**:
  ```bash
  adb shell input keyevent 3
  ```
* Розблокувати екран:
  ```bash
  adb shell input keyevent 82
  ```

---

<a name="section7"></a>
## 7. Корисні налаштування розробника

* **Увімкнути темну тему**:
  ```bash
  adb shell cmd uimode night yes
  ```
* **Увімкнути світлу тему**:
  ```bash
  adb shell cmd uimode night no
  ```
* **Вимкнути анімації** (швидше працюють автотести):
  ```bash
  adb shell settings put global transition_animation_scale 0
  ```
* Подивитись скільки оперативної пам'яті їсть ваш додаток:
  ```bash
  adb shell dumpsys meminfo <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb shell dumpsys meminfo com.example.app
  ```
* Записати відео роботи екрана (зупинити: Ctrl+C):
  ```bash
  adb shell screenrecord <шлях_збереження_на_пристрої>
  ```
  _Приклад:_
  ```bash
  adb shell screenrecord /sdcard/video.mp4
  ```

---

<a name="section8"></a>
## 8. Аналіз процесів та ресурсів (Linux рівень)

Оскільки Android базується на ядрі Linux, ви можете використовувати стандартні Unix-утиліти для моніторингу процесів на низькому рівні. Це корисно для глибокого дебагінгу.

* Відобразити список усіх активних процесів у системі:
  ```bash
  adb shell ps
  # або
  adb shell ps -A
  ```
* Дізнатися числовий ідентифікатор (PID) конкретного пакета (додатка):
  ```bash
  adb shell pidof <назва_пакета>
  ```
  _Приклад:_
  ```bash
  adb shell pidof com.example.app
  ```
* Переглянути поточне завантаження процесора (CPU) різними процесами:
  ```bash
  adb shell top -n 1
  ```
* М'яко завершити процес за його PID (відправити SIGTERM):
  ```bash
  adb shell kill <PID>
  ```
* Примусово знищити процес сигналом SIGKILL (жорстке завершення):
  ```bash
  adb shell kill -9 <PID>
  ```
* Вивести детальний системний статус процесу:
  ```bash
  adb shell cat /proc/<PID>/status
  ```
* Переглянути статистику швидкості рендерингу кадрів (корисно для пошуку UI-лагів):
  ```bash
  adb shell dumpsys gfxinfo <назва_пакета> framestats
  ```
  _Приклад:_
  ```bash
  adb shell dumpsys gfxinfo com.example.app framestats
  ```

---

<a name="section9"></a>
## 9. Системна інформація, мережа та налаштування

* Переглянути версію операційної системи Android:
  ```bash
  adb shell getprop ro.build.version.release
  ```
* Відобразити зайняте та вільне місце на дискових розділах:
  ```bash
  adb shell df -h
  ```
* Показати статистику використання оперативної пам'яті:
  ```bash
  adb shell free
  ```
* Вивести системну інформацію про версію ядра Linux:
  ```bash
  adb shell uname -a
  ```
* Переглянути конфігурацію мережевих інтерфейсів та IP-адреси:
  ```bash
  adb shell ifconfig
  # або
  adb shell ip addr
  ```
* Перевірити доступність мережевого вузла:
  ```bash
  adb shell ping <адреса>
  ```
  _Приклад:_
  ```bash
  adb shell ping google.com
  ```
* Показати перелік відкритих портів та з'єднань:
  ```bash
  adb shell netstat -an
  ```
* Вивести статистику використання мережевих сокетів:
  ```bash
  adb shell ss -tulnp
  ```
* Отримати поточне системне значення параметра:
  ```bash
  adb shell settings get <namespace> <ключ>
  ```
  _Приклад:_
  ```bash
  adb shell settings get system screen_brightness
  ```
* Записати нове значення системного налаштування:
  ```bash
  adb shell settings put <namespace> <ключ> <значення>
  ```
  _Приклад:_
  ```bash
  adb shell settings put system screen_brightness 150
  ```
* Зробити знімок екрана та зберегти на девайсі:
  ```bash
  adb shell screencap <шлях_збереження>
  ```
  _Приклад:_
  ```bash
  adb shell screencap /sdcard/screen.png
  ```
* Записати відео роботи інтерфейсу в файл на девайсі:
  ```bash
  adb shell screenrecord <шлях_збереження>
  ```
  _Приклад:_
  ```bash
  adb shell screenrecord /sdcard/video.mp4
  ```

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

