## Ubuntu-22.04-desktop-amd64에서 keyboard 커스터마이징

### 목표: I/O 장치의 이해
작성자: kkongnyang2 작성일: 2025-06-04

---
### 0> 환경 점검

하드웨어: 삼성 노트북
├─cpu: i5-10th amd64
├─ram: 8GB
└─ssd: 256GB
운영체제: ubuntu-22.04.5-desktop-amd64
외부 저장장치: 없음

---
### 1> 입력기 시스템 어떻게 되어있나요

[1] 하드웨어 | 키보드 자체
 ↓
[2] 커널 계층 | 키 입력 처리 (/dev/input/event로 전달) | libinput
 ↓
[3] 레이아웃 계층 | keycode → keysym 변환 | XKB
 ↓
[4] 입력기 프레임워크 | 문자 조합 | IBus, Fcitx
 ↓
[5] 입력기 엔진 | 각 언어별 로직 | ibus-hangul
 ↓
[6] 디스플레이 서버 | 최종 입출력 전달 백엔드(무대) | Wayland, X11
 ↓
[7] 데스크톱 환경 | 화면 구성 등 사용자 UI(겉모습) | Gnome

* gnome은 마스터 컨트롤러 같은 존재. 각 애들한테 명령을 내림.
* ibus는 그저 입력기 프로그램이기 때문에 gnome의 요청없이 그 상위계층에 입출력을 전달할 수가 없다.

```bash
~$ echo $XDG_SESSION_TYPE         #디스플레이 서버 확인
~$ echo $XMODIFIERS               #프레임워크 확인
~$ ibus list-engine               #엔진 종류 확인
~$ ps aux | grep -E 'ibus|fcitx|uim' | grep -v grep   #입력기 데몬 확인
~$ xev                            #키 매핑 확인
~$ sudo libinput debug-events     #입력 흐름 확인
```
* `ps aux`: 모든 사용자와 프로세스 출력
* `grep`: 찾기
* `-E`: 또는
* `-v`: 제외

---
### 2> ibus-hangul 설치

step 1. 설치
```bash
~$ sudo apt install ibus-hangul
```
step 2. ibus 재시작하기
```bash
~$ ibus restart
~$ ibus-daemon -drx
```
* `-d`: 백그라운드 데몬으로 실행 `-r`: 리셋 `-x`: X 모드로 실행

step 3. ibus-hangul 설치 여부 확인
```bash
~$ dpkg -l | grep ibus-hangul
```
* `dpkg -l`: 현재 설치된 모든 패키지 목록
* `grep`: 검색

step 4. ibus 목록 확인하기:
```bash
~$ ibus list-engine
```
step 5. Gnome 에서 추가
```bash
~$ gnome-control-center keyboard
입력기 전환 키 삭제
input sources에서 기본 영어 키보드 삭제
input sources에서 ibus-hangul 입력기 추가 (안에 영어 한글 다잇음)
```
step 6. ibus 에도 등록

Gnome에 이미 ibus-hangul 엔진을 추가했기에 하지 않아도 작동하지만, 안정성을 위해.
```bash
~$ ibus-setup
입력기 전환 키 삭제
ibus-hangul 추가
한영 토글 키 hangul키로 설정
```

> ⚠️ gnome 영향안받고 ibus만 입력기로 쓸거라면:
```bash
~$ echo 'GTK_IM_MODULE=ibus
~$ QT_IM_MODULE=ibus
~$ XMODIFIERS=@im=ibus' >> ~/.profile
```

---
### 3> 키 매핑 변경

```bash
~$ sudo gedit /usr/share/X11/xkb/keycodes/evdev
```

원본:
```xkb
<RALT> = 108;
<HNGL> = 130;
```

수정:
```xkb
<HNGL> = 108;
```

* 108번 하드웨어 키를 `HNGL`로 지정함으로써, 우측 ALT를 한영키로 대체
* 원본은 전부 before로 표기해두니 필요하면 ctrl+f로 찾아라

---
### 4> z(keycode 52) 저수준 차단하기

기존 키맵 복사:
```bash
~$ sudo cp /usr/share/X11/xkb/symbols/us ~/.XkbSymbols_us
~$ gedit ~/.XkbSymbols_us
```

원본:
```xkb
key <AB03> { [ z, Z ] };;
```

수정:
```xkb
key <AB03> { [ NoSymbol, NoSymbol ] };
```

반영:
```bash
~$ setxkbmap -I$HOME -layout us
```

* `setxkbmap`: X 맵 설정
* `-I$HOME` : 커스텀 심볼 파일을 홈 디렉토리에서 우선 찾게함
* `-layout us` : 레이아웃이 us 파일을 기반