РУССКИЙ 🇷🇺
1. Что такое AndroidIDE?

AndroidIDE — это среда разработки Android-приложений, работающая напрямую на Android-устройстве (без ПК). В отличие от обычных редакторов кода, AndroidIDE включает полноценную сборку Gradle, поддержку XML-ресурсов и возможность компилировать APK прямо на телефоне или планшете .

Ключевые особенности оригинального проекта:

· Полноценная IDE с подсветкой синтаксиса, автодополнением и рефакторингом
· Встроенный Termux для выполнения shell-команд
· Поддержка Gradle — сборка проектов без компьютера
· Открытый исходный код на GitHub (лицензия GPL-3.0)

Однако у AndroidIDE есть важная зависимость — библиотека HiddenApiBypass от LSPosed, которая позволяет приложению обходить ограничения Android на использование скрытых API (например, внутренних методов ActivityThread или PackageManager).

---

2. Почему оригинальный AndroidIDE падал на Android 15?

Техническая причина краша

Начиная с Android 15 (API 35), Google изменил внутреннюю реализацию механизма, отвечающего за ограничения скрытых API. Старые версии библиотеки HiddenApiBypass (до 5.1 включительно) использовали прямые JNI-вызовы к структурам ART (Android Runtime), которые в Android 15 были либо удалены, либо изменены .

Когда AndroidIDE пытался выполнить:

```java
HiddenApiBypass.setHiddenApiExemptions(...);
```

происходило следующее:

1. JNI-код обращался к несуществующему адресу в памяти (fault addr 0x0000000c)
2. Система убивала процесс сигналом SIGSEGV (нарушение сегментации)
3. В логах появлялась ошибка: Cause: null pointer dereference

Почему именно classes3.dex?

В лог-файле краша видно:

```
#02 pc 0000000000602c18 [anon:dalvik-classes3.dex] (org.lsposed.hiddenapibypass.HiddenApiBypass.getDeclaredMethods+180)
```

Это означает, что проблемный код находился в файле classes3.dex — третьем по счёту DEX-файле в составе APK. Android использует несколько DEX-файлов, когда приложение превышает лимит в 64K методов на один файл. В classes3.dex лежали не только классы HiddenApiBypass, но и другой код AndroidIDE (всего ~8 МБ).

---

3. Как я это исправил (техническое описание мода)

Мой мод решает проблему кардинально: заменяет устаревшую библиотеку на актуальную версию 6.1, которая совместима с Android 15.

Этапы модификации

Этап 1: Подготовка новой библиотеки

Была скачана последняя версия HiddenApiBypass (6.1) с официального GitHub LSPosed. В этой версии разработчики:

· Переписали JNI-слой для работы с новыми структурами ART
· Добавили динамическое определение версии Android для выбора правильного метода обхода
· Оптимизировали производительность (с 212 мс в ранних версиях 6.x до 15 мс в финальной) 

Этап 2: Извлечение и конвертация

Из файла hiddenapibypass-6.1.aar (Android Archive) был извлечён classes.jar, а затем сконвертирован в DEX-формат. Получился файл размером ~14 КБ — это все необходимые классы библиотеки в скомпилированном для Dalvik/ART виде.

Этап 3: Хирургическая замена в APK

Вместо полной замены classes3.dex (что удалило бы половину приложения) была выполнена точная замена только классов HiddenApiBypass:

1. Анализ: В оригинальном classes3.dex (~8 МБ) найдены все классы с пакетом org.lsposed.hiddenapibypass (около 20 упоминаний в структуре и 52 ссылки в коде)
2. Удаление старых классов: Через Dex Editor удалены все устаревшие классы библиотеки
3. Импорт новых: Подготовленные smali-файлы (полученные из версии 6.1) импортированы в то же место
4. Сборка: APK пересобран и подписан

Этап 4: Проверка

После установки модифицированной версии AndroidIDE:

· Приложение успешно запускается без SIGSEGV
· HiddenApiBypass корректно выполняет обход ограничений через обновлённый JNI-код
· Termux и Gradle работают в штатном режиме

Почему это сработало

Компонент Оригинал (падал) Мой мод (работает)
Версия HiddenApiBypass 4.x (старая) 6.1 (актуальная)
Совместимость с Android 15 Нет (устаревшие JNI-вызовы) Да (обновлённый код)
Способ интеграции Встроен в classes3.dex Точечная замена
Производительность Не измерялась из-за краша ~15 мс на инициализацию 

---

4. Что даёт этот мод

1. Стабильность: AndroidIDE больше не падает при запуске на Android 15
2. Совместимость: Работает на новых устройствах с Pixel/чистым Android 15
3. Актуальность: Используется последняя версия HiddenApiBypass с оптимизациями
4. Сохранение функциональности: Весь оригинальный код AndroidIDE (кроме библиотеки) остался нетронутым


5. ENGLISH 🇬🇧
6. 1. What is AndroidIDE?

AndroidIDE is a full-featured Integrated Development Environment (IDE) that runs natively on Android devices—no PC required. Unlike basic code editors, AndroidIDE includes a complete Gradle build system, XML resource handling, and the ability to compile APK files directly on your phone or tablet .

Key Features of the Original Project:

· Complete IDE with syntax highlighting, code completion, and refactoring tools
· Embedded Termux for shell command execution
· Gradle support — build Android projects without a computer
· Open source on GitHub (GPL-3.0 license)

However, AndroidIDE has a critical dependency: the HiddenApiBypass library from LSPosed, which allows the app to bypass Android's restrictions on using hidden/internal APIs (e.g., internal ActivityThread or PackageManager methods).

---

2. Why Did the Original AndroidIDE Crash on Android 15?

Technical Root Cause

Starting with Android 15 (API level 35), Google modified the internal implementation of the mechanism that enforces hidden API restrictions. Older versions of the HiddenApiBypass library (up to version 5.1) used direct JNI calls to ART (Android Runtime) internal structures—structures that were either removed or significantly changed in Android 15 .

When AndroidIDE attempted to execute:

```java
HiddenApiBypass.setHiddenApiExemptions(...);
```

the following occurred:

1. The JNI code tried to access a non-existent memory address (fault addr 0x0000000c)
2. The system terminated the process with signal SIGSEGV (segmentation violation)
3. The crash log showed: Cause: null pointer dereference

Why classes3.dex Specifically?

crash log clearly shows:

```
#02 pc 0000000000602c18 [anon:dalvik-classes3.dex] (org.lsposed.hiddenapibypass.HiddenApiBypass.getDeclaredMethods+180)
```

This indicates the problematic code was located in classes3.dex—the third DEX file in the APK. Android uses multiple DEX files when an app exceeds the 64K method limit per file. classes3.dex contained not only the HiddenApiBypass classes but also other AndroidIDE code (approximately 8 MB total).

---

3. How Fixed It (Technical Mod Description)

My mod solves the problem fundamentally: replacing the outdated library with the latest version 6.1, which is fully compatible with Android 15.

Modification Steps

Step 1: Acquire the New Library

The latest HiddenApiBypass (version 6.1) was downloaded from the official LSPosed GitHub repository. Key improvements in this version:

· Complete rewrite of the JNI layer to work with the new ART structures
· Dynamic Android version detection to select the appropriate bypass method
· Performance optimization (reduced from 212ms in early 6.x versions to 15ms in the final release)

Step 2: Extraction and Conversion

From the hiddenapibypass-6.1.aar (Android Archive) file, classes.jar was extracted and then converted to DEX format. The resulting file is ~14 KB—this contains all necessary library classes compiled for Dalvik/ART runtime.

Step 3: Surgical Replacement in the APK

Instead of replacing the entire classes3.dex (which would have deleted half the application), a precision replacement of only the HiddenApiBypass classes was performed:

1. Analysis: Within the original classes3.dex (~8 MB), all classes with the org.lsposed.hiddenapibypass package were identified (approximately 20 class structures and 52 code references)
2. Removal of old classes: Using Dex Editor, all outdated library classes were deleted
3. Import of new classes: Prepared smali files (from version 6.1) were imported into the same location
4. Rebuild: The APK was reassembled and resigned

Step 4: Verification

After installing the modified AndroidIDE version:

· The application launches successfully without SIGSEGV
· HiddenApiBypass correctly bypasses restrictions using updated JNI code
· Termux and Gradle functionality remain fully operational

Why This Works

Component Original (Crashed) My Mod (Works)
HiddenApiBypass Version 4.x (legacy) 6.1 (latest)
Android 15 Compatibility No (obsolete JNI calls) Yes (updated code)
Integration Method Embedded in classes3.dex Surgical replacement
Performance Not measurable (crashed) ~15ms initialization

## Ссылки / References

- **Оригинальный AndroidIDE**: [itsaky/AndroidIDE](https://github.com/itsaky/AndroidIDE)
- **HiddenApiBypass (LSPosed)**: [LSPosed/AndroidHiddenApiBypass](https://github.com/LSPosed/AndroidHiddenApiBypass)
- **Анализ производительности на Android 15**: [GitCode Blog](https://blog.gitcode.com/b0a23026262b04cecc9cea6bf91ad1d3.html)
- **Обзор библиотеки HiddenApiBypass**: [GitCode Blog](https://blog.gitcode.com/06329af32b433f8b4c6cffb1d40b0794.html)
