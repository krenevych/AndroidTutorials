# Зміст Документації

Тут зібрані всі навчальні матеріали (туторіали) по розробці під Android, поділені за темами.

## 01. Основи та Gradle
* [Основи Gradle (`gradle_guide.md`)](01_gradle/gradle_guide.md) — базове розуміння того, як збирається проект, що таке залежності та як влаштовані файли конфігурації.
* [AndroidManifest (`AndroidManifestTutorial.md`)](01_gradle/AndroidManifestTutorial.md) — детальний розбір головного конфігураційного файлу ("паспорту") застосунку, дозволів та оголошення компонентів.

## 02. ADB (Android Debug Bridge)
* [Команди ADB (`adb_documentation.md`)](02_adb/adb_documentation.md) — довідник команд для взаємодії з пристроєм чи емулятором через термінал (встановлення APK, логування, тощо).

## 03. Побудова інтерфейсу (View Approach)
* [Простий застосунок (`01_SimpleViewAppTutorial.md`)](03_ViewApproach/01_SimpleViewAppTutorial.md) — створення першого застосунку з XML-розміткою, базовими UI-елементами, обробкою натискань через `findViewById` та логуванням.
* [View Binding (`02_ViewBindingTutorial.md`)](03_ViewApproach/02_ViewBindingTutorial.md) — сучасний підхід до взаємодії з UI без використання `findViewById`.
* [Навігація (Intents) (`03_StartActivityTutorial.md`)](03_ViewApproach/03_StartActivityTutorial.md) — як відкривати нові екрани, передавати між ними дані та використовувати неявні інтенти (виклик браузера, пошти тощо) та як працюють Intent Filters.

## 04. Життєвий цикл (Lifecycle)
* [Application (`ApplicationTutorial.md`)](04_Lifecycle/ApplicationTutorial.md) — розбір того, як влаштовані процеси в Android, що таке Application Sandbox та навіщо створювати власний клас-спадкоємець `Application`.