## Ubuntu-22.04-desktop-amd64.iso를 알아보자

### 목표: OS의 이해
작성자: kkongnyang2 작성일: 2025-06-04

---
### 1> 컴퓨터의 구성 요소

CPU (중앙처리장치) : 명령어를 실행함
RAM (메모리) : CPU가 데이터를 읽고 쓰는 작업공간
저장장치 (SSD/HDD) : 모든 프로그램, 파일, 운영체제 등의 영구 저장장치
그 밖에 메인보드, GPU, I/O장치, 네트워크 카드 등

OS(운영체제)는 이 하드웨어들이 효율적으로 협력하도록 관리해주는 “조율자” 역할

---
### 2> amd64가 뭐에요? -CPU의 언어

CPU는 궁극적으로 0과 1만 이해하지만, “이 0과 1이 어떤 의미의 명령어인가?”를 설계 시 정의하기에
x86-64 CPU는 0xB8을 ‘mov eax, imm32’로 해석.
ARMv8-A CPU는 0xB8을 ‘뭔가 이상한 값’으로 해석(=에러).

즉 cpu가 이해할 수 있는 명령어 집합(ISA)을 정의한 cpu 아키텍처 중 한 종류이다.
x86-64 (별칭 amd64)는 인텔/amd 계열 cpu에서 사용
ARMv8-A (별칭 arm64)는 스마트폰 계열 cpu에서 사용

참고로 평소에 소스코드를 cpu의 언어로 번역하기 위해선 컴파일러를 활용한다.

[소스 코드]
    ↓ ISA별 컴파일러
[기계어]
    ↓ ELF 포맷에 담음
[실행 파일]
    ↓ OS가 메모리에 올림
[CPU가 실행]

컴파일러는 소스코드를 해당 ISA에 맞춘 기계어(0과 1)로 번역하고 ELF 포맷에 포장한다.
```
gcc hello.c -o hello
aarch64-linux-gnu-gcc hello.c -o hello
```

---
### 3> .iso가 뭐에요? -OS 이미지

운영체제의 배포를 위해 만들어진 디스크 이미지 파일. 압축된 운영체제의 전체 스냅샷.

.iso 파일: CD/DVD 규격으로 만들어진 파일
-> 이 파일을 usb에 맞게 변환시키기 위해 rufus 등 프로그램 사용. 이미지를 굽는다(flash)고 표현.
.img 파일: usb/sd카드 규격으로 만들어진 파일

이미지를 구워 usb를 만들고 나면, 원하는 기기에 꽂는다.
그대로 usb 베이스로 live 임시부팅을 시킬수도 있고, 설치 마법사를 통해 ssd로 옮기는 작업을 할 수 있다.

iso 상태 -> 압축된 공장형 킷
live 부팅 -> 킷을 풀어서 임시 테스트 작업장
ssd 설치 -> 킷을 완전히 조립하여 완제품

i. iso 파일 내부 구조
```
├── boot/
│   ├── grub/
│   └── ...
├── casper/
│   ├── filesystem.squashfs  ← 압축된 루트 파일 시스템
│   ├── initrd               ← 초기 RAM 디스크 이미지
│   └── vmlinuz              ← 커널 이미지
├── EFI/
│   └── boot/
│       └── bootx64.efi      ← UEFI 부트로더
├── isolinux/ 또는 boot/grub/
│   └── isolinux.cfg         ← BIOS 부트로더 설정
├── install/                 ← 설치 프로그램 관련 파일
├── pool/                    ← 패키지 저장소
├── .disk/                   ← ISO 정보 파일
├── md5sum.txt               ← 파일 무결성 검사 정보
└── README.diskdefines       ← 디스크 정의 정보
ISO 9660, UDF 파일 시스템 구조
```

ii. usb에 굽고 나면
```
├── EFI/
│   └── boot/
│       └── bootx64.efi
├── boot/
│   └── grub/
│       └── grub.cfg
├── casper/
│   ├── filesystem.squashfs
│   ├── initrd
│   └── vmlinuz
├── install/
├── pool/
├── .disk/
├── md5sum.txt
└── README.diskdefines
FAT32으로 포맷
```

iii. 임시 라이브 부팅
```
├── bin/
├── boot/
├── dev/
├── etc/
...
├── sys/
├── tmp/
├── usr/
└── var/
casper/filesystem.squashfs를 ram에 로드하여 임시 실행하는 것.
```

iv. ssd에 설치하면
(아래에 계속)

---
### 4> 설치하고 싶어요

설치란 usb의 내용물을 내 저장장치(ssd)에 옮겨 영구 저장하는 것이다. 이때 설치 마법사를 통해 물리적으로 디스크의 파티션을 나눠 설치한다. 나누는 이유는 맨 앞에 부팅 파티션이 있어야 컴퓨터가 로딩할 수 있기 때문이다.

---
### 5> 파티션을 어떻게 나눠요?

첫번째는 부팅 파티션, 그 뒤로 모든 파일들이 들어있는 루트 파티션!
```
/dev/sda
├── /dev/sda1   EFI 부팅 파티션 (ESP)
│       └── (FAT32, 약 100~500MB)
│       └── /boot/efi에 마운트
│       └── EFI/BOOT/bootx64.efi, grubx64.efi 등 부트로더 파일
│
├── /dev/sda2   리눅스 루트 파티션 (/)
│       └── (ext4, btrfs 등, 주로 수십 GB 이상)
│       └── bin/, boot/, etc/, home/, lib/, usr/, var/ 등 모든 OS 파일
│
└── /dev/sda3   (선택) Swap
        └── (메모리가 부족할 때 RAM처럼 사용)
```

* 참고로 마운트는 논리적으로 파일간의 관계를 잇는 것뿐(디렉토리). /boot/efi의 내용물은 물리적으로는 첫번째 파티션에 저장되어 있다.


듀얼부팅시 부팅 파티션은 공유됨
/dev/nvme0n1  
├─nvme0n1p1   EFI System Partition (FAT32)      300MB     → /boot/efi
├─nvme0n1p2   Microsoft Reserved (MSR)          16MB
├─nvme0n1p3   Windows C: (NTFS)                 100GB     → C:
├─nvme0n1p4   Ubuntu Root (ext4)                100GB      → /
└─nvme0n1p5   Ubuntu Swap                       4GB

---
### 6> 설치한 os는 매번 어떻게 부팅돼요?

i. 전원이 켜지면 cpu는 0번 줄을 읽어옴(롬이나 플래시). 그게 바로 bios/uefi 부팅 펌웨어
ii. bios/uefi 펌웨어는 하드웨어를 초기화하고, 어느 저장장치를 로딩할지 결정함.(f10 화면)
iii. 고른 저장장치의 첫 파티션에서 부트로더(grub).efi가 실행되면 /boot/grub/grub.cfg를 참고해 어떤 커널을 로드할지 결정함.(grub 화면)
iv. 그리고 /boot/vmlinuz-* 커널 이미지를 메모리에 올려 하드웨어를 초기화하고 os 뼈대를 생성, initrd 초기 램 디스크를 올려 임시 루트 역할 및 드라이브, 마운트 도구, 쉘 스크립트를 담당
v. 그러면 이제 커널이 주도권을 잡고 최초 사용자 공간 프로세스 PID 1을 실행. 그 init system(systemd)는 파일을 마운트하고, 네트워크, 로그인, GUI 서비스를 실행.
vi. 로그인하면 사용자 세션 시작.

---
### 7> 파일들 디렉토리가 너무 많아요

🔧부팅
/boot : 부트로더 관련 파일. vmlinuz, initrd.img, grub 등
/root : 루트 사용자의 홈
/run : 부팅 중 시스템이 사용하는 임시 파일(ram 상, 휘발성)

🧰명령어
/bin : 기본 명령어(ls,cp)
/sbin : 시스템 관리 명령어(reboot,mount)
/lib : 위 명령어들이 사용하는 기본 라이브러리

👤사용자
/usr : 사용자 프로그램. 안에 디렉토리가 많음
/usr/bin : 일반 실행파일 (apt,python)
/usr/sbin : 관리자 실행파일
/usr/lib 
/usr/share : 매뉴얼, 아이콘 등 정적 데이터
/usr/local : 패키지 이외로 직접 설치한 앱
아까랑 비슷. 다만 부팅이나 시스템 전반이 아닌 사용자 관련 소프트웨어라 보면됨

🏠홈
/home : 사용자 디렉토리

⚙️하드웨어
/proc : 커널에서 유저 공간으로 정보 전달. (프로세스, 메모리, cpu 정보)
/sys : 커널의 하드웨어 제어 상태 정보 제공 (장치, 드라이버, 커널 모듈)
/dev : 실제 하드웨어 장치들 파일

📂활동 파일
/etc : 시스템 설정 파일 집합
/var : 로그,캐시,데이터베이스 등 자주 바뀌는 데이터
/opt : 크롬같은 독립 설치 앱
/srv : 서비스 데이터(웹 등)
/mnt : 외부 디스크 마운트
/tmp : 임시파일(재부팅 시 삭제)

---
### 8> 패키지 설치는 어떤 과정을 거쳐요?

i. /etc/apt/sources.list에 저장소 정보 정의.(http://archive.ubuntu.com/ubuntu 등)
ii. apt update를 실행하면 각 저장소의 packages.gz파일(패키지명, 버전, 의존성 등의 메타데이터)을 받아와 /var/lib/apt/lists/에 풀어놓음
iii. apt install vim을 실행하면 그 메타데이터를 참고하여 지정된 저장소 서버에서 실제 패키지(.deb 파일)을 다운로드해서 /var/cache/apt/archives/에 저장
iv. dpkg가 .deb 파일을 풀어서 각 파일을 적절한 경로에 배치
* 실행파일: /usr/bin, /usr/sbin
* 설정파일: /etc
* 라이브러리: /usr/lib 또는 /lib
* 문서: /usr/share/doc
* 서비스파일: /lib/systemd/system, /etc/systemd/system
v. .deb 안에 있는 preinst/postinst 스크립트를 실행해 의존성 처리, 설정파일 생성 

---
### 9> 시스템 표준 변화 알아두기

파티션 구조 : MBR -> GPT
펌웨어 구조 : BIOS -> UEFI
파일 시스템 : FAI32 -> exFAT, NTFS, ext4
네트워크 : IPv4 -> IPv6
프로세서 구조 : x86 -> x86-64 -> ARM
운영체제 초기화 : init -> systemd

FAT 구조 : 4KB 블록으로 쪼갬. 그리고 각 블록당 32bit짜리 인덱스를 붙임. 그리고 이제 블록 순서가 담긴 백과사전을 따로 만들어놓음.
그러면 파일의 첫 블록인덱스만 저장해놔도, 파일 전체를 불러올 수 있음

ext4 구조 : 파일 하나당 inode(메타 데이터) 하나. 그 안에 데이터 블럭 포인터, 직접 포인터, 간접 포인터가 있음.
그러면 이제 파일마다 inode 번호만 저장해놔도, 파일 전체를 불러올 수 있음

### 10> 커널은 뭐에요?

OS의 한 부분으로, 하드웨어와 사용자 프로그램 사이를 연결하는 파트 관리자.
모든 자원들을 다룸
프로세스를 관리하고, 메모리 ram을 할당하고, 디스크에 저장된 데이터에 접근하고, 하드웨어와 통신하고, 네트워크를 관리하고, 사용자 프로그램이 기능을 요청할 수 있게 함

참고로 os = 커널 + 사용자 공간 도구 + 서비스