---
title: Отладка - Начало Работы
layout: docs.hbs
---

# <!--debugging-guide-->Руководство по отладке

Это руководство поможет вам начать отладку ваших приложений и скриптов Node.js.

## <!--enable-inspector-->Активация инспектора

При запуске с аргументом `--inspect` процесс Node.js прослушивает клиент отладки.
По умолчанию клиент прослушивается на хосте 127.0.0.1 с портом 9229.
Каждому процессу также назначается уникальный [UUID][].

Клиенты инспектора должны знать и указывать адрес хоста, порт и UUID для подключения.
Полный URL будет выглядеть примерно так:
`ws://127.0.0.1:9229/0f2c936f-b1cd-4ac9-aab3-f63b0f33d55e`.

Процесс Node.js также начнет прослушивать сообщения отладки, если он получит
сигнал `SIGUSR1`. (`SIGUSR1` не доступен в среде Windows.) В Node.js версии 7
и ниже это активирует устаревший Debugger API. В Node.js версии 8 и выше будет
активирован Inspector API.

---

## <!--security-implications-->Последствия для безопасности

Так как дебаггер имеет полный доступ к среде выполнения Node.js, злоумышленник,
способный подключиться к этому порту, сможет выполнить произвольный код от имени
процесса Node. Поэтому важно понимать последствия для безопасности при обличении
порта отладчика в публичных и частных сетях.

### <!--exposing-the-debug-port-publicly-is-unsafe-->Публичное обличение порта отладки небезопасно

Если отладчик привязан к какому-либо публичному IP-адресу, или к 0.0.0.0, любой клиент,
способный достичь вашего IP-адреса сможет подключиться к отладчику без каких-либо
ограничений и сможет запускать произвольный код.

По умолчанию `node --inspect` привязывается к 127.0.0.1. Чтобы разрешить внешние подключения,
вы должны явно предоставить общедоступный IP-адрес или 0.0.0.0 и т.д. Однако это может
подвергнуть приложение потенциально значительной угрозе его безопасности. Мы предлагаем
вам обеспечить наличие файрволов и других соответствующих средств контроля доступа для
того, чтобы предотвратить такую угрозу.

См. раздел '[Включение сценариев удаленной отладки](#enabling-remote-debugging-scenarios)', который
включает рекомендации о том, как безопасно подключить удаленные клиенты отладчика.

### <!--local-applications-have-full-access-to-the-inspector-->Локальные приложения имеют полный доступ к инспектору

Даже если вы привязали порт инспектора к 127.0.0.1 (по умолчанию), любые приложения,
запущенные локально на вашем компьютере, будут иметь неограниченный доступ.
Это сделано для того, чтобы локальные отладчики могли легко подключаться.

### <!--browsers-webSockets-same-origin-policy-->Браузеры, WebSockets, same-origin policy

Веб-сайты, открытые в веб-браузере, могут отправлять запросы WebSocket и HTTP
в соответствии с моделью безопасности браузера. Начальное HTTP-соединение необходимо
для получения уникального идентификатора сеанса отладчика. Правило ограничения домена
(Same Origin Policy) не позволяет веб-сайтам устанавливать это HTTP-соединение.
Для дополнительной защиты от [атак DNS rebinding](https://ru.wikipedia.org/wiki/DNS_rebinding)
Node.js проверяет, что заголовки 'Host' для соединения точно указывают IP-адрес,
localhost или localhost6.

Эти политики безопасности запрещают подключение к удаленному серверу отладки c
указанием имени хоста. Вы можете обойти это ограничение, указав либо IP-адрес,
либо используя ssh-туннели, как описано ниже.

## <!--inspector-clients-->Клиенты инспектора

Несколько коммерческих и открытых инструментов могут подключаться к инспектору Node.js.
Основная информация по ним:

### [node-inspect](https://github.com/nodejs/node-inspect)

- Отладчик CLI, поддерживаемый Фондом Node.js, который использует [Протокол Инспектора][].
- Соответствующая версия собирается вместе с Node.js,
  можно использовать с помощью команды `node inspect myscript.js`.
- Последняя версия также может быть установлена независимо (например, `npm install -g node-inspect`)
  и использоваться через `node-inspect myscript.js`.

### [Инструменты разработчика Chrome](https://github.com/ChromeDevTools/devtools-frontend) 55+, [Microsoft Edge](https://www.microsoftedgeinsider.com)

- **Вариант 1**: Откройте `chrome://inspect` в браузере на основе Chromium
  или `edge://inspect` в браузере Edge. Нажмите кнопку Configure и убедитесь,
  что нужные вам хост и порт перечислены в списке.
- **Вариант 2**: Скопируйте значение `devtoolsFrontendUrl` из вывода `/json/list`
  (`curl http://localhost:9229/json/list`) или текст подсказки --inspect и откройте его в Chrome.

### [Visual Studio Code](https://github.com/microsoft/vscode) 1.10+

- На панели "Отладка" (Debug) щелкните значок настроек, чтобы открыть файл `.vscode/launch.json`.
  Выберите "Node.js" для первоначальной настройки.

### [Visual Studio](https://github.com/Microsoft/nodejstools) 2017+

- В меню выберите "Debug > Start Debugging" или нажмите `F5`.
- [Детальные инструкции](https://github.com/Microsoft/nodejstools/wiki/Debugging).

### [JetBrains WebStorm](https://www.jetbrains.com/webstorm/) 2017.1+ и другие IDE JetBrains

- Создайте новую конфигурацию отладки Node.js и нажмите кнопку "Debug" (`Shift+F9`). `--inspect` будет
  использоваться по умолчанию для Node.js 7+. Чтобы отключить, снимите флажок
  `js.debugger.node.use.inspect` в реестре IDE.

### [chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface)

- Библиотека для облегчения подключения к эндпоинтам Протокола Инспектора.

### [Gitpod](https://www.gitpod.io)

- Запустите конфигурацию отладки Node.js из представления `Debug` или нажмите `F5`. [Детальные инструкции](https://medium.com/gitpod/debugging-node-js-applications-in-theia-76c94c76f0a1)

### [Eclipse IDE](https://eclipse.org/eclipseide) c расширением Eclipse Wild Web Developer

- Открыв файл .js, выберите "Debug As... > Node program", или
- Создайте конфигурацию отладки, чтобы присоединить отладчик к запущенному приложению Node (уже запущенному с `--inspect`).

---

## <!--command-line-options-->Аргументы командной строки

В следующей таблице перечислено влияние различных runtime флагов при отладке:

<table class="table-no-border-no-padding">
  <tr><th>Флаг</th><th>Значение</th></tr>
  <tr>
    <td>--inspect</td>
    <td>
      <ul>
        <li>Включить инспектор</li>
        <li>Прослушивать адрес и порт по умолчанию (127.0.0.1:9229)</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>--inspect=<em>[host:port]</em></td>
    <td>
      <ul>
        <li>Включить инспектор</li>
        <li>Прослушивать адрес <em>host</em> (по умолчанию: 127.0.0.1)</li>
        <li>Прослушивать порт <em>port</em> (по умолчанию: 9229)</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>--inspect-brk</td>
    <td>
      <ul>
        <li>Включить инспектор</li>
        <li>Прослушивать адрес и порт по умолчанию (127.0.0.1:9229)</li>
        <li>Прервать выполнение сценария перед началом выполнения пользовательского кода</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>--inspect-brk=<em>[host:port]</em></td>
    <td>
      <ul>
        <li>Включить инспектор</li>
        <li>Прослушивать адрес <em>host</em> (по умолчанию: 127.0.0.1)</li>
        <li>Прослушивать порт <em>port</em> (по умолчанию: 9229)</li>
        <li>Прервать выполнение сценария перед началом выполнения пользовательского кода</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td><code>node inspect <em>script.js</em></code></td>
    <td>
      <ul>
        <li>Запустить дочерний процесс для выполнения пользовательского скрипта под флагом --inspect;
            использовать основной процесс для запуска отладчика CLI.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td><code>node inspect --port=xxxx <em>script.js</em></code></td>
    <td>
      <ul>
        <li>Запустить дочерний процесс для выполнения пользовательского скрипта под флагом --inspect;
            использовать основной процесс для запуска отладчика CLI.</li>
        <li>Прослушивать порт <em>port</em> (по умолчанию: 9229)</li>
      </ul>
    </td>
  </tr>
</table>

---

## <!--enabling-remote-debugging-scenarios-->Включение сценариев удаленной отладки

Мы рекомендуем, чтобы отладчик никогда не прослушивал общедоступный IP-адрес.
Если вам необходимо разрешить удаленные подключения для отладки, мы рекомендуем
использовать SSH-тунелли. Следующий пример предоставляется только в целях
иллюстрации возможностей. Вы должны понимать все риски информационной безопасности,
связанные с предоставлением удаленного доступа к привилегированной службе.

Допустим вы запускаете на удаленной машине, remote.example.com, приложение Node,
которое вы хотите отлаживать. На этой машине следует запустить процесс Node
с инспектором, прослушивающим только localhost (по умолчанию).

```bash
node --inspect server.js
```

Теперь вы можете настроить ssh-туннель на локальном компьютере, с которого
вы хотите инициировать подключение клиента отладки.

```bash
ssh -L 9221:localhost:9229 user@remote.example.com
```

Это запустит сессию ssh, в которой соединение с портом 9221 на вашем локальном
компьютере будет перенаправлено к порту 9229 на remote.example.com. Теперь вы
можете подключить к localhost:9221 отладчик, такой как Chrome DevTools или
Visual Studio Code, у которого будет возможность отладки так, как если бы приложение
Node.js работало локально.

---

## Устаревший Debugger

**Debugger API устарело начиная с Node.js версии 7.7.0.
Вместо него следует использовать Inspector API с флагом --inspect.**

При запуске с флагом **--debug** или **--debug-brk** в версии 7 или ниже,
Node.js прослушивает команды отладки, определенные протоколом
отладки V8, на порту TCP (по умолчанию `5858`). Любой клиент отладки, который
понимает этот протокол, может подключиться и отладить работающий процесс;
пара популярных клиентов перечислены ниже.

Протокол отладки V8 более не поддерживается и не документируется.

### [Встроенный отладчик](https://nodejs.org/dist/latest/docs/api/debugger.html)

Введите `node debug script_name.js` для запуска скрипта со встроенным CLI отладчиком.
Сам скрипт будет запущен с флагом `--debug-brk` в другом процессе Node, а первоначальный
процесс Node запускает скрипт `_debugger.js` и подключается к целевому скрипту.

### [node-inspector](https://github.com/node-inspector/node-inspector)

Отлаживайте приложение Node.js с помощью Chrome DevTools используя
промежуточный процесс, который переводит протокол инспектора, используемый в Chromium,
в протокол отладчика V8, используемый в Node.js.

<!-- refs -->

[Протокол Инспектора]: https://chromedevtools.github.io/debugger-protocol-viewer/v8/
[UUID]: https://tools.ietf.org/html/rfc4122
