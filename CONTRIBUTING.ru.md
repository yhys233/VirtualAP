# Участие в разработке VirtualAP

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

Функция, которой не существует, лучше сломанной. VirtualAP работает от root на
телефонах с root-правами, на Wi-Fi чипах, драйверы которых расходятся в базовых вещах, поэтому патч, который
работает на вашем телефоне и ломается на чужом, это регрессия, а не вклад. И
десять строк, которые делают работу, лучше сотни, которые делают ту же работу: удаляйте раньше, чем добавляете, переиспользуйте
раньше, чем пишете.

## Что должно соблюдаться

Правила для всех, кто меняет этот репозиторий, включая агентов, находятся в
[AGENTS.ru.md](./AGENTS.ru.md): стиль, формат коммитов, узкие места и жёсткие
ограничения. Коротко: только Android, движок это однопоточный C на musl, который говорит с
ядром сам, одна забота на модуль, приложение работает на Android 8, а бинарники
приходят только из `scripts/build-static.sh` и, для `virtualap`, из корневого `Makefile`.

VirtualAP работает на чипах Qualcomm, Broadcom FullMAC (Samsung) и MediaTek. Исправление особенности одного
чипа принимается только тогда, когда backend адаптируется к ней во время выполнения, через определение, условный
путь или откат, не меняя поведения на остальных;
`nl80211_sta_freq`, читающий частоту связанной BSS, потому что Samsung опускает канал,
и `ensure_5g_ir`, срабатывающий только когда phy сообщает NO-IR, это образец. Если исправление нельзя ограничить так,
оно относится в форк. Тестируйте на реальном телефоне, эмулятора для Wi-Fi чипа нет, и скажите в
своём PR, на каких устройствах, драйверах, диапазонах и вышестоящих интерфейсах вы тестировали; непроверенное заявление считается
непроверенным.

## Сборка

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

Что делают скрипты и что делает CI, описано в разделе «Сборка» файла
[How-It-Works.ru.md](./Documentation/How-It-Works.ru.md). Релиз это тег: `make bump NEW=x.y.z`,
закоммитьте его как `VirtualAP: bump vX.Y.Z`, затем отправьте `main` и тег `vX.Y.Z`.

## Стиль кода

Четыре правила в AGENTS.md, раздел «Стиль». Сверх них:

- Backend: coreutils это `$ECHO`, `$GREP`, `$SED` и подобные, никогда голыми именами и никогда в кавычках.
  Вывод это `log`, `warn`, `error`. Подкоманды это `cmd_*`, помощники Droidspaces `ds_*`,
  пробы радио `phy_*`, политика `pick_*` и `resolve_*`. Разбор зеркалит настройку шаг за шагом,
  и каждый шаг терпит отсутствие. Каждый демон пишет в свой файл в `logs/`.
- App: состояние живёт в ViewModel, а не в stateful-композабле и не в синглтоне
  `util`. Списки рендерятся через `LazyColumn` со стабильными ключами. Цвета берутся из
  `MaterialTheme.colorScheme`, типографика из `MaterialTheme.typography`, тайминги из
  `AnimationUtils`, каждое другое визуальное значение из [DESIGN.ru.md](./DESIGN.ru.md). Любое значение, которое
  попадает в root-оболочку, проходит через `Backend.quote()` или валидатор по белому списку.

Коммиты подписываются и получают префиксы, как сказано в AGENTS.md, раздел «Коммиты».

## Переводы

Переводчики работают в Weblate, который коммитит в папки `values-<lang>/`; pull request
касается только английского в `res/values/`. Как настроены компоненты, описано в
разделе «Переводы» файла [How-It-Works.ru.md](./Documentation/How-It-Works.ru.md).

Добавление строки:

- Поместите её в `strings.xml` под комментарием для её экрана или в `plurals.xml`, если число
  меняет её формулировку. Kotlin читает её через `stringResource` или `getString` вне
  композиции.
- Плейсхолдеры позиционные, `%1$s`, чтобы перевод мог их переставить. Скажите, что каждый из них
  значит, в комментарии прямо над строкой, который Weblate показывает переводчику; сделайте то же
  для короткого слова, которое может значить две вещи.
- Одна целая фраза на строку. Никогда не склеивайте переведённые куски.
- Пути, команды, ключи настроек и то, что печатает движок, остаются в коде, на английском. Значение
  в файле, которое не меняет ни один язык, название продукта или пример адреса, помечается
  `translatable="false"`.

## Инвентарь переиспользования

Поищите grep по этому списку, прежде чем писать что-то новое. Если рядом есть что-то похожее, расширьте его,
а не добавляйте соседа. Пути Android относительны к
`Android/app/src/main/java/com/virtualap/app/`; модули движка это пути в `src/`, заголовок каждой
папки в `src/include/<folder>/`.
Этот список отвечает на вопрос «как это назвать»; DESIGN.md отвечает «как это должно выглядеть», когда
здесь ничего не подходит. Общие компоненты перенесены из Droidspaces; когда появится вызывающий для одного
из других его компонентов, скопируйте вышестоящий файл, а не пишите новый.

Формы и поля ввода:

- `DsDropdown(label, selected, options, displayName, onSelect, ...)` в
  `ui/component/DsDropdown.kt`: любое поле выбора. Никогда не пишите `ExposedDropdownMenuBox` вручную.
- `DsTextFieldDefaults.colors()` в `ui/component/DsTextFieldDefaults.kt`: каждый
  `OutlinedTextField`.
- `rememberClearFocus()`, который использует `DsDialog`, и `ClearFocusOnClickOutside`, корень главного
  экрана, в `ui/util/FocusUtils.kt`: убрать клавиатуру при нажатии вовне.

Диалоги и шторки:

- `DsDialog(onDismiss, modifier, footer) { }` в `ui/component/DsDialog.kt`: каждый диалог. Никогда
  не задавайте ширину, отступы или прокрутку.
- `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)` в
  `ui/component/DialogFooterRow.kt`: две кнопки в диалоге с выбором, в слоте
  `footer`.
- `DialogCloseButton(onClick, enabled)` в `ui/component/DialogCloseButton.kt`: квадрат закрытия 36dp
  в строке заголовка, для содержимого, которое закрывается сверху.
- `LogActionRow(logs, isBlocking, onClear)` в `ui/component/LogActionRow.kt`: строка Clear и
  Copy над консолью лога; `onClear = null` даёт только Copy.
- `DeviceRow`, `DeviceSheet(title, onDismiss, content)`, `BlockedList`, `AllowedList`,
  `BlockedDevicesSheet`, `AllowedDevicesSheet` и `rememberDeviceAction(reload)` в
  `ui/component/DeviceList.kt`: устройство со своим действием, шторка, в которой они лежат, два списка,
  и выполнение изменения для них с озвученным отказом.
- `AddDeviceDialog` и `AllowOnlyDialog` в `ui/component/AddDeviceDialog.kt`: поле MAC,
  которое принимает только шестнадцатеричные цифры, и кто вылетает, когда входить могут только разрешённые устройства.
- `Caption(text)` в `ui/component/Caption.kt`: строка вторичного текста.
- Шторка QR Wi-Fi, шторка устройств и шторка логов приватны для `ui/screen/MainScreen.kt`,
  диалог About и выбор языка для `ui/screen/SettingsScreen.kt`. Не импортируйте и не копируйте их.

Панели, настройка, обратная связь:

- `PrimaryActionBottomBar(label, icon, onClick, ...)` в
  `ui/component/PrimaryActionBottomBar.kt`: панель Start и Stop. Перегрузка со
  слотом `content` для кнопки, которая меняет своё содержимое.
- `SetupPage`, `SetupFrame`, `SetupHero`, `SetupActionBar` в `ui/component/SetupHero.kt`:
  каждый экран потока настройки, пейджер приветствия, проверка root и установщик. См.
  DESIGN.md, «Поток настройки».
- `PullToRefreshWrapper(onRefresh) { }` в `ui/component/PullToRefreshWrapper.kt`: любое тело экрана с
  обновлением жестом.
- `ToggleCard` в `ui/component/ToggleCard.kt`: строка переключателя в своей карточке,
  опции главного экрана. `SwitchItem` в `ui/component/SwitchItem.kt`: плоская строка переключателя внутри
  сгруппированной поверхности, экран Settings.
- `StatusPill(label, color)` в `ui/component/StatusPill.kt`: любой маленький чип статуса.
  `SectionHeader(text)` в `ui/component/SectionHeader.kt`: заголовок над группой карточек.
- `CardContentPadding`, `CardHeaderHeight` в `ui/component/CardMetrics.kt`: любая карточка с
  заголовком и чипом, чтобы разделители совпадали.
- `LoadingIndicator(size, color)` с `LoadingSize`, и `FullScreenLoading(message)`, в
  `ui/util/LoadingIndicator.kt`: каждый спиннер и каждое состояние загрузки на весь экран.
- `TerminalConsole(logs, isProcessing, maxHeight)` и `ShimmerAnimation(enabled) { }` в
  `ui/component/TerminalConsole.kt`: прокручиваемый вид лога и его рамка.

Тема:

- `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)` в `ui/theme/Theme.kt`:
  единственный корень темы, применяемый в `MainActivity`.
- `rememberThemeState()` и `ThemeState` в `ui/theme/ThemeStateHolder.kt`: живые
  настройки темы.
- `ThemePalette` в `ui/theme/Color.kt`: добавление акцентной палитры, здесь и нигде больше.
  `Theme.kt` и `Color.kt` единственные файлы, которые называют цвет буквально.
- `MaterialTheme.typography.*` и `JetBrainsMono` в `ui/theme/Type.kt`: все стили текста
  и моношрифт для текста логов и кода.
- `AnimationUtils` в `util/AnimationUtils.kt`: спецификации tween. Никогда литеральный `tween(300)`.
- `AccentColorPicker` и `ColorPaletteSwatch` в `ui/component/`: выбор палитры.
- Объекта токенов отступов или формы нет; числа живут в DESIGN.md и пишутся
  литеральными dp.

Навигация:

- `Screen` и `when` по нему в `MainActivity.kt`: единственное место, где выбирается
  экран верхнего уровня, сначала флаг приветствия, затем состояние root и установки, затем флаг Settings. Новый
  экран идёт туда.

Оболочка, root и backend:

- `Backend.quote(value)` и `Backend.startAp` в `util/Backend.kt`: каждое динамическое значение, которое
  попадает в root-команду, и префикс команды движка (с откатом к скрипту, который
  извлекла старая версия, пока поток настройки его не заменит).
- `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled` в `util/APManager.kt`:
  управление `virtualap`, и `getContainers`/`parseContainers`, которые спрашивают Droidspaces о его
  работающих контейнерах напрямую. Никогда не собирайте одну из этих команд вручную.
- `Hotspot.start(cfg)/stop()/refresh()`, `phase`, `status`, `actionLogs` в `util/Hotspot.kt`:
  запуск или остановка точки доступа откуда угодно и чтение состояния сеанса; также показывает
  уведомление о работе и будит плитку.
- `APConfig` с `isValid()`, `isValidIpv4`, `isValidDnsList`, `channelsForBand` в
  `util/APConfig.kt`: сохранённая конфигурация точки доступа, единственная проверка при запуске и проверки адресов,
  которые поля делят с ней.
- `BackendLogger` и `classifyLine(line)` в `util/BackendLogger.kt`: приёмник логов, который получают вызовы
  AP, и маркировка уровней `[ERROR]`/`[WARN]`.
- `PreferencesManager` в `util/PreferencesManager.kt`: собственные настройки приложения.
  `APManager.loadConfig/saveConfig`: конфигурация точки доступа, читаемая и записываемая целиком в `ap.conf`.
  `Constants` в `util/Constants.kt`: каждый путь и ключ настроек.
- `RootChecker.checkRootAccess(reprompt)` и `RootStatus` в `util/RootChecker.kt`;
  `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install` в
  `util/VirtualAPInstaller.kt`; `ModuleInstaller.install` в `util/ModuleInstaller.kt`, загрузочный
  модуль; `QrCodeGenerator.wifiPayload/encode` в
  `util/QrCodeGenerator.kt`, единственный вызывающий zxing.
- `HotspotTileService`, `HotspotStopReceiver` и `HotspotBootReceiver`: переключатель быстрых настроек,
  действие Stop в уведомлении и сообщение загрузочного модуля о том, что он запустил точку доступа.
  Ни один не хранит собственного состояния.
- `LocaleHelper.availableLanguages/currentLanguage/changeLanguage` в `util/LocaleHelper.kt`:
  язык приложения, единственный вызывающий `AppCompatDelegate.setApplicationLocales`.
- `AppViewModel` владеет флагом приветствия, состоянием root и состоянием установки; `APViewModel` владеет редактируемой конфигурацией,
  списками интерфейсов и контейнеров и отслеживаемым логом, и передаёт запуск и остановку в
  `Hotspot`. Опрос состояния каждые 3 с живёт в `MainScreen`, привязанный к его жизненному циклу.
- Глобальная конфигурация libsu живёт в `VirtualAPApplication.kt`, и только там.

Движок (`src/`):

- `core/`: `proc` (каждая запускаемая программа, pid-файлы и блокировка, которая позволяет одному запуску или
  остановке идти одновременно), `io` (лог: `vap_log`, `vap_warn`, `vap_error`, `log_stamp`;
  `read_file`, `write_file`, `write_atomic`; парсеры `valid_ip4`, `prefix_parse`,
  `subnet_from_gw`, и `csv_each` для списков через запятую в `-d`).
- `cli/`: `options` (таблица `vap_keys`, `opt_set`, `opts_parse`, `print_usage`), `conf`
  (`ap.conf` и `run.state`).
- `net/`: `nl` (транспорт rtnetlink, `nl_dump`, и помощники пространств имён `netns_open`,
  `in_netns`), `link` (линки, адреса и соседи), `route` (маршруты, правила политики и
  таблицы netd: `upstream_resolve`, `upstream_v6_iface`), `ipt/`, отделённый, потому что на
  musl заголовки netfilter ядра и сетевые заголовки libc не могут встретиться в одном файле:
  `ipt` (движок `ip_tables`), `ipt_rules` (FORWARD, MASQUERADE, проверка наличия,
  проба NAT, `ipt_bin`, правка TTL).
- `radio/`: `driver` (nl80211 через genetlink: создать и удалить ap0, phy, частоту
  станции, ширину; управляющий сокет wpa; `wifi_mac`, заводской MAC), `channel`
  (политика диапазона, канала и ширины, и `ensure_5g_ir`).
- `mode/`: `ap` (ap0, демоны, планирование радио, удержание сигнала, общее для обоих режимов, и
  `hostapd` и `dnsmasq`: их конфигурации и запуски, число клиентов), `routed` (с
  закреплёнными правилами политики, которые используют оба семейства), `ipv6` (`ipv6_decide`, настройка, разбор, NAT66 и
  общий /64 с обнаружением его клиентов), `managed` (контейнер Droidspaces: его pid,
  подготовка OpenWrt, маски RPS), `maclist` (файлы заблокированных и разрешённых устройств,
  которые являются `deny_mac_file` и `accept_mac_file` для `hostapd`, и сообщение работающему
  `hostapd`).
- `monitor/`: `monitor` (цикл демона), `upstream` (`reprobe`).
- `cmd/`: команды, которые вызывает приложение, диспетчеризуемые из `main.c`: `start` (`start` и
  `stop`), `status` (только читающие: `status`, `clients`, `interfaces`, `caps`) и
  `block` (`block`, `unblock`, `blocked`, `allow`, `disallow`, `allowed`, `allow-only`).

## Открытие PR

Форкните, работайте в ветке, открывайте PR против `main` и оставайтесь на связи: PR без ответа
закрывается, а замечания учитываются в том же PR, а не в новом. CLA нет; отправляя вклад, вы
соглашаетесь, что он может распространяться под лицензией проекта.

Каждый PR говорит:

1. Какую реальную проблему он решает. «Я этого хотел» не считается; скажите, что ломается, падает или
   отсутствует у реальных пользователей на реальном железе.
2. Скриншоты или вывод терминала, показывающие, что это работает.
3. Проверенные окружения: для приложения устройство, версия Android и провайдер root; для
   backend устройство, SoC и Wi-Fi чип, драйвер, диапазон и ширина, вышестоящий интерфейс (мобильный интернет,
   Wi-Fi, VPN), и проверялся ли управляемый режим.
4. Для изменений UI правила DESIGN.md, которым вы следовали, и от какого вы отступили и
   почему. Радиус или цвет, который расходится с DESIGN.md без причины, отправляется назад.

Что принимается: исправления с воспроизведением и подтверждённым решением, функции, проверенные
на разных устройствах, усиление безопасности, измеренные улучшения производительности, удаления и
исправления документации. Что отклоняется: изменение движка, которое запускает инструмент для того, что
может сделать netlink или собственные файлы ядра, или которое раздувает модуль за пределы одной заботы;
предсобранный или динамически слинкованный бинарник;
функция, о которой не сообщал ни один реальный пользователь или которая работает только на одной конфигурации железа; код,
который автор не может объяснить; изменение приложения, которое ломает Android 8; вторая копия чего-то
из инвентаря; любое динамическое значение, попадающее в root-оболочку без кавычек; всё, что регрессирует
то, что работало, как бы ни было полезно новое поведение.

Слитая функция становится вашей: когда новая версия Android или драйвер её ломает, вы
должны её исправить, а функция, чей автор недоступен, удаляется. Участник, чьи PR
снова и снова отклоняют по одним и тем же причинам: нет реальной проблемы, не проверено или раздувание
кодовой базы, блокируется; счётчика предупреждений нет, решение за мейнтейнером.

## Безопасность

Всё, что позволяет клиенту точки доступа или значению, введённому в приложении, достичь root-оболочки
без кавычек, это уязвимость. Не открывайте публичный issue; сообщите приватно на
droidcasts@protonmail.com или в [t.me/ravindu](https://t.me/ravindu) с воспроизведением,
затронутыми конфигурациями и устройством или драйвером, и не раскрывайте публично, пока исправление не
вышло. Патчи усиления безопасности проверяются первыми.
