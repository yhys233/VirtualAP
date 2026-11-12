# DESIGN.md

[English](DESIGN.md) | [简体中文](DESIGN.zh-CN.md) | [繁體中文](DESIGN.zh-TW.md) | [Русский](DESIGN.ru.md) | [Français](DESIGN.fr.md) | [Deutsch](DESIGN.de.md) | [日本語](DESIGN.ja.md) | [한국어](DESIGN.ko.md)

VirtualAP Android 앱의 시각 언어로, 그것을 구현하는 컴포넌트들과 함께 가져온 Droidspaces 디자인 언어입니다. 백엔드는 UI가 없는 셸 스크립트이므로 이 내용은 `backend/`에 해당하지 않습니다.

컴포넌트가 먼저, 값이 그다음입니다. [CONTRIBUTING.ko.md](./CONTRIBUTING.ko.md)의 재사용 목록이 존재하는 것들을 나열하고, 이 파일은 그 목록에 필요한 것을 덮는 항목이 없을 때를 위한 것입니다. 16진수 색상, sp 크기, 눈대중으로 고른 반경을 하드코딩하지 마세요.

## 앱이 어떻게 보이는지, 그 이유

이 앱에서 떠 있는 것은 없습니다. 깊이는 표면 색을 한 단계 올리고 1dp 테두리를 그려 표현합니다. 그림자를 찾고 있다면 이미 디자인에서 벗어난 것입니다. `Surface`가 기초 요소입니다. 버튼은 Material `Button`이 아니라 클릭 가능한 Surface이고, 다이얼로그는 `AlertDialog`가 아니라 `Dialog { Surface { } }`입니다. Material 컴포넌트가 금지된 것은 아니지만, 자기만의 elevation과 반경 기본값을 들고 오므로 정당화가 필요합니다.

색은 파생되며 결코 리터럴이 아닙니다. `ui/theme/Theme.kt`는 세 가지 팔레트 색에서 전체 스킴을 블렌딩하거나 Material You 스킴을 통째로 받습니다. 동적 색상은 기본으로 켜져 있고, 구형 기기와 그것을 끈 사용자를 위한 대체 팔레트가 함께 오며, AMOLED는 모든 표면을 검정으로 무너뜨립니다. 그래서 하드코딩된 `Color(0xFF...)`는 네 경로를 한꺼번에 깨뜨립니다.

테두리와 틴트가 상태를 나르고, 채움은 조용히 남습니다. 실행 중인 액세스 포인트는 초록 카드가 아니라, 틴트된 버튼과 액센트 테두리를 가진 평범한 카드입니다.

## 색상

역할은 `MaterialTheme.colorScheme`에서 옵니다. 스킴 자체는 `ui/theme/Color.kt`의 팔레트에서 `ui/theme/Theme.kt`가 만들며, 이 두 파일만 색을 문자 그대로 적습니다.

| 표면 | 역할 |
| --- | --- |
| 화면 배경 | `background` |
| 배경 위에 놓인 카드 | `surfaceContainer` |
| 다른 표면 안에 중첩된 카드나 필, 그리고 상호작용하는 설정 행 | `surfaceContainerHigh` |
| 다이얼로그와 바텀 시트 외곽 | `surfaceContainer` |

| 텍스트 | 역할 |
| --- | --- |
| 기본 | `onSurface` |
| 보조, 지원, 메타데이터 | 알파 `0.7f`의 `onSurfaceVariant` |
| 3차, 의도적으로 조용함 | 알파 `0.6f`의 `onSurfaceVariant` |
| 비활성 내용 | 알파 `0.38f`의 `onSurface` |
| 비활성 컨테이너 채움 | 알파 `0.12f`의 `onSurface` |

테두리는 항상 1dp입니다. 바뀌는 것은 알파입니다.

| 테두리 | 색상 |
| --- | --- |
| 카드 | `0.35f`의 `outlineVariant` |
| 다이얼로그 외곽 | `0.4f`의 `outlineVariant` |
| 액션 필 래퍼와 기타 중첩 표면 | `0.2f`의 `outlineVariant` |
| 구분선(`HorizontalDivider`) | `0.3f`의 `outlineVariant` |
| 텍스트 필드, 포커스 없음 | `0.5f`의 `outlineVariant` |
| 텍스트 필드, 포커스됨 | `0.8f`의 `primary` |
| 강조색이 적용된 버튼, 해당 액센트와 일치 | `0.2f`의 액센트 역할 |

### 색이 뜻하는 것

액세스 포인트 생명주기는 앱 전체에서 일관되며, 새 상태는 평행한 어휘를 발명하기보다 여기에 합류해야 합니다.

| 상태 | 액센트 |
| --- | --- |
| 실행 중, 정상, 시작 액션 | `primary` |
| 정지, 실패, 파괴적, 정지 액션 | `error` |
| 재시작 액션 | `secondary` |
| 진행 중, 재시작 중, 다운로드 중 | `tertiary` |
| 유휴, 알 수 없음, 보고할 것 없음 | 알파 `0.6f`의 `onSurfaceVariant` |
| 비활성 | 채움 `0.08f`의 `onSurface`, 액센트 `outlineVariant` |

틴트된 버튼은 알파 `0.4f`의 해당 컨테이너 역할에서 채움을 가져옵니다. 예를 들어 시작에는 `primaryContainer.copy(alpha = 0.4f)`, 정지에는 `errorContainer.copy(alpha = 0.4f)`이고, 알파 `0.2f`의 액센트로 1dp 테두리를 두릅니다.

콘솔은 `classifyLine`이 태그에서 읽은 수준에 따라 `[WARN]` 줄을 `tertiary`로, `[ERROR]` 줄을 `error`로 칠합니다. 백엔드는 이스케이프 코드 없이 평문을 내보냅니다.

## 타이포그래피

스케일은 Material 3 자체의 것이며, `ui/theme/Type.kt`의 `Typography()`입니다. 이름 있는 스타일을 쓰고 굵기는 쓰는 자리에서 넘깁니다. 제목은 `FontWeight.Bold`를 준 `titleLarge`, 카드 제목은 SemiBold의 `titleMedium`입니다. 앱 어디에도 `fontSize` 재정의는 없습니다.

UI 텍스트는 `FontFamily.Default`입니다. 기계가 만든 것, 즉 로그 줄, 설정 값, 경로는 `ui/theme/Type.kt`에서 가져온 JetBrains Mono입니다. 지역 글꼴 패밀리를 선언하지 마세요.

| 요소 | 스타일 |
| --- | --- |
| `TopAppBar`의 화면 제목 | `titleLarge`, Bold |
| 다이얼로그 제목 | `titleLarge`, Bold, 왼쪽 정렬 |
| 카드 제목 | `titleMedium`, SemiBold |
| 화면 안의 구획 제목 | `titleSmall`, Bold, `primary` |
| 버튼 레이블 | `labelLarge`, SemiBold |
| 다이얼로그 본문 | `bodyMedium` |
| 도움말 텍스트, 메타데이터, 부제 | `0.7f`의 `onSurfaceVariant`를 쓴 `bodySmall` |
| 상태 필 레이블 | `labelSmall`, Black, `letterSpacing = 0.5.sp`, 대문자 |
| 기계 출력 | JetBrains Mono 패밀리를 쓴 `bodySmall` |

## 간격

스케일은 **4, 8, 12, 16, 20, 24, 32**입니다. 그 외에는 없습니다. 간격이 14를 원하는 것 같으면 12나 16을 원하는 것입니다.

| 간격 | 값 |
| --- | --- |
| 화면 가장자리, 목록과 대시보드 | 가로 16 |
| 화면 가장자리, 마법사와 설정 흐름 | 가로 24 |
| 목록에서 카드 사이 | 16 |
| 카드 안 블록 사이 | 12 |
| 카드 안쪽 패딩 | 16 |
| 다이얼로그 안쪽 패딩 | 24 |
| 레이블에 붙는 아이콘, 칩 사이 | 8 |
| 빽빽한 메타데이터 행 | 4 |

탭 바는 없습니다. 메인 화면은 `Scaffold` 안에서 `PrimaryActionBottomBar` 위에 놓이므로, 바의 측정된 높이가 Scaffold의 안쪽 패딩으로 들어오고 목록은 아무도 여백 숫자를 적지 않고도 그것을 비켜 갑니다. 그대로 유지하세요. Scaffold의 `bottomBar`가 아닌 바에는 리터럴이 필요하고, 리터럴은 흘러갑니다.

### 카드 머리글이 정렬된다

활성 네트워크 카드에는 제목이 없습니다. 첫 행은 왼쪽의 네트워크 이름과 오른쪽의 상태 필이며, 구분선 위에서 최소 높이 `CardHeaderHeight`와 `CardContentPadding` 안에 있고, 둘 다 `ui/component/CardMetrics.kt`에서 옵니다. 높이가 48dp인 것은 그 행에 Wi-Fi QR 아이콘 버튼이 들어가고 48dp가 터치 하한이기 때문입니다.

제목과 필 헤더를 실제로 갖는 카드는 같은 두 값을 쓰므로, 그 구분선과 필이 이 카드와 같은 선에 놓입니다. 이 값들을 바꾸면 앱의 모든 카드 머리글을 옮기는 것이고, 그것이 의도된 방향입니다.

## 모양

| 요소 | 반경 |
| --- | --- |
| 카드 | 20 |
| 다이얼로그 외곽 | 24 |
| 바텀 시트, 위쪽 모서리만 | 24 |
| 전체 너비 기본 버튼 | 20 |
| 드롭다운 메뉴 | 20 |
| 액션 버튼 | 16 |
| 텍스트 필드 | 16 |
| 액션 필 래퍼 | 12 |
| 상태 필, 작은 배지 | 8 |

## 액션 필

서로 관련된 액션의 행은 틴트된 래퍼 안에 있습니다.

```kotlin
Surface(
    shape = RoundedCornerShape(12.dp),                    // wrapper
    color = MaterialTheme.colorScheme.surfaceContainerHigh,
    border = BorderStroke(1.dp, MaterialTheme.colorScheme.outlineVariant.copy(alpha = 0.2f))
) {
    Row(Modifier.padding(4.dp), horizontalArrangement = Arrangement.spacedBy(4.dp)) {
        Surface(
            modifier = Modifier.weight(1f).height(48.dp),
            shape = RoundedCornerShape(16.dp),            // button, larger than the wrapper
            color = accentContainer.copy(alpha = 0.4f),
            border = BorderStroke(1.dp, accent.copy(alpha = 0.2f))
        ) { /* icon + labelLarge SemiBold */ }
    }
}
```

버튼이 자기가 들어 있는 상자보다 더 둥글고, 그것이 디자인입니다. 4dp 안쪽 여백에 더 큰 안쪽 반경이 더해져 버튼이 래퍼 안에 납작하게 끼는 것이 아니라 래퍼 위로 솟아 보입니다. 두 반경을 같게 맞추면 그 효과가 평평해집니다.

메인 화면의 Interface 또는 Container 전환(`ui/screen/MainScreen.kt`의 `SegmentedSelector`)은 이 기하를 선택기로 쓴 것입니다. 선택된 구획이 틴트된 채움과 액센트 테두리를 갖고, 나머지는 투명하게 남습니다. 로그 콘솔 위의 지우기와 복사 행(`LogActionRow`)은 그것이 아닙니다. 래퍼 없이 38dp 버튼 둘이 있는 맨 행입니다.

## 다이얼로그

모든 다이얼로그는 `DsDialog`이고, 이것이 프레임과 레이아웃을 모두 소유합니다. 24dp 여백 안의 전체 너비 `Surface`, 화면 높이에서 그 여백을 뺀 값으로 제한되고, 본문이 스크롤되는 24dp 패딩 열입니다. 제한은 창이 아니라 `LocalConfiguration`에서 오므로, 회전 중에 열린 다이얼로그도 살아 있는 구성을 기준으로 측정합니다. 호출자는 너비도, 자기 패딩도, 자기 스크롤도 설정하지 않으며 `wrapContentHeight()`를 절대 넘기지 않습니다. 다이얼로그별로 다른 것은 `modifier`에 실립니다. 텍스트 필드에는 `imePadding()`, 아니면 `heightIn(max = ...)` 상한입니다.

제목은 왼쪽 정렬한 `titleLarge` Bold이고, 내용의 첫 번째 요소입니다.

## 다이얼로그 액션

About, 즉 정보 페이지는 헤더에서 `DialogCloseButton`으로 닫힙니다. 선택이 있는 다이얼로그인 언어 선택기는 `DialogFooterRow`로 끝납니다. 같은 무게의 48dp 버튼 둘, 반경 16, 조용한 톤의 닫기와 채워진 확인, 말줄임되는 한 줄 레이블입니다. `DsDialog`의 `footer` 슬롯에 들어가고 스크롤되는 본문보다 먼저 측정되므로 가로 화면에서도 눌리지 않습니다. 첫 파괴적 다이얼로그는 Droidspaces의 `error` 확인 버튼을 그대로 가져옵니다.

## 아이콘과 터치 대상

| 용도 | 크기 |
| --- | --- |
| 카드나 목록 행의 선행 아이콘 | 20 |
| 툴바 액션, 버튼 안의 아이콘 | 18 |
| `bodySmall` 텍스트와 같은 줄 | 16 |
| 다이얼로그 헤더 | 24 |
| 설정 히어로 글리프 | 240 히어로 안의 96 |
| 상태 점 | 6 |

탭할 수 있는 것은 무엇이든 최소 **48dp**입니다. 그것은 스타일 선호가 아니라 접근성 하한이며, 이 파일에서 디자인 논리로도 뒤집을 수 없는 유일한 규칙입니다.

## 정렬

행은 `Alignment.CenterVertically`입니다.

카드 제목과 다이얼로그 제목은 언제나 왼쪽 정렬입니다. 제목과 컨트롤을 짝지은 카드 행은 `Arrangement.SpaceBetween`을 씁니다.

가운데 정렬 텍스트는 전체 화면 상태에만 씁니다. 빈 목록, 로딩, 설정 히어로입니다. 카드 안에서 내용이 가운데 정렬되어 있다면 아마 틀린 것입니다.

## 모션과 로딩

스피너는 `LoadingSize`를 준 `LoadingIndicator`에서 옵니다. 진행 표시기에 맨 `.size(n.dp)`를 쓰지 않습니다. 전체 화면 로딩은 `FullScreenLoading`이고, `LoadingSize.Hero`(144)는 설정 흐름의 유일한 히어로 로더입니다. 애니메이션 타이밍은 `AnimationUtils`에서 오고, 히어로 상태 변경은 테마가 `MotionScheme.expressive()`로 설정하는 `MaterialTheme.motionScheme`을 씁니다.

### 설정 흐름

환영 페이저, root 확인, 백엔드 설치 프로그램은 하나의 기하를 공유하고, 그것은 `ui/component/SetupHero.kt`에 있습니다. `SetupPage`는 Scaffold 아래의 골격입니다. 위의 40dp 행(환영 페이저에서는 페이지 카운터, 다른 곳에서는 비어 있음), 내용을 가운데 두거나 창이 너무 짧으면 스크롤하는 프레임 슬롯, 아래의 44dp 행(페이지 점)입니다. 모든 화면이 두 행을 유지하므로 프레임과 버튼이 화면 사이에서 움직이지 않습니다. `SetupFrame`은 프레임입니다. 240dp 히어로 상자, 그다음 고정된 텍스트 블록 안의 제목과 본문입니다. `SetupHero`는 실제로 일을 하는 화면의 히어로로, `primaryContainer`나 `errorContainer` 안의 240dp `MaterialShapes.Cookie12Sided`가 로더나 96dp 글리프를 담고, 상태 사이에서 색과 내용만 바뀝니다. 환영 페이저의 히어로는 대신 자체 `Shape`로, 페이지가 그 위를 미끄러질 때 쿠키, 클로버, 해로 정착하는 부드러운 버스트입니다. `SetupActionBar`는 유일한 버튼입니다. 이것들로 새 설정 화면을 만들면 히어로, 제목, 버튼이 다른 화면과 같은 행에 놓입니다.

## 규칙이 맞지 않을 때

때로는 맞지 않을 것이고, 그때의 답은 조용히 다른 숫자를 쓰는 것이 아닙니다.

PR에서 말하고, 그 자리에 벗어나서 무엇을 얻는지 설명하는 주석을 남기세요. 아래의 결정된 예외들이 그런 방식입니다. 문서화되지 않은 예외는 표류가 되어 다음 기여자가 "고쳐" 버립니다. 문서화된 예외는 결정이고, 살아남습니다.

## 결정된 예외

검토했고, 의도적으로 그대로 두었습니다. 원래 이유가 놓친 이유 없이 이것들을 다시 열지 마세요.

- **QR 판은 `Color.White`입니다.** Wi-Fi QR 시트에서 그러하고, `QrCodeGenerator.encode`는 흰 바탕에 검은 모듈을 그립니다. 스캐너는 명세가 전제하는 대비가 필요하고, 어두운 팔레트 아래의 테마 판은 깨진 코드처럼 읽힙니다.
- **상태 카드는 탭에 반응합니다.** (`ui/screen/MainScreen.kt`의 `ActiveNetworkCard`) 아무것도 열지 않지만, 사용자가 앱이 살아 있는지 확인하려고 건드리는 카드이므로, 손가락 아래에서 죽어 있는 대신 리플을 내고 필드 포커스를 내립니다. 대가는 TalkBack이 그것을 하나의 액션 가능 노드로 읽는다는 것입니다.
- **Wi-Fi QR 시트, 기기 시트, 로그 시트**와 `ui/screen/MainScreen.kt`의 그것들, 그리고 `ui/component/DeviceList.kt`의 기기 목록 시트들은 `Dialog { Surface { } }`가 아니라 Material `ModalBottomSheet`로 남습니다. 모두 아래 가장자리에 속한 끌어서 닫는 내용이고, 로그 시트는 명령이 실행 중일 때 제스처 닫기를 막아 시트가 터치를 먹는 스크림을 남기지 않게 합니다. 그 외곽은 모양 표에 맞춘, 위쪽 모서리 24dp의 `surfaceContainer`입니다.
- **설정 흐름의 버튼**(`ui/component/SetupHero.kt`의 `SetupActionBar`)은 바 표면도 구분선도 없는 완전한 알약으로, 20dp 바 버튼이 아닙니다. 스크롤되는 내용 위가 아니라 히어로 아래 배경에 홀로 놓입니다. 비활성 색상은 활성 색상과 같은 값으로 남습니다. 검사나 설치가 진행 중일 때만 비활성이고, 그 순간을 위해 회색으로 칠하면 깜빡임으로 읽힙니다.
- **0.03과 0.06의 희미한 채움 알파**는 `LogActionRow`에서 공유하지 않고 남습니다. 단독으로는 보이지 않고, 이 둘을 맞추려면 아무도 볼 수 없는 것에 토큰을 발명해야 합니다.
