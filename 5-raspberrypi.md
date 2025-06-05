## Raspberrypi 4B에 우분투 서버 환경을 세팅하자

### 목표: 미니 컴퓨터 이해
작성자: kkongnyang2 작성일: 2025-06-05

---
### 1> 환경 점검

하드웨어: raspberrypi4 model B
사용할 저장장치: 32GB microSD카드

> ⚠️ 원래는 usb로 부팅하려 했으나 실패. 그냥 공식대로 sd카드로 하는게 안전해보임
> ⚠️ 모니터 연결을 위해선 microHDMI가 필요하지만 ssh로 대체 가능.

---
### 2> sd 카드 만들기 - 방법1. etcher로 굽기

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
### 3> sd 카드 만들기 - 방법2. raspi-imager로 굽기

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
비밀번호: 1234
무선 LAN SSID: KT_GiGA_1D39
무선 LAN 비밀번호: eec0hc4206
무선 LAN 국가: KR
로케일 설정 지정: Asia/Seoul
키보드 레이아웃: us
ssh 사용: 비밀번호 인증 사용
```

---
### 4> 부팅하기

step 1. 라즈베리파이에 sd를 꽂고 act led가 적어질 때까지 기다리기

step 2. 공유기 들어가서 새로 연결된 단말의 ip 확인
```
http://172.30.1.254로 들어갔더니 라즈베리파이 172.30.1.89 확인
```

step 3. 노트북에서 라즈베리파이 ssh 접속
```bash
~$ ssh kkongnyang2@172.30.1.89
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1061-raspi aarch64)
```

step 4. 패키지 최신상태 확인
```bash
~$ sudo apt update && sudo apt upgrade -y
```
* &&는 명령어 연속. -y는 자동 yes

step 5. eeprom 최신상태 확인
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
~$ sudo reboot
```