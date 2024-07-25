<p align="right" width="100%"><a href="https://sploid.github.io/">В начало</a></p>
<p align="right" width="100%"><a href="https://sploid.github.io/desktop_services/">Английская версия этой статьи</a></p>

<p align="center" width="100%">Несколько важных моментов при создании десктопного приложения</p>

Привет всем.

Меня зовут Иван и в настоящий момент я работаю руководителем группы разработки кросс-платформенных десктопных приложений. Я занимаюсь разработкой десктопных приложений более 20 лет и имею большой опыт как запуска новых приложений, так и поддержки уже запущенных ранее. Одно из разработанных моей командой приложений имеет дневную аудиторию более 100к уникальных пользователей. As a freelancer for several years, I see that many customers want to develop an application, but they miss that not everything is limited to the app itself.
По статистике, большинство пользователей не обращаются в поддержку при возникновении проблем с приложением, а просто отказываются от его использования, поэтому очень важно исправлять даже те проблемы, которые еще не дошли до поддержки. Как можно избежать большинство проблем при сопровождении приложения? В этой статье я расскажу о некоторых инструментах, которые могут быть полезны при развертывании и поддержки десктопных приложений с большой аудиторией пользователей.

# Инсталлятор

Инсталлятор очень важен при установке современного программного обеспечения и существует множество фреймворков для его создания. Распространение приложение в виде архива выглядет непрофисионально и может оттолкнуть пользователей. В инсталяторе очень важна подпись. Подписан должен быть как сам инсталятор, так и исполняемые файлы, которые в него входят. Антивирусы (включая встроенный в  Windows) могут блокировать установку не подписанных файлов. Это может оттолкнуть некоторых пользователей от установки и они могут подумать что приложение не безопасно.

# Версионирование

Очень важно точно знать из какого исходного кода собрано приложение. Существует множество различных форматов версии приложения, но главное это точно знать из каких исходников оно скомпилированно. Исходники важны для разбора проблем при получении жалобы о работе приложения. Команда поддержки может проверить установлена ли у пользователя последняя версия приложения или нет, и, если не последняя, то в начале они попросят обновиться до последней и проверить, проявляется ли данная проблема в этой версии. В моих приложениях я использую формат версии "год.месяц.номер сборки". Этот формат достаточно хорошо себя зарекомендовал, потому что он подходит для различных платформ и легко можно проверить дату выхода версии. Например, версия, которая создана в июне 2024 года буедт выглядеть "24.06.0101".

# Сборка и выкатка (CI/CD)

When developing a cross-platform application, it is quite normal for some code to be compiled on one platform but not on another. It is necessary to rebuild the application in all possible configurations during the commit in order to fix the assembly while the memories are still fresh. It often happens that due to the complex deployment of the application, the number of releases drops, and features with fixes lie unused for a long time. Integration and deployment of the application is very important for the convenience of maintenance.

# Updating the application

It is not enough to implement some features; you still need to roll it to the user. Automatic application update is the best decision here, but it can be difficult to implement in practice. The sticking points here are the need to restart the PC after the update and file access rights. In any case, implementing some minimal functionality to notify the user that a new version is available and the user can download it is required. Without notification of a new version, users will never know that you have made some cool feature or fixed an annoying bug. Frequent (but not too frequent) updates show that you are actively working on the project and increase user loyalty.

# Monitoring

The main task of monitoring is to diagnose the problem as early as possible and begin to solve it. According to statistics, not many people contact support when a problem occurs, so if the notification of problems is only a request to support, then you won't even know about most problems without monitoring. It is also very useful to have properly configured alerts in monitoring, so there will be no need to constantly monitor the charts. Monitoring can allow you to start solving the problem even if the user does not know about it yet. For example, if some servers are unavailable, then the user can only decrease the network speed and he will not pay attention to it, but you will already find out about it.

# Sending reports about problems

When a user contacts support with a problem, you need to collect the maximum possible information about the environment and the application itself. If the user contacts by e-mail, this will not be possible, so it is better that he writes from the application itself. At least work logs should be added to the problem report. You can also attach screenshots of the screen or a video of the problem. Crash messages can be sent automatically with a crash dump attached.

So, your team has created the app. Congratulations! What else do you need to do:

- Create a signed installer
- Make a unique version for every release
- Implement updating of the app
- Implement a monitoring system
- Develop a process for sending reports about problems.

Now your application is really ready for users, and you can breathe deeply and.. start a new app.
