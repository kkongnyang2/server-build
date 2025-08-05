## 파이에 우분투를 설치하자

작성자: kkongnyang2 작성일: 2025-06-09

---
### 환경 점검

하드웨어: raspberrypi4 model B

> ⚠️ 원래는 usb로 부팅하려 했으나 실패. 그냥 공식대로 sd카드로 하는게 안전해보임
> ⚠️ 모니터 연결을 위해선 microHDMI가 필요하지만 ssh로 대체 가능.

---
### sd 카드 만들기

ubuntu-22.04.5-preinstalled-server-arm64+raspi.img.xz 다운받기
* preintalled: 얘는 따로 설치 없이 이미 그 자체를 저장장치로 씀.

rpi-imgaer 1.9.3 다운로드
```
$ sudo snap install rpi-imager
```
rpi-imager에서 굽기 전 사용자 커스터마이징
```
hostname: raspberrypi.local
사용자 이름: kkongnyang2
비밀번호: U2FsdGVkX19dkJUxkq0yzdg74lVWKoQtbJma/YUyJCs=(업로드용)
무선 LAN SSID: KT_GiGA_1D39
무선 LAN 비밀번호: eec0hc4206
무선 LAN 국가: KR
로케일 설정 지정: Asia/Seoul
키보드 레이아웃: us
ssh 사용: 비밀번호로
```

---
### 부팅하기

라즈베리파이에 sd를 꽂고 act led가 적어질 때까지 기다리기

공유기 들어가서 새로 연결된 단말의 ip 확인
```
http://172.30.1.254로 들어갔더니 라즈베리파이 172.30.1.89 확인
```

노트북에서 라즈베리파이 ssh 접속
```
$ ssh kkongnyang2@172.30.1.89
The authenticity of host '172.30.1.89 (172.30.1.89)' can't be established.
ED25519 key fingerprint is SHA256:B5ycXkgh3UjpXdAqAmERIIt83MZghSEiIo+p+tj3fvo.    #처음에 ssh키 생성
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?  yes
Warning: Permanently added '172.30.1.89' (ED25519) to the list of known hosts.
kkongnyang2@172.30.1.89's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1061-raspi aarch64)
$ exit                       #ssh 종료 명령어
```

> ⚠️ 기존 known_hosts를 지워야 한다면
```
~$ ssh-keygen -R 172.30.1.89
```

패키지 최신상태 확인
```
$ sudo apt update && sudo apt upgrade -y
커널 새버전 업데이트 됐는데 알아서 재부팅하라는 안내문 -> 확인
서비스들 지금 재시작할거니? -> tab으로 cancel 선택(어차피 재부팅할거니)
```
* &&는 명령어 연속. -y는 자동 yes

재부팅
```
$ sudo poweroff          # 전원끄기
act led 반응 없고 빨간불만 남으면 전원 뽑기.
그리고 전원 꼽으면 다시 부팅
```

eeprom 최신상태 확인
```
$ sudo rpi-eeprom-update
BOOTLOADER: up to date
   CURRENT: Tue Jan 25 14:30:41 UTC 2022 (1643121041)
    LATEST: Tue Jan 25 14:30:41 UTC 2022 (1643121041)
   RELEASE: default (/lib/firmware/raspberrypi/bootloader/default)
            Use raspi-config to change the release.

  VL805_FW: Dedicated VL805 EEPROM
     VL805: up to date
   CURRENT: 000138a1
    LATEST: 000138a1
```
* 이미 최신 버전이니 건드릴 필요x

---
### 부팅 설정파일 만들어지는 과정

step 0. SD 카드 제작:	pi OS 이미지를 굽는 시점에 /boot/firmware/network-config, user-data, meta-data 등의 시드(seed) 파일을 넣어 둔다.
step 1. init-local: cloud-init-local.service가 파티션 확장, seed 파일 수집.
step 2. init: cloud-init.service가 시드 설정으로 /etc/netplan/50-cloud-init.yaml•/etc/hostname 등을 작성.
step 3. modules: cloud-config.service, cloud-final.service가 유저 추가, SSH 키 배포, 팩키지 설치 “run-once” 모듈 실행.
step 4. 정상 부팅: 이제부터는 일반 부팅 루틴. systemd-networkd, sshd 등이 작동.

/var/lib/cloud/instance/
├─ cloud-config.txt
├─ datasource.json
└─ boot-finished -> 이게 있으면 첫 부팅 아닌걸로 인지.
