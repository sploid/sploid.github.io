<p align="right" width="100%"><a href="https://sploid.github.io/">В начало</a></p>
<p align="right" width="100%"><a href="https://sploid.github.io/logs/">Английская версия этой статьи</a></p>

![заголовок](https://sploid.github.io/imgs/logs_header.png)

# Отправляем Qt-логи с метками в Sentry

Привет всем.

Я давно занимаюсь разработкой десктопных приложений с использованием Qt и уже наработал некоторые подходы по работе с логами. В данной статье я бы хотел поделиться с сообществом этими подходами, они могут пригодиться в разных ситуациях.

Все действия проводились в релизной сборке в Visual Studio 2022 под Windows и clang под macOS, но, по большей части, это справедливо и для других компиляторов.

## Имя исходника в логах

Не будем повторять часть про установку своего обработчика записи в логи через `qInstallMessageHandler` и коснемся более интересных вещей. Итак, для начала нам нужно получить имя файла с исходником и номер строки, откуда осуществлялась запись в лог. Мы установили свой обработчик записи и начинаем писать в лог:

```cpp
void MessageOutput(QtMsgType type, const QMessageLogContext& context, const QString& msg) {
  if (!context.file) {
    std::cout << "no file" << std::endl;
  }
}

int main(int, char*[]) {
  const QtMessageHandler prev_handler{qInstallMessageHandler(MessageOutput)};
  qInfo() << "example";
  qInstallMessageHandler(prev_handler);
}
```

[ссылка на пример](https://github.com/sploid/samples/blob/main/logs/one.cc)

в релизной сборке в выводе мы видим:

`no file`

Получается, что имена не передаются в релизе в Qt. В С++ есть макрос для имени файла и строки, подставляем его и проверяем что будет под Windows:

```cpp
#define myInfo QMessageLogger(__FILE__, __LINE__, nullptr).info

void MessageOutput(QtMsgType type, const QMessageLogContext& context, const QString& msg) {
  if (!context.file) {
    std::cout << "no file" << std::endl;
  } else {
    std::cout << context.file << std::endl;
  }
}

int main(int argc, char* argv[]) {
  const QtMessageHandler prev_handler{qInstallMessageHandler(MessageOutput)};
  myInfo() << "example";
  qInstallMessageHandler(prev_handler);
}
```

[ссылка на пример](https://github.com/sploid/samples/blob/main/logs/two.cc)

и получаем:

`C:\soft\samples\logs\two.cc`

Следующий этап - отрезать путь к файлу во время компиляции, чтобы выводилось только имя файла:

```cpp
template <typename T, size_t S>
inline constexpr size_t get_file_name_offset(const T (& str)[S], size_t i = S - 1) {
  return (str[i] == '/' || str[i] == '\\') ? i + 1 : (i > 0 ? get_file_name_offset(str, i - 1) : 0);
}

template <typename T>
inline constexpr size_t get_file_name_offset(T (& str)[1]) {
  (void)str;
  return 0;
}

namespace utility {
  template <typename T, T v>
  struct const_expr_value {
    static constexpr const T value = v;
  };
}

#define UTILITY_CONST_EXPR_VALUE(exp) ::utility::const_expr_value<decltype(exp), exp>::value

#define myInfo QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, nullptr).info

void MessageOutput(QtMsgType type, const QMessageLogContext& context, const QString& msg) {
  if (!context.file) {
    std::cout << "no file" << std::endl;
  } else {
    std::cout << context.file << std::endl;
  }
}

int main(int argc, char* argv[]) {
  const QtMessageHandler prev_handler{qInstallMessageHandler(MessageOutput)};
  myInfo() << "example";
  qInstallMessageHandler(prev_handler);
}
```

[ссылка на пример](https://github.com/sploid/samples/blob/main/logs/three.cc)

и получаем:

`three.cc`

## Передача уникальной метки

Иметь имя файла + номер строки в логах очень удобно для нахождения места ошибки, но эта информация не подойдет для уникальной метки, так как номер строки меняется в процессе изменения исходного кода приложения. Уникальная метка нам нужна, чтобы мы могли агрегировать логи по ней и понимать, сколько и каких событий произошло. Метки дают возможность агрегации одинаковых логов для разных версий приложений.
Для передачи метки мы используем “категорию” в терминологии Qt. Тут есть проблема: все “метки” должны быть уникальными. При сборке приложения у нас запускаются скрипты, которые проверяют, что все метки уникальны, и, если нет, то сборка приостанавливается.

Для передачи метки через категорию мы используем такой код:

```cpp
#define myInfoC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).info()

void MessageOutput(QtMsgType type, const QMessageLogContext& context, const QString& msg) {
  const QString file{context.file ? u"%1:%2"_s.arg(context.file).arg(context.line) : u"no file"_s};
  std::cout << file.toStdString() << " (" << context.category << ")" << std::endl;
}

int main(int argc, char* argv[]) {
  const QtMessageHandler prev_handler{qInstallMessageHandler(MessageOutput)};
  myInfoC("category") << "example";
  qInstallMessageHandler(prev_handler);
}
```

[ссылка на пример](https://github.com/sploid/samples/blob/main/logs/four.cc)

и видим что и нужно:

`four.cc:37 (category)`

## Info/Warning/Error

Еще одним важным параметром вывода в лог является его тип. В своих приложениях, на сервер мы отправляем только предупреждения и ошибки. Информационные сообщения остаются только в логе на ПК пользователя. Лучше не пропускать в релиз отладочные предложения.

```cpp
#define myInfoC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).info()
#define myWarningC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).warning()
#define myCriticalC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).critical()
```

[ссылка на пример](https://github.com/sploid/samples/blob/main/logs/five.cc)

и в выводе видим:

```txt
Debug no file (default) Message
Info five.cc:56 (cat_info) Message
Warning five.cc:57 (cat_warning) Message
Critical five.cc:58 (cat_critical) Message
```

## Отправка в sentry

Итак, у нас есть следующая информация при выводе в лог:

- тип сообщения,
- текст сообщения,
- метка,
- имя файла,
- номер строки.

и мы готовы отправлять всю эту информацию в sentry. Не буду тут описывать непосредственно отправку данных, так как в сети достаточно много информации по реализации сетевого взаимодействия на Qt, поэтому я опишу только отправляемый JSON. Мы отправляем значительно больше информации, но примеры касаются данной статьи. Хочу заметить, что мы не используем SDK sentry, а делаем всю отправку через их API средствами QtNetwork.

[Документацию по API для sentry можно найти тут](https://develop.sentry.dev/sdk/store/)

Для примера отправим три сообщения с предупреждением (warning):

```json
{
    "event_id": "00000000000000000000000000000001",
    "message": "example",
    "level": "warning",
    "transaction": "main.cc:10",
    "fingerprint": ["one"]
}
```

И три сообщения об ошибке (error):

```json
{
    "event_id": "00000000000000000000000000000004",
    "message": "example",
    "level": "error",
    "transaction": "main.cc:10",
    "fingerprint": ["two"]
}
```

события в sentry сгруппировались по метке:

![grouped by tag](https://sploid.github.io/imgs/logs_img_1.png)

Если зайти в детали сообщения, то видим переданные данные:

![details of the message](https://sploid.github.io/imgs/logs_img_2.png)

- 1 - текст сообщение
- 2 - уровень сообщения (warning/error)
- 3 - файл и строка
- 4 - наша метка

## Ссылка на место отправки сообщения

В качестве дополнительной информации мы отправляем ссылку на место, откуда было отправлено сообщение. Для этого у нас все релизы имеют свою версию, которая помечается тегом в git. В sentry эта информация отправляется в качестве дополнительной информации, и JSON выглядит так:

```json
{
    "event_id": "00000000000000000000000000000007",
    "message": "example",
    "level": "warning",
    "transaction": "main.cc:10",
    "extra": {
        "source": "https://github.com/sploid/samples/blob/v0.1/README.md?plain=1#L5"
    },
    "fingerprint": ["three"]
}
```

в sentry мы можем быстро перейти на строку, откуда была сделана отправка сообщения:

![детали сообщения](https://sploid.github.io/imgs/logs_img_3.png)

## Не делайте flush логов без крайней необходимости

В какой-то момент времени приложение начало аварийно завершаться и, чтобы быть уверенным, что мы видим все записанные логи, мы вставили вызов flush в файл после каждой записи в лог. После этого приложение начало жутко тормозить именно на записи в лог, поэтому мы пришли к выводу, что не стоит делать flush без крайней необходимости, даже если очень хочется.

## Преобразование времени в строку может ощутимо замедлить работу приложения

В разрабатываемом приложении мы активно писали логи, и при записи логов выводилось время события с точностью до секунды. В некоторые моменты времени писалось до 100 событий в секунду.

Время события формировалось примерно таким кодом:

```cpp
std::string GetEventTime() {
  const std::time_t now{std::time(NULL)};
  const std::tm* const ptm{std::localtime(&now)};
  char buffer[32];
  std::strftime(buffer, 32, "%Y-%m-%d %H:%M:%S", ptm);
  return std::string{buffer};
}
```

После исследования профайлером приложения, выяснилось что преобразование времени в текст происходит долго и код был изменен на такой:

```cpp
time_t gPrevTime{};
std::string gPrevTimeStr;

const std::string& GetEventTime() {
  const std::time_t now{std::time(NULL)};
  if (now == gPrevTime) {
    return gPrevTimeStr;
  }
  const std::tm* const ptm{std::localtime(&now)};
  char buffer[32];
  std::strftime(buffer, 32, "%Y-%m-%d %H:%M:%S", ptm);
  gPrevTimeStr = std::string(buffer);
  gPrevTime = now;
  return gPrevTimeStr;
}
```

Несомненно, нельзя забывать о многопоточности и нужно защищать данные, я намерено пренебрег этими аспектами в данном примере, чтобы акцентировать внимание читателей именно на теме статьи.
