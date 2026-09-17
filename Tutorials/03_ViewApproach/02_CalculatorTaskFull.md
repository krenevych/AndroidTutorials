# Завдання: Повноцінний Калькулятор (View Approach)

## Мета завдання
Навчитися створювати складніші інтерфейси з великою кількістю елементів, працювати з `GridLayout` або `ConstraintLayout`, а також керувати внутрішнім станом застосунку (зберігати проміжні дані між натисканнями кнопок).

Ви створите повноцінний калькулятор, який виглядає та працює як стандартний калькулятор у вашому смартфоні (з розкладкою кнопок від 0 до 9 та кнопкою дорівнює "=").

## Вимоги до інтерфейсу (UI)
Ваш екран (в `activity_main.xml`) має містити:

1. **Екран калькулятора (TextView):**
   * Одне велике текстове поле зверху, яке буде відображати поточний ввід або результат обчислень.
   * Текст має бути вирівняний по правому краю (використовуйте `android:gravity="end"`).
2. **Сітка кнопок (Кнопки 0-9, операції та інше):**
   * Кнопки з цифрами від `0` до `9`.
   * Кнопки базових операцій: `+`, `-`, `*`, `/`.
   * Кнопка результату: `=`.
   * Кнопка очищення: `C` (Clear).
   * *Порада:* Для рівномірного розміщення такої кількості кнопок у вигляді сітки найкраще підійде `GridLayout` або сучасний `ConstraintLayout`. У випадку `GridLayout` ви можете задати кількість колонок (наприклад, `android:columnCount="4"`).

### Візуальна схема (Мокап)
Ось приблизно як має виглядати ваш калькулятор на екрані пристрою (це схема для орієнтиру, можете зробити свій дизайн):

<div>
  <table style="width: 100%; max-width: 198px; border-collapse: collapse; font-family: sans-serif; text-align: center; table-layout: fixed; border: 1px solid #ccc; margin: 0 auto;">
    <tr>
      <td colspan="4" style="text-align: right; font-size: 32px; padding: 20px; background-color: #e0e0e0; font-family: monospace; border: 1px solid #ccc;"><b>3.141592</b></td>
    </tr>
    <tr style="font-size: 20px;">
      <td style="width: 25%; padding: 15px 0; background-color: #ffcccc; border: 1px solid #ccc;"><b>C</b></td>
      <td style="width: 25%; padding: 15px 0; background-color: #dcdcdc; border: 1px solid #ccc;"><b>()</b></td>
      <td style="width: 25%; padding: 15px 0; background-color: #dcdcdc; border: 1px solid #ccc;"><b>%</b></td>
      <td style="width: 25%; padding: 15px 0; background-color: #ffe4b5; border: 1px solid #ccc;"><b>/</b></td>
    </tr>
    <tr style="font-size: 20px;">
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>7</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>8</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>9</b></td>
      <td style="padding: 15px 0; background-color: #ffe4b5; border: 1px solid #ccc;"><b>*</b></td>
    </tr>
    <tr style="font-size: 20px;">
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>4</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>5</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>6</b></td>
      <td style="padding: 15px 0; background-color: #ffe4b5; border: 1px solid #ccc;"><b>-</b></td>
    </tr>
    <tr style="font-size: 20px;">
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>1</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>2</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>3</b></td>
      <td style="padding: 15px 0; background-color: #ffe4b5; border: 1px solid #ccc;"><b>+</b></td>
    </tr>
    <tr style="font-size: 20px;">
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>0</b></td>
      <td style="padding: 15px 0; background-color: #ffffff; border: 1px solid #ccc;"><b>.</b></td>
      <td style="padding: 15px 0; background-color: #dcdcdc; border: 1px solid #ccc;"><b>⌫</b></td>
      <td style="padding: 15px 0; background-color: #ffe4b5; border: 1px solid #ccc;"><b>=</b></td>
    </tr>
  </table>
</div>
<br/>

## Технічні вимоги (Логіка)

1. Проект **обов'язково** повинен використовувати **ViewBinding** (оскільки кнопок дуже багато, використання `findViewById` заборонено — це зробить код занадто громіздким).
2. **Керування станом:** Оскільки користувач вводить числа по одній цифрі, ваша `MainActivity` повинна мати змінні на рівні класу (властивості), щоб "пам'ятати":
   * Перше введене число (до натискання на операцію).
   * Обрану математичну операцію (наприклад, у вигляді рядка `"+"` чи Enum).
   * Стан: чи вводиться зараз перше число, чи вже друге.
3. **Логіка натискання на цифри:**
   * При натисканні на будь-яку цифру, вона має дописуватись до тексту, який зараз на екрані (`TextView`). 
4. **Логіка натискання на операцію (+, -, *, /):**
   * Поточне значення з екрану зберігається в змінну як "перше число".
   * Обрана операція запам'ятовується.
   * Екран очищується (або готується до вводу нового числа).
5. **Логіка натискання на "=":**
   * Зчитується поточне значення з екрану як "друге число".
   * Виконується збережена математична операція між першим та другим числом.
   * Результат виводиться на екран.
6. **Кнопка "С" (Clear):**
   * Повністю очищує екран (встановлює "0") та скидає всі збережені змінні (перше число, обрану операцію).
7. **Обробка помилок:**
   * Не забудьте обробити ділення на нуль (виводити "Помилка" замість крашу застосунку).

## Додаткове завдання (із зірочкою ⭐)
* **Підтримка дробових чисел:** Додайте кнопку крапки `.` для вводу десяткових дробів. Переконайтеся, що користувач не може ввести дві крапки в одному числі (наприклад, `5.5.2` — це помилка).
* **Видалення останнього символу:** Додайте кнопку `⌫` (Backspace / Delete), яка видаляє лише останню введену цифру, а не весь екран.
* **Безперервні обчислення:** Зробіть так, щоб користувач міг рахувати ланцюжком. Наприклад, ввів `2`, натиснув `+`, ввів `3`, натиснув `+` (в цей момент екран показує проміжний результат `5` і чекає наступне число).

Успіхів!