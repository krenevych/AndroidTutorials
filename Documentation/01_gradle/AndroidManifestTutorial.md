# Що таке AndroidManifest.xml?

Будь-який Android-застосунок обов'язково повинен містити файл `AndroidManifest.xml` у корені набору сирців (зазвичай він знаходиться за шляхом `app/src/main/AndroidManifest.xml`). 

Цей файл — це своєрідний **"паспорт"** вашого застосунку. Він надає операційній системі Android, а також магазину Google Play, найважливішу інформацію про вашу програму ще до того, як вона буде запущена.

## Основні функції Маніфесту:
1. **Оголошення компонентів:** Всі екрани (Activities), фонові служби (Services) та інші базові компоненти повинні бути записані в маніфесті. Якщо ви створите нову Activity, але не додасте її в маніфест — програма впаде (краш) при спробі її відкрити.
2. **Дозволи (Permissions):** Якщо вашому застосунку потрібен доступ до інтернету, камери, геолокації чи контактів користувача, це *обов'язково* треба вказати тут.
3. **Метадані застосунку:** Тут визначається іконка програми, її назва, тема оформлення та інші загальні налаштування.
4. **Точка входу:** Маніфест вказує системі, який саме екран (Activity) потрібно відкрити першим при натисканні на іконку застосунку на телефоні.

## Структура файлу

Давайте розглянемо типовий `AndroidManifest.xml` для простого застосунку:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- ДОЗВОЛИ (Permissions) -->
    <!-- Наприклад, дозвіл на використання Інтернету -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- ГОЛОВНИЙ БЛОК APPLICATION -->
    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"         <!-- Іконка застосунку -->
        android:label="@string/app_name"           <!-- Назва застосунку -->
        android:roundIcon="@mipmap/ic_launcher_round" <!-- Кругла іконка -->
        android:supportsRtl="true"
        android:theme="@style/Theme.MyApplication" <!-- Тема оформлення -->
        tools:targetApi="31">

        <!-- ОГОЛОШЕННЯ ЕКРАНУ (Activity) -->
        <activity
            android:name=".MainActivity"
            android:exported="true">
            
            <!-- INTENT FILTER - вказує, що ця Activity є точкою входу -->
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

        <!-- Якщо ви створюєте ще один екран, наприклад DetailActivity, 
             його треба просто додати сюди (без intent-filter): -->
        <activity android:name=".DetailActivity" />

    </application>
</manifest>
```

## Розбір ключових тегів

### `<manifest>`
Це кореневий тег. У старих версіях Android тут вказувався `package="com.example.app"`, але зараз в сучасних проектах (з Gradle) ідентифікатор пакету перенесено в файл `build.gradle.kts` (як `namespace`). 

### `<uses-permission>`
Використовується для запиту дозволів у системи. Без запису `<uses-permission android:name="android.permission.INTERNET" />` ваш застосунок не зможе завантажити жодного байту з мережі, навіть якщо ви напишете повністю правильний код. Деякі дозволи (як інтернет) надаються автоматично, а небезпечні (камера, локація) — вимагатимуть додаткового запиту у користувача під час роботи програми (Runtime Permissions).

### `<application>`
Обгортка для всіх компонентів застосунку. Тут налаштовуються глобальні речі:
* `android:icon` — посилання на іконку (зазвичай знаходиться в папці `res/mipmap/`).
* `android:label` — назва, яка буде відображатися під іконкою на робочому столі телефону.
* `android:theme` — глобальна тема/стилі програми (кольори, шрифти).

### `<activity>`
Кожен екран у класичному Android представлений класом `Activity` (або ж використовується одна `Activity` як контейнер, що зараз є нормою для Jetpack Compose та Navigation Component). 
Атрибут `android:exported="true"` означає, що цю Activity можуть запускати інші застосунки (або операційна система). Для головного екрану це значення обов'язково має бути `true`.

### `<intent-filter>` з `MAIN` та `LAUNCHER`
Цей блок всередині `<activity>` повідомляє Android наступне: *"Ось цей екран є головним (`MAIN`), і його треба показати у списку всіх програм телефону (`LAUNCHER`)"*. 
Якщо ви приберете цей блок, застосунок збереться і встановиться, але на телефоні не з'явиться його іконка і користувач не зможе його запустити!

## Підсумок
`AndroidManifest.xml` — це головний конфігураційний файл, який описує архітектуру застосунку для операційної системи Android. 

**Головне правило:** якщо ви створили новий екран, додали фоновий сервіс або вам знадобився доступ до системних функцій телефону (інтернет, контакти, блютуз) — перше, що треба зробити, це перевірити та оновити Маніфест!