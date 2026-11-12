# VirtualAP

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [Русский](README.ru.md) | [Français](README.fr.md) | [Deutsch](README.de.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

루팅된 Android 휴대폰을 진짜 Wi-Fi 라우터로 바꿉니다.

VirtualAP는 휴대폰의 Wi-Fi 칩에서 제대로 된 액세스 포인트를 실행합니다. 네트워크
이름과 비밀번호, 대역과 채널, 기기가 보게 될 게이트웨이 주소, 그리고 트래픽이 향할
곳(모바일 데이터, Wi-Fi, VPN)을 직접 정할 수 있습니다.

## 할 수 있는 일

- **OpenWrt를 핫스팟의 라우터로 실행합니다.** 방화벽 존, 포트 포워딩, 트래픽 규칙,
  대역폭 제한, LuCI 웹 인터페이스를 모두 휴대폰에서. 아래의 관리 모드를 참고하세요.
- **VPN을 공유합니다.** 핫스팟에 연결된 모든 기기가 휴대폰의 VPN을 거칩니다. TV,
  콘솔, 노트북에 설치할 것은 없습니다.
- **Wi-Fi를 중계합니다.** Wi-Fi 네트워크에 연결된 상태로 그것을 자신의 네트워크로
  다시 브로드캐스트합니다.
- **고정 게이트웨이를 유지합니다.** 게이트웨이 주소가 바뀌지 않으므로 포트 포워딩과
  저장된 SSH 대상이 계속 동작합니다.
- **무선을 조정합니다.** 2.4 또는 5 GHz, 채널과 대역폭, WPA2 또는 WPA3, 네트워크
  이름 숨김.
- **통신사에 테더링을 숨깁니다**: 커널이 지원하는 경우 선택적 TTL 보정으로.

## 관리 모드: OpenWrt가 핫스팟을 운영

이것이 VirtualAP를 특별하게 만듭니다. 핫스팟을
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)에서 실행되는 OpenWrt
컨테이너에 맡기면, OpenWrt가 실제 Wi-Fi 라우터처럼 연결된 모든 기기를 관리합니다.
브라우저에서 선택한 게이트웨이 주소를 열면 LuCI로 들어갑니다.

OpenWrt가 맡으면 다음을 할 수 있습니다.

- 원하는 만큼 많은 존으로 방화벽 구성
- 포트 포워딩과 트래픽 규칙 설정
- 기기별 대역폭과 속도 제한
- 다른 기기에 접근할 수 없는 게스트 네트워크 운영
- 연결된 모든 기기 확인, 고정 주소와 이름 부여
- 네트워크 전체의 DNS 서버 선택, OpenWrt 패키지 피드로 모든 기기에 광고 차단 추가
- 기기에 IPv4와 함께 IPv6 제공
- Droidspaces 컨테이너를 같은 라우터 뒤에 두어 Wi-Fi 기기와 같은 네트워크에 두거나
  완전히 분리
- `opkg`로 OpenWrt의 다른 소프트웨어 설치

OpenWrt의 WAN 쪽은 Droidspaces이며, Droidspaces는 모바일 데이터, Wi-Fi, VPN 터널
사이에서 업링크를 실시간으로 전환합니다. 핫스팟은 이를 그대로 누립니다.

1. Droidspaces에서 `tun0`을 OpenWrt 컨테이너의 첫 업스트림 인터페이스로 지정하고,
   이어서 `wlan0`과 `rmnet0`을 지정합니다. 아직 VPN이 실행되지 않았으므로 `tun0`은
   존재하지 않습니다.
2. VirtualAP를 시작합니다. 핫스팟의 트래픽은 Wi-Fi 또는 모바일 데이터로 나갑니다.
3. VPN 앱을 시작합니다. Droidspaces는 `tun0`이 나타난 것을 보고 OpenWrt를 그쪽으로
   옮기므로, 잠시 뒤 핫스팟의 모든 기기가 VPN 뒤에 놓입니다. 재시작할 것은 없습니다.

OpenWrt는 휴대폰 자체 커널의 컨테이너에서 실행되며, 중간에 가상 머신이 없습니다.

관리 모드 요구 사항:

- root 권한
- 설치되어 동작하는 [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)
- [공식 Droidspaces rootfs 저장소](https://github.com/Droidspaces/Droidspaces-rootfs-builder/releases)의
  OpenWrt 컨테이너를 설치하고 실행 중일 것

## 인터페이스 모드

컨테이너도 웹 인터페이스도 없는 간단한 모드입니다. VirtualAP가 직접 주소를 할당하고,
업스트림이 지원하면 기기에 IPv6를 제공하며, 선택한 인터페이스로 트래픽을 내보내거나
Android가 현재 사용하는 네트워크를 따릅니다. VPN 공유나 Wi-Fi 중계에는 이것으로
충분합니다.

## 요구 사항

- 루팅된 휴대폰
- Android 8.0 이상
- ARM 휴대폰, 64비트 또는 32비트. 하나의 APK로 둘 다 지원합니다.
- 액세스 포인트 자동 시작: Magisk 또는 KernelSU
- 관리 모드: Droidspaces와 그 OpenWrt 컨테이너(위 참고)
- TTL 보정: 이를 지원하는 커널. 앱이 검사하여 없으면 옵션이 회색으로 표시되며,
  핫스팟은 그것 없이도 동일하게 동작합니다.

## 시작하기

1. [Releases](https://github.com/ravindu644/VirtualAP/releases)에서 최신 APK를 내려받아
   설치합니다.
2. 앱을 열고 root를 허용합니다. 선택적 Magisk/KernelSU 부팅 모듈을 포함해 필요한
   모든 설정을 스스로 처리합니다. 플래시하거나 재부팅할 것은 없습니다.
3. 네트워크 이름과 비밀번호를 입력합니다.
4. 업스트림을 선택합니다. **인터페이스**(또는 자동), 관리 모드라면 OpenWrt
   **컨테이너**입니다.
5. 핫스팟을 시작하고 기기를 연결합니다.

설정에서 **부팅 시 액세스 포인트 시작**을 활성화하면 재부팅 후에도 자동으로
되살아납니다.

관리 모드에서는 연결된 기기에서 `http://<your gateway address>`를 열어 LuCI에
접속합니다.

## 더 보기

- [동작 방식](Documentation/How-It-Works.md): 라우팅, IPv6, 브리지, 무선, 휴대폰의
  파일, 바이너리와 APK의 빌드 방법
- [기여하기](CONTRIBUTING.md)

## 라이선스

GNU General Public License v3.0. [LICENSE](LICENSE)를 참고하세요.

VirtualAP는 [Ubuntu-Chroot](https://github.com/ravindu644/Ubuntu-Chroot)의
[`start-hotspot`](https://github.com/ravindu644/Ubuntu-Chroot/blob/main/tools/start-hotspot)
스크립트에서 출발했으며, 앱은 AI 어시스턴트의 도움을 받아 작성되었습니다.
