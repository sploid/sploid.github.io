<p align="right" width="100%"><a href="https://sploid.github.io/">To the begining</a></p>
<p align="right" width="100%"><a href="https://sploid.github.io/ru/logs/">Russian version of this article</a></p>

<p align="center" width="100%">Sending Qt logs with tags to sentry</p>

Hi everybody.

I have been developing desktop applications using Qt for a long time and have already found some approaches to work with logs. Since these approaches can be useful in different situations, I would like to share them with the community.
I carried out actions in Visual Studio 2022 release for Windows and clang for macOS, but, for the most part, this is true for other compilers.

# Source name for logs

Let us not repeat the part about installing the logging handler via `qInstallMessageHandler` and get down to the point. Firstly, we need to get the name of the source file and the line number from where the log was recorded. We have installed record handler and are starting to write to the log:

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

[link to the example](https://github.com/sploid/samples/blob/main/logs/one.cc)

And here is the output:

`no file`

The names are not passed to Qt. C++ has a script for the file name and string, let's try it and check the result:

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

[link to the example](https://github.com/sploid/samples/blob/main/logs/two.cc)

and get:

`C:\soft\samples\logs\two.cc`

The next step is to cut off the path to the file during compilation so that only the file name will output:

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

[link to the example](https://github.com/sploid/samples/blob/main/logs/three.cc)

and get:

`three.cc`

# Passing a unique tag

The file name + line number in the logs are very convenient to locate errors, but this information is a bad match for a unique tag, since the line number transforms when changing the source code of the application. We need a unique tag to aggregate logs based on it and understand how many and what events occurred. Tags make it possible to aggregate the same logs for different versions of applications.
We use a “category” in Qt terminology to convey the tags. The issue is that all “tags” must be unique. When building the application, scripts check that all labels are unique, and if not, then the build is suspended.
To pass a tag through a category, we use the following code:

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

[link to the example](https://github.com/sploid/samples/blob/main/logs/four.cc)

And that's what we need:

`four.cc:37 (category)`

# Info/Warning/Error

Another important parameter of log output is its type. We send only warnings and errors to the server. Information messages remain only in the log on the user's PC. It is better not to skip debugging suggestions in the release.

```cpp
#define myInfoC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).info()
#define myWarningC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).warning()
#define myCriticalC(err_fingerprint) QMessageLogger(&__FILE__[UTILITY_CONST_EXPR_VALUE(get_file_name_offset(__FILE__))], __LINE__, "", err_fingerprint).critical()
```

[link to the example](https://github.com/sploid/samples/blob/main/logs/five.cc)

and we see:

```txt
Debug no file (default) Message
Info five.cc:56 (cat_info) Message
Warning five.cc:57 (cat_warning) Message
Critical five.cc:58 (cat_critical) Message
```

# Sending to sentry

So, we have the following information in the output to the log:

- message type,
- the message itself,
- tag,
- file name,
- line number.

Now we are ready to send all this information to sentry. We will not describe the sending of data directly, you can find a lot of information on network interaction in Qt, so I will describe only the JSON being sent. Of course, we send much more information, but here we keep examples related to the topic. Please note that we do not use the SDK sentry, but do all the sending via QtNetwork API.

[You can find API documentation for sentry here](https://develop.sentry.dev/sdk/store/)

For example, we will send three warning messages:

```json
{
    "event_id": "00000000000000000000000000000001",
    "message": "example",
    "level": "warning",
    "transaction": "main.cc:10",
    "fingerprint": ["one"]
}
```

And three error messages:

```json
{
    "event_id": "00000000000000000000000000000004",
    "message": "example",
    "level": "error",
    "transaction": "main.cc:10",
    "fingerprint": ["two"]
}
```

Events in the sentry are grouped by tag:
![grouped by tag](https://sploid.github.io/imgs/logs_img_1.png)

We will see the transmitted data in the details of the message:
![details of the message](https://sploid.github.io/imgs/logs_img_2.png)

- 1 - text message
- 2 - message level (warning/error)
- 3 - file and string
- 4 - tag

# Link to the place where the message was sent

As additional information, we sent a link to the place where the message was sent. To do this, all releases have their own version, which is tagged in git. This information is sent as an additional in sentry, and the JSON looks like this:

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

In sentry, we can quickly find the line where the message was sent from:
![details of the message](https://sploid.github.io/imgs/logs_img_3.png)
the place where the message was sent.

# Make flush logs only if necessary

At some point in time, the application started to crash, and, to be sure that we saw all the recorded logs, we inserted a flush call into the file after each log entry. Then the application began to slow down terribly precisely on logging, so we came to the conclusion that we should not flush unless absolutely necessary.

# Converting time to a string can significantly slow down the application

We actively wrote logs when developing applications, and the time was displayed with up-to-the-second accuracy. At some points in time, up to 100 events per second were written.
The time was formed like this:

```cpp
std::string GetEventTime() {
  const std::time_t now{std::time(NULL)};
  const std::tm* const ptm{std::localtime(&now)};
  char buffer[32];
  std::strftime(buffer, 32, "%Y-%m-%d %H:%M:%S", ptm);
  return std::string{buffer};
}
```

Analyzing by the application profiler shows that the conversion of time to text takes a long time, and we changed the code:

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

We intentionally neglected multithreading and data protection in order to focus attention on the core point.
