# VirtualAP에 기여하기

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

존재하지 않는 기능이 망가진 기능보다 낫습니다. VirtualAP는 루팅된 휴대폰에서 root로 실행되며, 기본적인 것들에 대해서도 드라이버마다 의견이 다른 Wi-Fi 칩 위에서 동작합니다. 그래서 당신의 휴대폰에서는 되고 남의 휴대폰에서는 깨지는 패치는 기여가 아니라 회귀입니다. 그리고 그 일을 해내는 열 줄이 같은 일을 하는 백 줄보다 낫습니다. 더하기 전에 지우고, 쓰기 전에 재사용하세요.

## 반드시 지켜야 할 것

에이전트를 포함해 이 저장소를 바꾸는 모든 사람을 위한 규칙은 [AGENTS.ko.md](./AGENTS.ko.md)에 있습니다. 스타일, 커밋 형식, 병목 지점, 강한 제약입니다. 요약하면 Android 전용, 엔진은 커널과 직접 대화하는 musl 위의 단일 스레드 C, 모듈마다 관심사 하나, 앱은 Android 8에서 동작, 바이너리는 `scripts/build-static.sh`와, `virtualap`의 경우 루트 `Makefile`에서만 나옵니다.

VirtualAP는 Qualcomm, Broadcom FullMAC(삼성), MediaTek 칩에서 실행됩니다. 한 칩의 특이 동작에 대한 수정은 백엔드가 런타임에 감지, 조건부 경로, 대체 경로로 적응하고 다른 칩의 동작을 바꾸지 않을 때만 받습니다. 삼성이 채널을 생략하기 때문에 연결된 BSS의 주파수를 읽는 `nl80211_sta_freq`와, phy가 NO-IR을 보고할 때만 작동하는 `ensure_5g_ir`가 그 패턴입니다. 그런 식으로 게이트할 수 없는 수정은 포크에 속합니다. 실제 휴대폰에서 테스트하세요. Wi-Fi 칩용 에뮬레이터는 없습니다. 그리고 PR에 어떤 기기, 드라이버, 대역, 업스트림에서 테스트했는지 밝히세요. 테스트하지 않은 주장은 테스트하지 않은 것으로 칩니다.

## 빌드

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

스크립트가 하는 일과 CI가 하는 일은 [How-It-Works.ko.md](./Documentation/How-It-Works.ko.md)의 Building 절에 있습니다. 릴리스는 태그입니다. `make bump NEW=x.y.z`로 올리고, `VirtualAP: bump vX.Y.Z`로 커밋한 다음, `main`과 태그 `vX.Y.Z`를 푸시합니다.

## 코드 스타일

네 가지 규칙은 AGENTS.md의 Style에 있습니다. 그 위에 다음이 더해집니다.

- 백엔드: coreutils는 `$ECHO`, `$GREP`, `$SED` 같은 변수로 쓰고, 맨 이름도, 따옴표 없이도 쓰지 않습니다. 출력은 `log`, `warn`, `error`입니다. 서브커맨드는 `cmd_*`, Droidspaces 도우미는 `ds_*`, 무선 탐색은 `phy_*`, 정책은 `pick_*`와 `resolve_*`입니다. 정리는 설정을 단계마다 그대로 비추고, 모든 단계는 대상이 없어도 견딥니다. 각 데몬은 `logs/` 아래 자기 파일에 기록합니다.
- 앱: 상태는 상태를 가진 컴포저블도, `util` 싱글턴도 아닌 ViewModel에 삽니다. 목록은 `LazyColumn`과 안정적인 키로 그립니다. 색은 `MaterialTheme.colorScheme`, 타이포그래피는 `MaterialTheme.typography`, 타이밍은 `AnimationUtils`, 나머지 시각적 값은 모두 [DESIGN.ko.md](./DESIGN.ko.md)에서 옵니다. root 셸에 도달하는 모든 값은 `Backend.quote()`나 허용 목록 검증기를 거칩니다.

커밋은 AGENTS.md의 Commits가 말하는 대로 서명하고 접두사를 붙입니다.

## 번역

번역자는 Weblate에서 작업하며, Weblate는 `values-<lang>/` 폴더에 커밋합니다. 풀 리퀘스트는 `res/values/`의 영어만 건드립니다. 컴포넌트를 어떻게 구성하는지는 [How-It-Works.ko.md](./Documentation/How-It-Works.ko.md)의 Translations 절에 있습니다.

문자열을 추가할 때:

- 그 화면을 위한 주석 아래 `strings.xml`에 넣거나, 개수에 따라 문구가 바뀌면 `plurals.xml`에 넣습니다. Kotlin은 `stringResource`로, 컴포지션 밖에서는 `getString`으로 읽습니다.
- 자리 표시자는 위치 기반인 `%1$s`를 쓰므로 번역에서 순서를 바꿀 수 있습니다. 각 자리 표시자가 무엇인지는 문자열 바로 위의 주석에 적고, Weblate가 그것을 번역자에게 보여 줍니다. 두 가지로 읽힐 수 있는 짧은 단어도 같은 방식으로 합니다.
- 문자열 하나에 완전한 구절 하나. 번역된 조각을 절대 이어 붙이지 않습니다.
- 경로, 명령, 환경설정 키, 엔진이 출력하는 것은 코드에 영어로 남깁니다. 파일에서 어떤 언어도 바꾸지 않는 값, 제품 이름, 예시 주소는 `translatable="false"`입니다.

## 재사용 목록

새로 무언가를 쓰기 전에 이 목록을 grep하세요. 가까운 것이 있으면 형제를 추가하지 말고 그것을 확장하세요. Android 경로는 `Android/app/src/main/java/com/virtualap/app/` 기준이며, 엔진 모듈은 `src/` 아래 경로이고, 각 폴더의 헤더는 `src/include/<folder>/` 아래에 있습니다. 이 목록은 "무엇을 뭐라고 부르는가"에 답하고, 여기에 맞는 것이 없을 때 DESIGN.md가 "어떻게 보여야 하는가"에 답합니다. 공유 컴포넌트는 Droidspaces에서 옮겨 왔습니다. 그 다른 컴포넌트를 쓰는 호출자가 나타나면 새로 쓰지 말고 업스트림 파일을 복사하세요.

폼과 입력:

- `ui/component/DsDropdown.kt`의 `DsDropdown(label, selected, options, displayName, onSelect, ...)`: 모든 선택 필드. `ExposedDropdownMenuBox`를 직접 만들지 않습니다.
- `ui/component/DsTextFieldDefaults.kt`의 `DsTextFieldDefaults.colors()`: 모든 `OutlinedTextField`.
- `ui/util/FocusUtils.kt`의, `DsDialog`가 쓰는 `rememberClearFocus()`와 메인 화면의 루트인 `ClearFocusOnClickOutside`: 바깥을 탭하면 키보드를 내립니다.

다이얼로그와 시트:

- `ui/component/DsDialog.kt`의 `DsDialog(onDismiss, modifier, footer) { }`: 모든 다이얼로그. 너비, 패딩, 스크롤을 직접 설정하지 않습니다.
- `ui/component/DialogFooterRow.kt`의 `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)`: 선택이 있는 다이얼로그의 두 버튼으로, `footer` 슬롯에 둡니다.
- `ui/component/DialogCloseButton.kt`의 `DialogCloseButton(onClick, enabled)`: 헤더 행의 36dp 닫기 사각형으로, 위에서 닫는 내용을 위한 것입니다.
- `ui/component/LogActionRow.kt`의 `LogActionRow(logs, isBlocking, onClear)`: 로그 콘솔 위의 지우기와 복사 행. `onClear = null`이면 복사만 나옵니다.
- `ui/component/DeviceList.kt`의 `DeviceRow`, `DeviceSheet(title, onDismiss, content)`, `BlockedList`, `AllowedList`, `BlockedDevicesSheet`, `AllowedDevicesSheet`, `rememberDeviceAction(reload)`: 동작이 붙은 기기, 그것들이 들어 있는 시트, 두 목록, 그리고 실패를 소리 내어 알리면서 그 목록에 변경을 적용하기.
- `ui/component/AddDeviceDialog.kt`의 `AddDeviceDialog`와 `AllowOnlyDialog`: 16진수만 받는 MAC 필드, 그리고 허용된 기기만 참여할 수 있을 때 누가 빠지는지.
- `ui/component/Caption.kt`의 `Caption(text)`: 보조 텍스트 한 줄.
- Wi-Fi QR 시트, 기기 시트, 로그 시트는 `ui/screen/MainScreen.kt` 전용이고, About 다이얼로그와 언어 선택기는 `ui/screen/SettingsScreen.kt` 전용입니다. 가져오거나 복사하지 않습니다.

바, 설정 흐름, 피드백:

- `ui/component/PrimaryActionBottomBar.kt`의 `PrimaryActionBottomBar(label, icon, onClick, ...)`: 시작과 정지 바. `content` 슬롯을 받는 오버로드는 내용을 바꿔 끼우는 버튼을 위한 것입니다.
- `ui/component/SetupHero.kt`의 `SetupPage`, `SetupFrame`, `SetupHero`, `SetupActionBar`: 설정 흐름의 모든 화면, 환영 페이저, root 확인, 설치 프로그램. DESIGN.md의 "The setup flow"를 보세요.
- `ui/component/PullToRefreshWrapper.kt`의 `PullToRefreshWrapper(onRefresh) { }`: 당겨서 새로 고치는 모든 화면 본문.
- `ui/component/ToggleCard.kt`의 `ToggleCard`: 자기 카드 안에 있는 스위치 행, 메인 화면의 옵션. `ui/component/SwitchItem.kt`의 `SwitchItem`: 묶인 표면 안의 납작한 스위치 행, 설정 화면.
- `ui/component/StatusPill.kt`의 `StatusPill(label, color)`: 모든 작은 상태 칩. `ui/component/SectionHeader.kt`의 `SectionHeader(text)`: 카드 묶음 위의 제목.
- `ui/component/CardMetrics.kt`의 `CardContentPadding`, `CardHeaderHeight`: 제목과 필 헤더가 있는 모든 카드. 구분선이 나란히 맞습니다.
- `ui/util/LoadingIndicator.kt`의 `LoadingSize`와 함께 쓰는 `LoadingIndicator(size, color)`, 그리고 `FullScreenLoading(message)`: 모든 스피너와 모든 전체 화면 로딩 상태.
- `ui/component/TerminalConsole.kt`의 `TerminalConsole(logs, isProcessing, maxHeight)`와 `ShimmerAnimation(enabled) { }`: 스크롤되는 로그 뷰와 그 프레임.

테마:

- `ui/theme/Theme.kt`의 `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)`: 유일한 테마 루트로, `MainActivity`에서 적용합니다.
- `ui/theme/ThemeStateHolder.kt`의 `rememberThemeState()`와 `ThemeState`: 실시간 테마 환경설정.
- `ui/theme/Color.kt`의 `ThemePalette`: 액센트 팔레트 추가는 여기서만 합니다. `Theme.kt`와 `Color.kt`는 색을 문자 그대로 적는 유일한 파일입니다.
- `ui/theme/Type.kt`의 `MaterialTheme.typography.*`와 `JetBrainsMono`: 모든 텍스트 스타일, 그리고 로그와 코드 텍스트용 고정폭 글꼴.
- `util/AnimationUtils.kt`의 `AnimationUtils`: 트윈 명세. `tween(300)` 같은 리터럴을 절대 쓰지 않습니다.
- `ui/component/`의 `AccentColorPicker`와 `ColorPaletteSwatch`: 팔레트 선택기.
- 간격이나 모양 토큰 객체는 없습니다. 숫자는 DESIGN.md에 있고 리터럴 dp로 적습니다.

내비게이션:

- `MainActivity.kt`의 `Screen`과 그에 대한 `when`: 최상위 화면을 고르는 유일한 곳으로, 먼저 환영 플래그, 그다음 root와 설치 상태, 그다음 설정 플래그입니다. 새 화면은 그곳에 들어갑니다.

셸, root, 백엔드:

- `util/Backend.kt`의 `Backend.quote(value)`와 `Backend.startAp`: root 명령에 도달하는 모든 동적 값, 그리고 엔진의 명령 접두사(설정 흐름이 그것을 교체할 때까지, 이전 릴리스가 풀어 놓은 스크립트로 되돌아가는 대체 경로 포함).
- `util/APManager.kt`의 `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled`: `virtualap` 구동, 그리고 Droidspaces에 실행 중인 컨테이너를 직접 묻는 `getContainers`/`parseContainers`. 이 명령들 중 하나를 손으로 조립하지 않습니다.
- `util/Hotspot.kt`의 `Hotspot.start(cfg)/stop()/refresh()`, `phase`, `status`, `actionLogs`: 어디서든 AP를 시작하거나 정지하고 세션 상태를 읽기. 실행 중 알림을 띄우고 타일도 깨웁니다.
- `util/APConfig.kt`의 `isValid()`, `isValidIpv4`, `isValidDnsList`, `channelsForBand`를 가진 `APConfig`: 저장된 핫스팟 설정, 유일한 시작 검증, 그리고 필드들이 그것과 공유하는 주소 검사.
- `util/BackendLogger.kt`의 `BackendLogger`와 `classifyLine(line)`: AP 호출이 받는 로그 싱크, 그리고 `[ERROR]`/`[WARN]` 수준 태깅.
- `util/PreferencesManager.kt`의 `PreferencesManager`: 앱 자체의 설정. `APManager.loadConfig/saveConfig`: `ap.conf`에 통째로 읽고 쓰는 핫스팟 설정. `util/Constants.kt`의 `Constants`: 모든 경로와 환경설정 키.
- `util/RootChecker.kt`의 `RootChecker.checkRootAccess(reprompt)`와 `RootStatus`; `util/VirtualAPInstaller.kt`의 `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install`; 부팅 모듈인 `util/ModuleInstaller.kt`의 `ModuleInstaller.install`; 유일한 zxing 호출자인 `util/QrCodeGenerator.kt`의 `QrCodeGenerator.wifiPayload/encode`.
- `HotspotTileService`, `HotspotStopReceiver`, `HotspotBootReceiver`: 빠른 설정 토글, 알림의 정지 액션, 그리고 핫스팟을 시작했다는 부팅 모듈의 말. 스스로 상태를 가진 것은 없습니다.
- `util/LocaleHelper.kt`의 `LocaleHelper.availableLanguages/currentLanguage/changeLanguage`: 앱의 언어, `AppCompatDelegate.setApplicationLocales`의 유일한 호출자.
- `AppViewModel`은 환영 플래그, root 상태, 설치 상태를 소유하고, `APViewModel`은 편집 가능한 설정, 인터페이스와 컨테이너 목록, 따라가는 로그를 소유하며 시작과 정지를 `Hotspot`으로 넘깁니다. 3초 상태 폴링은 `MainScreen`에 살고 그 생명주기에 묶입니다.
- 전역 libsu 구성은 `VirtualAPApplication.kt`에 있으며, 오직 거기에만 있습니다.

엔진(`src/`):

- `core/`: `proc`(모든 프로그램 실행, pid 파일, 한 번에 하나의 시작이나 정지만 돌게 하는 잠금), `io`(로그: `vap_log`, `vap_warn`, `vap_error`, `log_stamp`; `read_file`, `write_file`, `write_atomic`; 파서 `valid_ip4`, `prefix_parse`, `subnet_from_gw`, 그리고 `-d`의 쉼표 목록을 위한 `csv_each`).
- `cli/`: `options`(`vap_keys` 표, `opt_set`, `opts_parse`, `print_usage`), `conf`(`ap.conf`와 `run.state`).
- `net/`: `nl`(rtnetlink 전송, `nl_dump`, 네임스페이스 도우미 `netns_open`, `in_netns`), `link`(링크, 주소, 이웃), `route`(라우트, 정책 규칙, netd의 표: `upstream_resolve`, `upstream_v6_iface`), `ipt/`는 musl에서 커널의 netfilter 헤더와 libc의 네트워킹 헤더가 한 파일에서 만날 수 없기 때문에 따로 둡니다: `ipt`(`ip_tables` 엔진), `ipt_rules`(FORWARD, MASQUERADE, 존재 확인, NAT 탐색, `ipt_bin`, TTL 보정).
- `radio/`: `driver`(genetlink 위의 nl80211: ap0 생성과 삭제, phy, 스테이션의 주파수, 폭; wpa 제어 소켓; 출하 시 MAC인 `wifi_mac`), `channel`(대역, 채널, 폭 정책, 그리고 `ensure_5g_ir`).
- `mode/`: `ap`(ap0, 데몬, 무선 계획, 두 모드가 공유하는 신호 유지, 그리고 `hostapd`와 `dnsmasq`: 그 설정과 시작, 클라이언트 수), `routed`(두 주소 체계가 쓰는 고정 정책 규칙 포함), `ipv6`(`ipv6_decide`, 설정, 정리, NAT66, 클라이언트 탐색이 있는 공유 /64), `managed`(Droidspaces 컨테이너: pid, OpenWrt 프로비저닝, RPS 마스크), `maclist`(차단된 기기와 허용된 기기의 파일, 즉 `hostapd`의 `deny_mac_file`과 `accept_mac_file`, 그리고 실행 중인 `hostapd`에 알리기).
- `monitor/`: `monitor`(데몬의 루프), `upstream`(`reprobe`).
- `cmd/`: 앱이 호출하는 명령으로, `main.c`에서 디스패치합니다: `start`(`start`와 `stop`), `status`(읽기 전용 명령: `status`, `clients`, `interfaces`, `caps`), `block`(`block`, `unblock`, `blocked`, `allow`, `disallow`, `allowed`, `allow-only`).

## PR 열기

포크하고, 브랜치에서 작업하고, `main`을 대상으로 PR을 열고, 응답을 유지하세요. 답이 없는 PR은 닫히고, 피드백은 새 PR이 아니라 같은 PR에서 반영합니다. CLA는 없습니다. 제출하면 기여가 프로젝트 라이선스로 배포될 수 있음에 동의하는 것입니다.

모든 PR은 다음을 말합니다.

1. 어떤 실제 문제를 해결하는지. "이게 갖고 싶었다"는 그것이 아닙니다. 실제 사용자가 실제 하드웨어에서 무엇이 깨지고, 실패하고, 빠져 있는지 말하세요.
2. 동작함을 보여 주는 스크린샷이나 터미널 출력.
3. 테스트한 환경: 앱이라면 기기, Android 버전, root 제공자; 백엔드라면 기기, SoC, Wi-Fi 칩, 드라이버, 대역과 폭, 업스트림(모바일 데이터, Wi-Fi, VPN), 그리고 관리 모드를 시험했는지.
4. UI 변경이라면 따른 DESIGN.md 규칙과, 어느 규칙에서 벗어났고 그 이유. 이유 없이 DESIGN.md와 어긋나는 반경이나 색은 되돌려 보냅니다.

병합되는 것: 재현과 검증된 해결이 있는 수정, 여러 기기에서 검증된 기능, 보안 강화, 측정된 성능 개선, 삭제, 문서 수정. 거부되는 것: netlink나 커널 자체 파일로 할 수 있는 일에 도구를 실행하는 엔진 변경, 모듈을 자기 관심사 밖으로 키우는 변경, 미리 빌드되었거나 동적으로 링크된 바이너리, 실제 사용자가 보고한 적 없거나 하나의 하드웨어 구성에서만 동작하는 기능, 저자가 설명할 수 없는 코드, Android 8을 깨는 앱 변경, 목록에 있는 것의 두 번째 사본, 따옴표 없이 root 셸에 도달하는 모든 동적 값, 새 동작이 아무리 유용해도 동작하던 것을 회귀시키는 모든 것.

병합된 기능은 앞으로 당신의 것입니다. 새 Android 버전이나 드라이버가 그것을 깨면 고칠 것으로 기대되며, 저자와 연락이 닿지 않는 기능은 제거됩니다. 같은 이유로 계속 PR이 거부되는 기여자, 즉 실제 문제가 없거나, 테스트하지 않았거나, 코드베이스를 부풀리는 경우는 차단됩니다. 스트라이크 횟수는 없고 결정은 메인테이너의 몫입니다.

## 보안

핫스팟 클라이언트나 앱에 입력된 값이 따옴표 없이 root 셸에 도달하게 하는 것은 무엇이든 취약점입니다. 공개 이슈를 열지 마세요. 재현, 영향을 받는 구성, 기기나 드라이버와 함께 droidcasts@protonmail.com이나 [t.me/ravindu](https://t.me/ravindu)로 비공개로 알리고, 수정이 나올 때까지 공개를 보류하세요. 강화 패치는 먼저 검토됩니다.
