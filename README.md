# ИНСТРУКЦИЯ К БИБЛИОТЕКЕ VKassistent

   **VKassistent** — Кастомная библиотека для создания умных устройств
на базе микроконтроллеров с процессором ESP32, управляемых с помощью
чат-бота в социальной сети «ВКонтакте».

   Библиотека позволяет принимать команды с помощью специально
создаваемых умных клавиатур или отдельно высылаемых сообщений,
а также принимать и отправлять вложения: фото, документы, гео-метки.

   Для работы с библиотекой вам надо будет заранее создать сообщество,
внутри которого будет находиться управляемый микроконтроллером бот.
Документация VK для разработки:
*https://dev.vk.com/ru/api/bots/getting-started*

## НАСТРОЙКА И ПОДГОТОВКА ДАННЫХ

> **ПРЕДУПРЕЖДЕНИЕ**
>
> ДЛЯ РАБОТЫ ТРЕБУЕТСЯ СКАЧАТЬ ВСПОМОГАТЕЛЬНУЮ БИБЛИОТЕКУ ArduinoJson.h,
> Она может скачаться автоматически или предложить автоскачивание.

   В начале кода вы обязаны указать данные, нужные для подключения
к вашей сети под типом данных String:
- Название сети WiFi, куда будет подключаться устройство.
- Пароль от подключаемой сети WiFi.
- Токен вашего VK бота.
- ID вашей группы/сообщества, где находится бот.

Учтите, что название именно этих констант не должно отличаться, именно
они используются в файлах библиотеки, пример смотреть ниже.

``` cpp
String SSID     = "__________";
String PASSWORD = "__________";
String Token    = "__________";
String GroupID  = "__________";
```

> **ПРЕДУПРЕЖДЕНИЕ**
>
> Убедитесь что ваша WiFi сеть работает на частоте 2.4 ГГц, ибо не все
> процессоры ESP32 могут работать на частоте 5 ГГц.

   Далее в коде указываются переменные, отвечающие за администраторов,
которые имеют право принимать/видеть и отвечать/управлять ботом,
тип данных для них — long.

``` cpp
long MyID         = __________;
long FriendID     = __________;
long MinosPrimeID = __________;
```

## ПРАВА ТОКЕНА

> **ВАЖНО**
>
> Библиотека использует **два типа токенов**:
>
> 1. **Токен сообщества** — для Long Poll и отправки сообщений.
> 2. **Пользовательский токен** — для загрузки фото на сервер VK.
>
> Метод `photos.getMessagesUploadServer` **не работает** с токеном
> сообщества. Для отправки своих фото (с ESP32-CAM, из FS) нужен
> пользовательский токен.

**Как создать токен сообщества:**

1. Управление → Работа с API → Ключи доступа → Создать ключ.
2. Отметьте галочки:
   - ☑ **Сообщения сообщества** — для `messages.send`.
   - ☑ **Управление сообществом** — для Long Poll.
3. Скопируйте токен в переменную `Token`.

**Как получить пользовательский токен:**

1. Зайдите на https://vkhost.github.io/
2. Выберите «VK Admin» или «Kate Mobile».
3. Нажмите «Разрешить».
4. Скопируйте токен из URL (часть после `access_token=`).
5. Вставьте в переменную `UserToken` (если используете отправку фото).

``` cpp
String UserToken = "__________"; // для загрузки фото (опционально)
```

## РАБОТА С БОТОМ

   В `void setup()` нужно настроить скетч для верной работы бота.
Используем эти команды:
- `VKassistent.connectWIFI(SSID, PASSWORD);` — подключение к вашей сети WiFi.
- `VKassistent.begin();` — запуск VK бота (инициализация LittleFS + Long Poll).
- `VKassistent.addAdmin(__________);` — добавление в админы.

``` cpp
void setup() {
  Serial.begin(115200); // важно сделать именно эту частоту

  VKassistent.connectWIFI(SSID, PASSWORD);  // Подключаем Wi-Fi
  VKassistent.begin();                      // Запускаем бота
  VKassistent.addAdmin(MyID);               // Добавляем админа
  VKassistent.addAdmin(FriendID);
  ...
}
```

   В `void loop()` обязательно должна быть команда `VKassistent.loop();`.
Она опрашивает VK через Long Poll и обрабатывает входящие сообщения.

``` cpp
void loop() {
  VKassistent.loop();
}
```

## СОЗДАНИЕ КЛАВИАТУРЫ

   Для создания умной клавиатуры вызывается функция `createKeyboard`:

``` cpp
String menu = createKeyboard(
  Button("ВКЛЮЧИТЬ", positive),
  Button("ВЫКЛЮЧИТЬ", negative),
  Line(),
  Button("СТАТУС", primary),
  Button("НАСТРОЙКИ", secondary)
);
```

Где `menu` — название объекта клавиатуры, а `Button` — сама кнопка.
Она имеет два параметра: текст кнопки в «кавычках» и цвет кнопки.

VK поддерживает данные цвета кнопок:
- 🔵 **primary** — синий, для важных/основных действий.
- ⚪ **secondary** — белый, для стандартных/обычных действий.
- 🟢 **positive** — зелёный, для подтверждения/одобрения.
- 🔴 **negative** — красный, для удаления/предупреждения.

> **ПРЕДУПРЕЖДЕНИЕ**
>
> Цвета кнопок несут в себе чисто эстетическое значение.

   Функция `Line()` служит для переноса кнопок на новую строку. Если её
не использовать — кнопки автоматически разместятся по 5 в ряд.

   Пример без `Line()` — все кнопки в одном ряду:

``` cpp
String row = createKeyboard(
  Button("1"), Button("2"), Button("3")
);
// → [1][2][3] в одну строку
```

   Пример с `Line()` — перенос:

``` cpp
String menu = createKeyboard(
  Button("ВКЛ", positive),
  Button("ВЫКЛ", negative),
  Line(),
  Button("СТАТУС", primary)
);
// → [ВКЛ][ВЫКЛ]
//   [СТАТУС]
```

## РЕГИСТРАЦИЯ КОМАНД

   Библиотека предоставляет **три способа** регистрации команд.

### 1. Старый API — processMessage

Принимает два параметра: `long userId` и `String text`.

``` cpp
VKassistent.processMessage("ВКЛЮЧИТЬ", [](long UserID, String text) {
  digitalWrite(LED_PIN, HIGH);
  VKassistent.send(UserID, "🔛 Включено");
});
```

### 2. Новый API без параметров — onMessage

Короткий вариант для простых команд.

``` cpp
VKassistent.onMessage("СТАТУС", []() {
  VKassistent.sendToLast("📊 Система работает");
});
```

`sendToLast` — отправляет сообщение в тот же чат, откуда пришла команда.
**Работает только внутри колбэка.**

### 3. Новый API с VKMessage — onMessage

Полный вариант — доступ к `peerId`, `fromId`, `attachments`.

``` cpp
VKassistent.onMessage("ID", [](VKMessage& msg) {
  String info = "Твой peerId: " + String(msg.peerId) +
                "\nТвой fromId: " + String(msg.fromId);
  VKassistent.send(msg.peerId, info);
});
```

> **ВАЖНО**
>
> Текст команды в «кавычках» должен **точно совпадать** с текстом на
> кнопке или сообщением, которое может написать пользователь.
> Регистр важен.

## ПРИЁМ ВЛОЖЕНИЙ

   Библиотека автоматически распознаёт вложения и вызывает
соответствующий обработчик.

### Фото — onPhoto

``` cpp
VKassistent.onPhoto([](VKMessage& msg) {
  if (!msg.hasPhoto()) return;
  String url = msg.getPhotoUrl();
  Serial.println("📸 Фото: " + url);
  VKassistent.send(msg.peerId, "✅ Фото принято");
});
```

### Документ — onDoc

``` cpp
VKassistent.onDoc([](VKMessage& msg) {
  if (!msg.hasDoc()) return;
  String title = msg.getDocTitle();
  String url   = msg.getDocUrl();
  Serial.println("📄 Документ: " + title);
  VKassistent.send(msg.peerId, "✅ Документ " + title + " принят");
});
```

### Гео-метка — onGeo

``` cpp
VKassistent.onGeo([](VKMessage& msg) {
  float lat = msg.getLat();
  float lon = msg.getLon();
  VKassistent.send(msg.peerId, "📍 " + String(lat, 6) + ", " + String(lon, 6));
});
```

### Стикер — onSticker

``` cpp
VKassistent.onSticker([](VKMessage& msg) {
  int id = msg.attachments[0].id;
  VKassistent.send(msg.peerId, "🎨 Стикер id: " + String(id));
});
```

> **ПРЕДУПРЕЖДЕНИЕ**
>
> VK **не позволяет** ботам сообществ отправлять стикеры от своего имени.
> Принимать стикеры — можно, отправлять — нет.

## СОХРАНЕНИЕ ВЛОЖЕНИЙ В ПАМЯТЬ

   Библиотека умеет сохранять входящие фото и документы в файловую
систему LittleFS (внутренняя память ESP32, ~1.4 МБ).

### Сохранить фото из сообщения

``` cpp
VKassistent.onPhoto([](VKMessage& msg) {
  String path = VKassistent.savePhoto(msg);
  if (path != "") {
    VKassistent.send(msg.peerId, "✅ Сохранено: " + path);
  }
});
```

### Сохранить фото и сразу ответить

``` cpp
VKassistent.onPhoto([](VKMessage& msg) {
  VKassistent.savePhotoAndReply(msg);
});
```

### Сохранить документ

``` cpp
VKassistent.onDoc([](VKMessage& msg) {
  VKassistent.saveDocAndReply(msg);
});
```

> **ВАЖНО**
>
> При сохранении в LittleFS **все старые файлы удаляются** перед записью
> нового. Это сделано, чтобы не забивать память.

## ОТПРАВКА ФОТО

   Библиотека умеет отправлять фото **из файловой системы** (LittleFS).
Для этого нужно **два токена**: токен сообщества (для Long Poll
и `messages.send`) и пользовательский (для загрузки фото).

### Отправить фото из LittleFS

``` cpp
VKassistent.onMessage("отправь", [](VKMessage& msg) {
  VKassistent.sendPhotoFromFS(msg.peerId, "📸 Вот фото:", LittleFS, "/photo.jpg");
});
```

> **ПРЕДУПРЕЖДЕНИЕ**
>
> Для отправки фото **обязательно** нужен пользовательский токен.
> Токен сообщества **не подойдёт** — VK вернёт `error_code: 15`.

## ОТПРАВКА СООБЩЕНИЙ

### Простое сообщение

``` cpp
VKassistent.send(msg.peerId, "Hello_world!");
```

### Сообщение с клавиатурой

``` cpp
VKassistent.sendWithKeyboard(msg.peerId, "Выбери действие:", menu);
```

### Гео-метка

``` cpp
VKassistent.sendGeo(msg.peerId, 55.753994, 37.620446);
VKassistent.sendGeo(msg.peerId, "📍 Красная площадь", 55.753994, 37.620446);
```

## ПОЛНЫЙ ПРИМЕР

``` cpp
#include <VKassistent.h>

String SSID     = "__________";
String PASSWORD = "__________";
String Token    = "__________";
String GroupID  = "__________";

VKassistent bot(Token, GroupID);

// Клавиатура главного меню
String mainMenu = createKeyboard(
  Button("ВКЛЮЧИТЬ", positive),
  Button("ВЫКЛЮЧИТЬ", negative),
  Line(),
  Button("СТАТУС", primary),
  Button("МЕНЮ", secondary)
);

void setup() {
  Serial.begin(115200);

  bot.connectWIFI(SSID, PASSWORD);
  bot.begin();

  // Приём фото — сохраняем
  bot.onPhoto([](VKMessage& msg) {
    bot.savePhotoAndReply(msg);
  });

  // Приём документа — сохраняем
  bot.onDoc([](VKMessage& msg) {
    bot.saveDocAndReply(msg);
  });

  // Приём гео — показываем координаты
  bot.onGeo([](VKMessage& msg) {
    bot.send(msg.peerId, "📍 " + String(msg.getLat(), 6) + ", " +
                          String(msg.getLon(), 6));
  });

  // Кнопка "ВКЛЮЧИТЬ" — старый API
  bot.processMessage("ВКЛЮЧИТЬ", [](long UserID, String text) {
    digitalWrite(LED_PIN, HIGH);
    bot.send(UserID, "🔛 Включено");
  });

  // Кнопка "ВЫКЛЮЧИТЬ" — старый API
  bot.processMessage("ВЫКЛЮЧИТЬ", [](long UserID, String text) {
    digitalWrite(LED_PIN, LOW);
    bot.send(UserID, "🔴 Выключено");
  });

  // Кнопка "СТАТУС" — новый API без параметров
  bot.onMessage("СТАТУС", []() {
    bot.sendToLast("📊 Система работает штатно");
  });

  // Кнопка "МЕНЮ" — новый API с VKMessage
  bot.onMessage("МЕНЮ", [](VKMessage& msg) {
    bot.sendWithKeyboard(msg.peerId, "Главное меню:", mainMenu);
  });

  Serial.println("✅ Бот запущен");
}

void loop() {
  bot.loop();
}
```

## ЧЕК ЛИСТ

В случае, если код не хочет работать, убедитесь в наличии этих пунктов:

1) Убедитесь, что в коде и настройках `Serial.begin()` значение стоит 115200.
2) Название и пароль сети WiFi соответствуют реальным.
3) Текст кнопки соответствует ожидаемому тексту в команде и наоборот.
4) Плата подключена к компьютеру.
5) Убедитесь, что вы подключаетесь к сети 2.4 ГГц.
6) Убедитесь, что токен (Token) скопирован полностью и без лишних символов.
7) Проверьте, что бот включён в настройках сообщества:
   Управление → Сообщения → Сообщения сообщества → Включены.
8) Проверьте, что Long Poll API включён:
   Управление → Работа с API → Long Poll API → Включено.
9) В разделе Long Poll API → Типы событий отметьте:
   - ☑ Входящие сообщения
10) Убедитесь, что вы подписаны на сообщество, где находится VK бот.
11) При использовании внешнего питания — проверьте напряжение (5В).
12) Убедитесь, что клавиатура создана через `createKeyboard()`.
13) Проверьте, что в `void loop()` есть команда `VKassistent.loop();`.
14) Если используете отправку фото — убедитесь, что у вас есть
    **пользовательский токен** (не токен сообщества).

Иногда может помочь перезагрузка кнопкой RESET (RST) на плате.

## ЧАСТЫЕ ОШИБКИ

### Бот молчит, ничего не приходит

- Проверьте права токена (пункт 6–8 чек-листа).
- Проверьте, что бот **включён** в настройках сообщества.
- Проверьте, что вы **подписаны** на сообщество.

### error_code: 15 — Access denied

- Для `photos.getMessagesUploadServer` нужен **пользовательский токен**.
- Токен сообщества **не подходит** для загрузки фото.

### Фото не сохраняется

- Проверьте `LittleFS.begin()` в начале — должно быть `✅ LittleFS OK`.
- Проверьте свободное место: `LittleFS.totalBytes() - LittleFS.usedBytes()`.
- Размер фото не должен превышать ~100 КБ.

### sendToLast не работает

- `sendToLast` работает **только внутри колбэка** (`onMessage`, `onPhoto` и т.д.).
- Если вызывать вне — будет `⚠️ sendToLast: вне колбэка`.

### Стикеры не отправляются

- VK **не позволяет** ботам сообществ отправлять стикеры.
- Это ограничение VK, не баг библиотеки.

---

Автор библиотеки — Arctic_WAWES.
Поддержка — @arctic_wawes.
