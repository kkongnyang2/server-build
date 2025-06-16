## Ubuntu-22.04-server-arm64.img를 설치하자

### 목표: 미니 컴퓨터 이해
작성자: kkongnyang2 작성일: 2025-06-05

---
### 0> 환경 점검

하드웨어: raspberrypi4 model B
├─cpu: arm cortex A72
├─ram: 4GB
└─sd카드: 32GB micro
운영체제: 설치 예정
외부 저장장치: 없음

> ⚠️ 원래는 usb로 부팅하려 했으나 실패. 그냥 공식대로 sd카드로 하는게 안전해보임
> ⚠️ 모니터 연결을 위해선 microHDMI가 필요하지만 ssh로 대체 가능.

---
### 1> sd 카드 만들기 - 방법1. etcher로 굽기(권장x)

step 1. ubuntu-22.04.5-preinstalled-server-arm64+raspi.img.xz 다운받기
* preintalled: 얘는 따로 설치 없이 이미 그 자체를 저장장치로 씀.

step 2. balena-etcher 2.1.2 다운로드
```bash
인터넷에서 balena-etcher.deb 파일 다운
~$ sudo dpkg -i ~/Downloads/balena-etcher_2.1.2_amd64.deb
```
step 3. balenaEtcher로 해당 이미지를 sd 카드에 굽기

step 4. 부팅시 ssh를 위해 system-boot에서 cloud-init 미리 설정해주기

```bash
~$ touch /media/$USER/system-boot/ssh       #ssh 빈파일 만들기
~$ nano /media/$USER/system-boot/network-config #네트워크 설정
wifis:
  wlan0:
    dhcp4: true
    optional: true
    access-points:
      "KT_GiGA_1D39":
        password: "eec0hc4206"

    regulatory-domain: KR
~$ nano /media/$USER/system-boot/cloud-config   #사용자 설정
hostname: raspberrypi
ssh_pwauth: true
disable_root: false
chpasswd:
  list: |
    ubuntu:ubuntu
  expire: false
```
* `hostname`: 네트워크에서 표시될 이름
* `ssh_pwauth`: true: SSH 비밀번호 로그인 허용
* `chpasswd`: 초기 사용자 계정(ubuntu) 비밀번호 설정

> ⚠️ 만약 라즈비언os라면 wpa_supplicant.conf를 만들어주기

---
### 2> sd 카드 만들기 - 방법2. raspi-imager로 굽기

step 1. ubuntu-22.04.5-preinstalled-server-arm64+raspi.img.xz 다운받기
* preintalled: 얘는 따로 설치 없이 이미 그 자체를 저장장치로 씀.

step 2. rpi-imgaer 1.9.3 다운로드
```bash
~$ sudo snap install rpi-imager
```
step 3. rpi-imager에서 굽기 전 사용자 커스터마이징
```
hostname: raspberrypi.local
사용자 이름: kkongnyang2
비밀번호: 4108
무선 LAN SSID: KT_GiGA_1D39
무선 LAN 비밀번호: eec0hc4206
무선 LAN 국가: KR
로케일 설정 지정: Asia/Seoul
키보드 레이아웃: us
ssh 사용: 비밀번호로
```

---
### 3> 부팅하기

step 1. 라즈베리파이에 sd를 꽂고 act led가 적어질 때까지 기다리기

step 2. 공유기 들어가서 새로 연결된 단말의 ip 확인
```
http://172.30.1.254로 들어갔더니 라즈베리파이 172.30.1.89 확인
```

step 3. 노트북에서 라즈베리파이 ssh 접속
```bash
~$ ssh kkongnyang2@172.30.1.89
The authenticity of host '172.30.1.89 (172.30.1.89)' can't be established.
ED25519 key fingerprint is SHA256:B5ycXkgh3UjpXdAqAmERIIt83MZghSEiIo+p+tj3fvo.    #처음에 ssh키 생성
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?  yes
Warning: Permanently added '172.30.1.89' (ED25519) to the list of known hosts.
kkongnyang2@172.30.1.89's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1061-raspi aarch64)
~$ exit                       #ssh 종료 명령어
```

> ⚠️ 기존 known_hosts를 지워야 한다면
```
~$ ssh-keygen -R 172.30.1.89
```

step 4. 패키지 최신상태 확인
```bash
~$ sudo apt update && sudo apt upgrade -y
커널 새버전 업데이트 됐는데 알아서 재부팅하라는 안내문 -> 확인
서비스들 지금 재시작할거니? -> tab으로 cancel 선택(어차피 재부팅할거니)
```
* &&는 명령어 연속. -y는 자동 yes

step 5. 재부팅
```bash
~$ sudo poweroff          #전원끄기
act led 반응 없고 빨간불만 남으면 전원 뽑기.
그리고 전원 꼽으면 다시 부팅
```

step 6. eeprom 최신상태 확인
```bash
~$ sudo rpi-eeprom-update
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
### 4> 부팅 설정파일 만들어지는 과정

step 0. SD 카드 제작:	pi OS 이미지를 굽는 시점에 /boot/firmware/network-config, user-data, meta-data 등의 시드(seed) 파일을 넣어 둔다.
step 1. init-local: cloud-init-local.service가 파티션 확장, seed 파일 수집.
step 2. init: cloud-init.service가 시드 설정으로 /etc/netplan/50-cloud-init.yaml•/etc/hostname 등을 작성.
step 3. modules: cloud-config.service, cloud-final.service가 유저 추가, SSH 키 배포, 팩키지 설치 “run-once” 모듈 실행.
step 4. 정상 부팅: 이제부터는 일반 부팅 루틴. systemd-networkd, sshd 등이 작동.

/var/lib/cloud/instance/
├─ cloud-config.txt
├─ datasource.json
└─ boot-finished -> 이게 있으면 첫 부팅 아닌걸로 인지.
