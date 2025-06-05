## Ubuntu 22.04.5에서 웹서버를 구축해보자.

### 목표: 네트워크의 이해
작성자: kkongnyang2 작성일: 2025-06-05

---
### 1> 서버란?

"내 컴퓨터가 다른 기기의 요청을 받아 응답할 수 있게 만든다"
`http://컴퓨터주소`를 입력하면 내 컴퓨터가 웹사이트, 파일, 사진을 보내주는 역할.

그럼 컴퓨터 주소가 뭔지를 설명해야한다.

네트워크가 크게 로컬 네트워크와 외부 네트워크로 나뉘는 건 알 것이다.
로컬 네트워크는 한 공유기 내에 기기들이 wi-fi(무선 랜선)로 서로 연결되어 로컬 ip주소로 서로를 찾는 것이다.
이때 공유기는 라우터 기능을 겸용하여 외부에 집 안내원 역할도 해준다. 외부 공인 ip주소(집주소)를 바깥에 내세우는 것이다.
쉽게 말해 외부 공인 ip주소 = 집주소, 로컬 ip주소 = 안방, mac 주소 = 김철수

내 컴퓨터로 만든 서버 주소 = 내 컴퓨터 로컬 ip주소 지만, 외부에서는 우리들 방까지 알 수는 없다.
따라서 외부에서 `http://공유기 외부 공인 ip` 를 치고 들어오면 내 로컬 주소로 들어올 수 있도록 누가 안내해줘야 한다. 그게 포트포워딩이다. 

접속 과정
1. 누군가 공유기 주소를 입력
2. 라우터(공유기)가 포트포워딩으로 해당 요청을 내 컴퓨터로 전달
3. 내 컴퓨터가 응답하여 파일 등을 반환 


---
### 2> 환경 점검

오늘 만들 구성은 다음과 같다

* 하드웨어: raspberrypi4 model B
* 운영체제 : Ubuntu 22.04.5 server
* 서버 프로그램 : Apache(웹서버)
* 네트워크 설정 : IP주소 고정, 포트포워딩, DDNS 설정
* 보안 설정 : 인증서

---
### 3> ip 주소 확인

step 1. 내 기기 ip 주소 확인
```bash
~$ ip a
```
결과: 
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 0c:7a:15:96:bb:e9 brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
    inet 172.30.1.54/24 brd 172.30.1.255 scope global dynamic noprefixroute wlo1
       valid_lft 2048sec preferred_lft 2048sec
    inet6 fe80::699e:5519:cade:8599/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
* `lo` : 자기 자신과 통신하는 루프백 인터페이스(네트워크 카드가 없는 가상 네트워크)
* `scope host` : 이 주소는 자기 자신에게만
* `wlo1` : 실제 무선 네트워크 인터페이스(wifi)
* `BROADCAST`: 브로드캐스트 기능 사용 가능 (네트워크 전체에 신호 보내기)
* `MULTICAST`: 멀티캐스트 기능 가능 (특정 그룹에만 데이터 보내기)
* `UP`: 장치가 활성화됨
* `LOWER_UP`: 연결도 됨 (Wi-Fi가 연결된 상태)
* `mtu 1500`: 한 번에 보낼 수 있는 최대 패킷 크기 (보통 1500 byte)
* `link/ether`: 이더넷 방식의 링크계층(wifi든 유선이든 lan 방식이면) 
* `0c:7a:15:96:bb:e9` : 내 네트워크 카드의 mac주소(이름표)
* `altname` : 자세한 이름 wlp0s20f3
* `inet` : 내 기기의 IPv4 주소는 172.30.1.54(논리적 로컬주소)
* `/24` : 서브넷마스크(ip 대역폭의 범위). 앞 24비트가 네트워크 부분, 나머지 8비트는 호스트 부분. 따라서 172.30.1.0~172.30.1.255가 같은 네트워크.
* `brd` : 브로드캐스트 IPv4 주소는 172.30.1.255
* `scope gloabl` : 이 주소는 외부에서 접근 가능
* `dynamic` : DHCP로 자동 할당됨. 즉 부팅할때마다 바뀜
* `noprefixroute` : 고급 라우팅 설정이 비활성화
* `valid_lft` : 이 ip가 유효한 기간
* `preferred_lft` : 이 ip가 우선적으로 사용되는 기간
* `inet6` : IPv6주소

step 2. 공유기 ip주소 확인
```bash
~$ ip route
```
```
default via 172.30.1.254 dev wlo1 proto dhcp metric 600 
169.254.0.0/16 dev wlo1 scope link metric 1000 
172.30.1.0/24 dev wlo1 proto kernel scope link src 172.30.1.210 metric 600
```
* 172.30.1.254로 확인

---
### 4> ip 주소 고정 - server 버전

step 1. 부팅 네트워크 파일 수정
```bash
~$ ls -l /boot/firmware/network-config     #파일위치 확인
~$ sudo nano /boot/firmware/network-config
version: 2
wifis:
  renderer: networkd
  wlan0:
    dhcp4: false
    addresses:
      - 172.30.1.220/24
    gateway4: 172.30.1.254
    nameservers:
      addresses:
        - 8.8.8.8
        - 1.1.1.1
    optional: true
    access-points:
      "KT_GiGA_1D39":
        password: "eec0hc4206"
```

step 2. netplan 설정파일 수정
```bash
~$ ls /etc/netplan/             #파일이름 확인
50-cloud-init.yaml
~$ sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak      #백업파일생성
~$ sudo nano /etc/netplan/50-cloud-init.yaml
```
원본:
```
network:
    version: 2
    wifis:
        renderer: networkd
        wlan0:
            access-points:
                KT_GiGA_1D39:
                    password: 4cfa7026f817ab13ef66eb1e898d54830629ad2b173447184>
            dhcp4: true
            optional: true
```
수정:
```
network:
    version: 2
    wifis:
        renderer: networkd
        wlan0:
            access-points:
                "KT_GiGA_1D39":
                    password: "eec0hc4206"
            dhcp4: false
            addresses:
              - 172.30.1.220/24
            gateway4: 172.30.1.254
            nameservers:
              addresses:
                - 8.8.8.8
                - 1.1.1.1
            optional: true
```
* dhcp4 할당을 없애고, 주소를 172.30.1.220으로 고정
* gateway4는 공유기 주소 입력란. 우리는 172.30.1.254임
* 8.8.8.8은 구글에서 제공하는 DNS 서버

step 3. 적용
```
~$ sudo netplan apply
```

step 4. ip 주소 변화로 ssh 끊겼으니 재접속
```
~$ ssh kkongnyang2@172.30.1.220
```

---
### 5> ip 주소 고정 - desktop 버전

*ubuntu desktop에서는 netplan보다 networkmanager우선

step 1. networkmanager 설정
```bash
~$ ls ~$ nm-connection-editor
KT_GiGA_1D39 - IPv4 settings 들어가서
dhcp가 아닌 manual로 바꾸고
address: 170.30.1.210, netmask: 24, gateway: 172.30.1.254
DNS servers: 8.8.8.8, 1.1.1.1 입력
```

step 2. 저장 후 wifi 껐다 키기


> ⚠️ 안될때 테스트:
```bash
ping 172.30.1.254     #안되면 로컬 공유기 통신 문제
ping 8.8.8.8          #안되면 게이트웨이 문제
ping google.com       #안되면 DNS 문제
```

여기까지 하면 내 기기의 ip주소를 하나로 고정한거임!

---
### 6> Apache2 웹 서버 설치

Apache2는 가장 널리 쓰이는 웹 서버로, 브라우저에서 http://내IP로 접속하면 html페이지를 보여줌

```bash
~$ sudo apt update && sudo apt install apache2  #2.4.52-1ubuntu4.14 버전
```

확인:
```
로컬 네트워크로 http://172.30.1.220 들어가보기
```

---
### 7> 포트포워딩

외부에서 접속 가능하도록.
외부 공인 ip인 125.154.102.145 를 치고 들어오면 공유기(안내원)가 포트포워딩 규칙에 의해 172.30.1.220 서버로 전달되도록.

```
http://172.30.1.254 로 접속(공유기 페이지)
외부 포트 80과 125.154.102.145 -> 내부 포트 80과 172.30.1.220로 대응. 프로토콜은 TCP.
외부 포트 443과 125.154.102.145 -> 내부 포트 443과 172.30.1.220로 대응. 프로토콜은 TCP.
```
확인:
```
외부 네트워크로 http://125.154.102.145 들어가보기
```

---
### 8> 도메인 및 DDNS

step 1. No-ip 사이트 가입
```
이메일: i.kkongnyang2@gmail.com
비번: naciwotfhr@8104
```

step 2. 도메인 생성
```
도메인 http://i-kkongnyang2.myddns.me와 외부 공인 ip(현재는 125.154.102.145) 연결.
```

step 3. DDNS 클라이언트 설치
```bash
~$ wget https://www.noip.com/client/linux/noip-duc-linux.tar.gz
~$ tar xzf noip-duc-linux.tar.gz
~$ cd noip-2.1.9-1
~/noip-2.1.9-1$ sudo make
~/noip-2.1.9-1$ sudo make install
가입한 이메일 입력
가입한 비번 입력
도메인 선택
갱신 시간 설정(기본 30분)
갱신되면 뭐 할거니? y
그 프로그램 이름 입력 안함
이 설정파일들은 /usr/local/etc/no-ip2.conf 에 저장
```

step 4. DDNS 클라이언트 실행
```bash
~$ sudo /usr/local/bin/noip2            #실행
~$ sudo /usr/local/bin/noip2 -S         #실행 확인
```
* 부팅마다 자동으로 noip를 실행해주는 파일은 만들까 말까 고민중이다.

---
### 9> 테스트

      테스트 환경           로컬(172.30.1.220)    외부IP(125.154.102.145)    도메인
KT_GiGA_1D39(동일 와이파이)          O                     X                X            #NAT loopback이 없어서 불가능
KT_GiGA_5G_1D39(다른 와이파이)       O                     O                O
LTE/5G 데이터                       X                     O                O           #인터넷에선 로컬 ip주소를 쳤을때 안내해줄 안내원이 없음

---
### 10> https로 리디렉션

```bash
~$ sudo apt install certbot python3-certbot-apache
~$ sudo certbot --apache
안내받을 이메일 i.kkongnyang2@gmail.com
약관동의 y
광고수신동의 N
도메인 입력 i-kkongnyang2.myddns.me
```

### 11> html 파일 업로드

▶️ Apache 기본 웹페이지 수정해보기

    → HTML 파일을 직접 바꾸고, 내 홈페이지를 직접 구성

step 1. 컴퓨터에 있는 파일 html로 번역
```
~$ pandoc -s ~/server-build/index.md -o ~/Downloads/index.html
```
step 2. 라즈베리파이에 전송(복사)
```bash
~$ scp ~/Downloads/index.html kkongnyang2@172.30.1.220:/home/kkongnyang2/
```
* 루트 경로로는 바로 못가기 때문에 한번 홈을 거침

step 3. 기존 default html 파일 삭제
```bash
~$ ssh kkongnyang2@172.30.1.220
kongnyang2@raspberrypi:~$ sudo mv /var/www/html/index.html /var/www/html/index.html.bak
```

step 4. html 파일 배치
```bash
kkongnyang2@raspberrypi:~$ sudo mv ~/index.html /var/www/html/
```