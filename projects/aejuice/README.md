
# AEJuice Pack Manager 4

Период работы: июль 2020 - настоящее время

В данной статье я бы хотел рассказать о продукте AEJuice Pack Manager 4. Этот продукт включает в себя одно отдельное приложение и плагины для Adobe After Effect и Adobe Premiere Pro. Внутренности приложения и плагинов одинаковые, различается только как главное окно отображается пользователю. Данное приложение является витриной продуктов компании AEJuice и предоставляет возможность удобной работы с ними.

# Задачи

- Приложение было запущено с нуля. Была доступна предыдущая версия приложения, но ее код нельзя было использовать из-за его качества. Из предыдущей версии приложения был частично позаимствован дизайн.
- Настроено CI/CD. В первой версии сборка была настроена на базе Jenkins, затем перенесена на Github Actions. Сборка реализована скриптами на языке lua.
- Реализованы инсталляторы под Windows и macOS.
- Занимался решением проблем от пользователей, выступая второй линией поддержки. Также, занимался обучением специалистов поддержки первой линии.
- Настроил мониторинг работоспособности приложения на базе Graphite + Grafana.
- Занимался поиском сотрудников, проведением собеседований и онбордингом.
- Интеграция с Adobe Premiere Pro была реализована посредствам dll-injection, так как данный инструмент не позволяет писать плагина на С++.
- Управлял командой до двух человек. Оформлял задачи от руководства, распределял их в команде и контролировал их выполнение.
- Реализована защита приложения средствами VMProtect.

# Ссылки

- [AEJuice](https://aejuice.com/)
- [YouTube](https://youtu.be/cfwZCq504kY?si=X6Y0Vph3_Jn4yABa)
- [AEJuice Pack Manager 4 for macOS](https://aejuice.com/pack_manager/AEJuice_Pack_Manager_mac.zip)
- [AEJuice Pack Manager 4 for Windows](https://aejuice.com/pack_manager/AEJuice_Pack_Manager.zip)

# Стек

С++, Qt, STL, Visual Studio 2019, Visual Studio Code, Clang, macOS, Windows, Adobe After Effect, Adobe Premiere Pro, Github, Git, Jenkins, InnoSetup, Packages (for macOS), Graphite, Grafana, lua, pixel-perfect components, Figma, network programming, multithreaded programming.

# Скриншоты

![Инсталлятор Windows](https://sploid.github.io/imgs/projects/aejuice_4.png)
![Инсталлятор macOS](https://sploid.github.io/imgs/projects/aejuice_5.png)
![Список паков](https://sploid.github.io/imgs/projects/aejuice_1.png)
![Интеграция в Adobe Premiere Pro](https://sploid.github.io/imgs/projects/aejuice_2.png)
![Настройки](https://sploid.github.io/imgs/projects/aejuice_3.png)
