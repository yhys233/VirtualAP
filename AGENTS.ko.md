# AGENTS.md

[English](AGENTS.md) | [简体中文](AGENTS.zh-CN.md) | [繁體中文](AGENTS.zh-TW.md) | [Русский](AGENTS.ru.md) | [Français](AGENTS.fr.md) | [Deutsch](AGENTS.de.md) | [日本語](AGENTS.ja.md) | [한국어](AGENTS.ko.md)

이 저장소에서 작업하는 AI 코딩 에이전트를 위한 지침입니다. 사람은 [CONTRIBUTING.ko.md](./CONTRIBUTING.ko.md)를 읽으세요. 그 문서에는 이곳에 이미 존재하는 것들의 목록이 담겨 있습니다. 동작 방식은 [Documentation/How-It-Works.ko.md](./Documentation/How-It-Works.ko.md)에 있습니다.

## 이것은 무엇인가

VirtualAP는 루팅된 휴대폰을 Wi-Fi 액세스 포인트로 바꿉니다. 두 부분입니다.

- `src/`는 엔진인 `virtualap`으로, root로 실행되는 정적 C 바이너리이며 두 개를 더 구동합니다. 그 둘은 `scripts/build-static.sh`가 `externals/` 아래의 git 서브모듈에서 빌드하는 `hostapd`와 `dnsmasq`입니다. `make aarch64 armhf`가 `virtualap`을 빌드합니다. 셋 모두 `backend/aarch64/`와 `backend/armhf/`에 들어가며, 이 디렉터리들은 gitignore 대상입니다.
- `Android/`는 root 셸을 통해 `virtualap`을 구동하는 Compose 앱입니다.

`prepareAssets` gradle 태스크는 두 아키텍처 트리를 `Android/app/build/` 아래의 생성된 assets 루트로 모으고, 앱은 첫 실행 때와 APK를 업데이트할 때마다 일치하는 아키텍처를 `/data/local/virtualap`에 배포하며, root 관리자를 위한 부팅 모듈도 함께 둡니다. chroot는 없습니다.

## 빌드

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

첫 앱 빌드 전에, 그리고 `externals/`나 빌드 스크립트가 바뀔 때마다 첫 번째를 실행합니다. 앱 빌드가 `make aarch64 armhf`를 스스로 실행하므로 `virtualap`은 항상 `src/`와 일치합니다. `backend/<arch>/`에 셋 중 하나가 없으면 `prepareAssets`가 실패합니다. `make check`가 테스트를 실행합니다. 버전은 `src/include/virtualap.h`의 `VAP_VERSION`에서 옵니다. 버전은 `make bump NEW=x.y.z`로만 올리며, 다른 곳에서는 올리지 않습니다. 빌드와 CI의 세부 사항은 How-It-Works.md의 Building 절에 있습니다.

## 커밋

- 모든 커밋에 서명을 남깁니다: `git commit -s`.
- AI 에이전트를 위한 `Co-Authored-By:` 트레일러는 절대 추가하지 않습니다. 사람 공동 저자는 괜찮습니다.
- 기존 이력에 맞춘 접두사:
  - `app:`는 Android 앱, 해당하는 경우에는 `app: fix:`와 `app: refactor:`
  - `backend:`는 `src/` 아래의 엔진, 엔진 버그 수정에는 `fix:`
  - `scripts:`는 `scripts/`와 `externals/` 아래의 정적 바이너리 빌드, `ci:`는 `.github/workflows/`, `docs:`는 문서
  - `fix(security):`는 보안과 관련된 모든 것, 어느 쪽 절반이든
  - `VirtualAP: bump vX.Y.Z`는 버전 올리기, 그리고 그 커밋에는 다른 것을 넣지 않습니다.

`ap:`, `build:`, `ui:`, `feat:`, `refactor:`는 오래된 이력에 나타납니다. 새 커밋에는 쓰지 않습니다.

## 스타일

네 가지 규칙입니다. 코드, 주석, 커밋 메시지, 문서에 모두 적용됩니다.

**em dash를 쓰지 않습니다.** 쉼표나 마침표로 끊거나 문장을 다시 씁니다.

**ASCII 배너 주석을 쓰지 않습니다.** `-----`의 줄도, `=====`도, 상자 모양 구획 제목도 쓰지 않습니다.

**주석은 사람이 쓴 것처럼 읽혀야 합니다.** 코드가 왜 그렇게 하는지, 그렇게 하지 않으면 무엇이 깨지는지 말합니다. 바로 아래 줄을 다시 말하는 주석은 생략합니다.

```sh
# Bad: bump the priority
PRIO_FROM_AP=7010

# Good: netd's own rules start at 10000, so 7010 wins before the VPN catch-all
PRIO_FROM_AP=7010
```

**동작하는 열 줄이 같은 일을 하는 백 줄보다 낫습니다.** 더하기 전에 지웁니다. 목표는 아무 데나 작은 diff가 아니라, 알맞은 자리에 작은 diff입니다.

문서에서는 모든 사실을 독자가 찾아볼 자리인 한 페이지에 한 번만 적고, 다른 페이지는 그것을 링크합니다. 분담은 고정입니다. README.md는 무엇을 누구를 위해 어떤 순서로 하는지, How-It-Works.md는 모든 기술적 사실을, 이 파일은 에이전트를 위한 규칙을, CONTRIBUTING.md는 사람이 변경을 병합받는 방법과 목록을, DESIGN.md는 UI 값을 담습니다. DESIGN.md의 표는 남겨 둡니다. 산문으로는 동작하지 않는 역할 대응과 크기이기 때문입니다. 다른 곳에서는 목록이나 문단으로 씁니다.

## UI 변경

`Android/`의 시각적인 모든 것은 [DESIGN.ko.md](./DESIGN.ko.md)를 따릅니다. 색, 타이포그래피, 간격, 반경, 액션 필 패턴이 모두 거기에 명시되어 있고, 그것들은 지어낸 것이 아니라 기존 앱에서 읽어낸 값이므로 따르는 것이 곧 가장 작은 diff입니다.

규칙이 정말로 맞지 않으면 벗어나되, 그 자리에 벗어나서 무엇을 얻는지 주석으로 남기고 PR에서도 말합니다. 문서화되지 않은 예외는 표류가 되어 다음 기여자가 "고쳐" 버립니다. DESIGN.md의 "Decided exceptions" 절은 결정으로 남긴 예외를 나열하니, 그것들은 "고치지" 않습니다.

## 무엇이든 쓰기 전에

이 목록을 따라 내려가며, 맞는 답이 나오는 첫 지점에서 멈춥니다.

1. 이것이 존재해야 합니까? 아무도 요청하지 않은 플래그, 절대 바뀌지 않는 값을 위한 손잡이, 구현이 하나뿐인 인터페이스라면 건너뛰고 한 줄로 그렇게 말합니다.
2. 여기에 이미 있습니까? 아래 "쓰기 전에 재사용"을 보세요. 먼저 grep합니다.
3. 플랫폼이 이미 합니까? 엔진에서는 musl의 libc와 커널 자체 인터페이스(netlink, `/proc/sys`), 앱 쪽에서는 Kotlin과 Java 표준 라이브러리와 Android 프레임워크입니다. `minSdk = 26`을 유념합니다.
4. 설치된 의존성이 그것을 덮습니까? 앱 쪽의 libsu, Compose, lifecycle, zxing, appcompat입니다. 몇 줄로 할 수 있는 일에 새 의존성을 절대 추가하지 않습니다.
5. 한 줄로 될까요? 그렇다면 한 줄입니다.
6. 그다음에야 동작하는 가장 작은 것을 씁니다.

두 답이 모두 맞습니까? 더 위의 것을 택하고 넘어갑니다.

이 목록은 해법을 줄일 뿐, 읽는 양을 줄이지 않습니다. 단계를 고르기 전에 변경이 건드리는 흐름을 따라가 봅니다. 잘못된 자리의 작은 diff는 게으른 수정이 아니라 두 번째 버그입니다. 버그 보고도 같습니다. 보고는 증상을 말하므로 고치기 전에 모든 호출자를 grep합니다. 공유 함수의 가드 하나가 호출자마다의 가드보다 작고, 보고가 언급하지 않은 형제들까지 고칩니다.

신뢰 경계의 입력 검증, 실패 시 닫히는 검사, 상태를 잃는 오류 처리, 요청자가 이름을 대고 요청한 것은 어떤 것도 단순화해서 없애지 않습니다.

## 쓰기 전에 재사용

CONTRIBUTING.md의 목록을 grep하고, 그다음 트리를 grep합니다. 다음은 병목 지점입니다. 하나를 우회하는 것은 지름길이 아니라 버그입니다.

- `Android/app/src/main/java/com/virtualap/app/util/Backend.kt`의 `Backend.quote()`는 root 셸에 도달하는 모든 동적 값을 감쌉니다. `virtualap`을 실행하는 클래스는 `APManager` 하나뿐이고, 그 경로를 적는 유일한 방법은 `Backend.startAp`입니다.
- `Hotspot`은 `APManager.start`와 `APManager.stop`의 유일한 호출자입니다. 시작과 정지 단계, 명령 로그, 실행 중 알림, 빠른 설정 타일 동기화를 소유하므로 화면과 타일이 항상 일치합니다. 시작 검증은 `APConfig.isValid()` 하나뿐이며, 화면의 버튼과 타일이 모두 그것을 묻습니다.
- `src/`는 계층마다 폴더 하나입니다: `core`, `cli`, `net`(`net/ipt/` 포함), `radio`, `mode`, `monitor`, `cmd`, 그리고 `main.c`입니다. 파일은 자기 계층과 그 아래 계층만, 그 순서대로 포함합니다. 각 폴더에는 공개 헤더가 하나씩 있으며 `src/include/<folder>/<folder>.h`에 있고, 프로토타입은 정의하는 파일 이름을 붙인 라벨 아래 묶입니다. 폴더 자체 파일과 테스트만 쓰는 것은 `src/<folder>/internal.h`에 들어가고 `"internal.h"`로 포함됩니다(테스트는 경로로 가져갑니다). 나머지는 모두 `static`입니다. 파일은 200에서 800줄입니다. 새 함수는 그 서브시스템의 파일로 들어가고, 새 파일은 새 서브시스템이나 800줄을 넘긴 파일을 위한 것입니다. `net/ipt/`는 musl에서 커널의 netfilter 헤더와 libc의 네트워킹 헤더가 한 파일에서 만날 수 없기 때문에 자체 헤더를 가집니다. Droidspaces와 USB-Arsenal에서 옮긴 코드는 이 규칙을 따르고, vendored 코드는 `externals/`에 손대지 않고 남습니다.
- 로깅은 `vap_log`, `vap_warn`, `vap_error`(`core/io`)입니다. 이들은 stdout에 출력하고, 앱은 그것을 읽는 `[WARN]`과 `[ERROR]` 태그와 함께 로그 시트로 흘려보내며, 타임스탬프가 붙은 사본을 `logs/ap.log`에 덧붙입니다. 앱은 그 파일을 읽지 않습니다. 맨 `printf`는 로그에서 보이지 않고, stderr로 나간 것은 사라집니다.
- 프로그램은 `core/proc`(`run_quiet`, `run_capture`, `run_shown`, `run_logged`, `run_lines`)로 실행하며, `popen`이나 `system`은 쓰지 않습니다. Android에는 `/bin/sh`가 없고, 각 자식은 stdin으로 `/dev/null`과 깨끗한 시그널 마스크를 받습니다. pid 파일은 그곳의 `pidfile_*` 함수를 거치며, 이 함수들은 기대한 프로그램을 실행 중인 프로세스에만 시그널을 보냅니다.
- 설정은 `cli/options.c`의 `vap_keys` 표 하나에 있습니다. 플래그 문자, `ap.conf` 키, 기본값, 사용법 문구입니다. 새 설정은 `struct vap_opts`의 필드이자 그 표의 행이고, `APConfig`의 필드입니다. 파싱과 사용법 문구가 뒤따릅니다. 모든 값은 줄바꿈을 거부하는 `opt_set`을 거칩니다. `ap.conf`는 앱의 것입니다. `APManager.saveConfig`만 쓰고, `APConfig.isValid()`를 통과한 설정만 씁니다. 엔진은 그것을 읽기만 합니다.
- 커널과는 `net/`(rtnetlink: 주소와 이웃을 다루는 `link`, 정책 규칙과 netd의 표를 다루는 `route`, `net/ipt/`의 `ip_tables` 엔진: `ipt`, `ipt_rules`)과 `radio/`(`driver`: nl80211과 wpa 제어 소켓)를 통해 대화합니다. 어떤 모듈도 `ip`, `iw`, `hostapd_cli`를 실행하지 않으며, `iptables`는 TTL 보정(`net/ipt/ipt_rules`의 `ttl_*`)에만 실행됩니다.
- `upstream_resolve`(`net/route`)는 업스트림 표를 해석하는 유일한 곳입니다. `radio/channel`은 `ensure_5g_ir`을 포함해 대역과 채널 정책이 사는 유일한 곳입니다: 스테이션의 채널을 따르기, DFS 대체, NO-IR 국가 보정. 다른 어디에서도 채널 번호를 비교하지 않습니다. `ipv6_decide`(`mode/ipv6`)는 클라이언트의 IPv6를 고르는 유일한 곳이며, 시작과 모니터가 모두 그것을 묻습니다.
- `Constants`는 앱 쪽의 모든 경로와 환경설정 키를 가지며, 엔진의 것(`Constants.ENGINE`)도 포함합니다. `VirtualAPInstaller`가 바이너리를 배포하며, 다른 곳에 중복되지 않습니다.
- Kotlin에 텍스트를 두지 않습니다. 앱이 보여 주는 모든 문자열은 `res/values/strings.xml`이나 `plurals.xml`에 있고, `stringResource`로, 컴포지션 밖에서는 `getString`으로 읽으므로 Weblate가 번역할 수 있습니다. 규칙은 [CONTRIBUTING.ko.md](./CONTRIBUTING.ko.md)의 Translations에 있습니다.

## 강한 제약

- 앱은 `minSdk = 26`을 대상으로 합니다. API가 있다고 가정하기 전에 Android 8 동작에서 테스트합니다.
- 엔진은 musl 위의 C11(`gnu11`)이며 단일 스레드입니다. `setns`는 호출한 스레드만 옮기고, netlink 소켓은 열었던 네임스페이스에 남습니다. Makefile의 gcc 설정 아래에서 경고 없이 빌드되며, CI는 경고를 오류로 바꿉니다.
- 바이너리는 `externals/` 서브모듈에서 빌드한 완전 정적 musl 빌드이며, 두 아키텍처 모두 그렇습니다. 동적 링크 바이너리도, 다른 곳에서 가져온 미리 빌드된 것도 절대 아닙니다. 새 도구는 새 서브모듈과 `scripts/build-in-container.sh`의 블록입니다. `virtualap`은 이 저장소 자체의 `src/`에서 musl-cross gcc와 루트 `Makefile`로 빌드하는 유일한 바이너리입니다.
- 설치된 의존성이 덮는 일에 새 의존성을 추가하지 않습니다. 앱에는 이미 libsu, Compose, lifecycle, zxing, appcompat이 있습니다.
- 무선 가정의 하한은 삼성의 Broadcom FullMAC 드라이버이고 두 번째는 MediaTek의 gen4m입니다. 각자가 무엇을 갖추지 못했는지는 How-It-Works.md의 "The radio"에 있습니다. 런타임 감지로 게이트되지 않은 특이 동작 수정은 들어가지 않습니다.
