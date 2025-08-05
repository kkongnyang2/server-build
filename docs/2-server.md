## 웹서버를 구축해보자.

작성자: kkongnyang2 작성일: 2025-06-09

---
### 서버란?

"내 컴퓨터가 다른 기기의 요청을 받아 응답할 수 있게 만든다"
`http://컴퓨터주소`를 입력하면 내 컴퓨터가 웹사이트, 파일, 사진을 보내주는 역할.

한 공유기 내에 기기들이 wi-fi(무선 랜선)로 서로 연결되어 컴퓨터 주소(로컬 ip)로 서로를 찾는다.
하지만 외부에서까지 내 컴퓨터 주소를 알 방법은 없다. 이때 공유기가 라우터 기능을 겸용하여 외부에 집 안내원 역할을 해주는 것이다.
외부에서 `http://공유기 외부 공인 ip:포트` 를 치고 들어오면 내 로컬 주소로 들어올 수 있도록 누가 안내해줘야 한다. 그게 포트포워딩이다. 
외부 공인 주소 = 집주소, 컴퓨터 주소 = 방주소, mac 주소 = 이름

접속 과정
1. 누군가 공유기 주소:포트를 입력
2. 라우터(공유기)가 포트포워딩으로 해당 요청을 내 컴퓨터로 전달
3. 내 컴퓨터가 응답하여 파일 등을 반환 

---
### 전송 원리

“TCP 소켓(IP:Port) 위에서 SSH/HTTP/HTTPS 같은 응용 프로토콜이 자기 규칙대로 대화한다.”

[응용 계층]  SSH   HTTP   HTTPS   …
──────────────────────────────────
[전송 계층]  TCP (or UDP)
──────────────────────────────────
[인터넷 계층]  IP
──────────────────────────────────
[링크 계층]   이더넷, Wi-Fi …

응용 계층 프로토콜
ssh(22) : 원격 터미널, 파일 전송을 암호화해서 제공
http(80) : 웹 페이지 요청 및 응답
https(443) : http에 tls 암호화
같은 tcp 위에서 여러 응용 프로토콜이 포트를 나눠씀

전송 계층 프로토콜
tcp : 데이터가 순서, 무결성을 잃지 않도록 전달
끝점을 (IP, Port) ↔ (IP, Port) 쌍으로 구분

ip 주소 : 어느 컴퓨터냐
포트 번호 : 그 컴퓨터 안의 어느 서비스냐(창구번호)

[클라이언트]           [서버]
     49152              22       ← SSH용 소켓
     50432              80       ← HTTP용 소켓
     49500              443      ← HTTPS용 소켓
   (임의)

8080같은 임의 포트로 웹서버를 띄우려면 http://example.com:8080

---
### 구성 결정

오늘 만들 구성은 다음과 같다

* 서버 프로그램 : Apache(웹서버)
* 네트워크 설정 : IP주소 고정, 포트포워딩, 방화벽 설정, 도메인 생성, DDNS 설정, 인증서

---
### ip 주소 확인

내 기기 ip 주소 확인
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

공유기 ip주소 확인

```
$ ip route
```
```
default via 172.30.1.254 dev wlo1 proto dhcp metric 600 
169.254.0.0/16 dev wlo1 scope link metric 1000 
172.30.1.0/24 dev wlo1 proto kernel scope link src 172.30.1.210 metric 600
```
* 172.30.1.254로 확인

---
### ip 주소 고정 - server 버전

netplan 설정파일 수정
```
$ ls /etc/netplan/             #파일이름 확인
50-cloud-init.yaml
$ sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak      #백업파일생성
$ sudo nano /etc/netplan/50-cloud-init.yaml
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
                    password: 4cfa7026f817ab13ef66eb1e898d54830629ad2b1734471844c7cb8ca7d31fba
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
                KT_GiGA_1D39:
                    password: 4cfa7026f817ab13ef66eb1e898d54830629ad2b1734471844c7cb8ca7d31fba
            dhcp4: false
            addresses:
              - 172.30.1.222/24
            gateway4: 172.30.1.254
            nameservers:
              addresses:
                - 8.8.8.8
                - 1.1.1.1
            optional: true
```
* dhcp4 할당을 없애고, 주소를 172.30.1.222으로 고정
* gateway4는 공유기 주소 입력란. 우리는 172.30.1.254임
* 8.8.8.8은 구글에서 제공하는 DNS 서버

> ⚠️ 만약 부팅 파일이 어떤 일이 있어도 영향을 안주게 하고 싶다면 주석이 시키는대로 (어차피 최초 부팅때 netplan/50-cloud-init.yaml을 생성하고 사라지긴 함)
```
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
```

적용
```
$ sudo netplan generate      #문법 확인
$ sudo netplan apply
```

ip 주소 변화로 ssh 끊겼으니 재접속
```
$ ssh-keygen -R 172.30.1.89      #기존 known_hosts 삭제
$ ssh kkongnyang2@172.30.1.222
The authenticity of host '172.30.1.222 (172.30.1.222)' can't be established.
ED25519 key fingerprint is SHA256:B5ycXkgh3UjpXdAqAmERIIt83MZghSEiIo+p+tj3fvo.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?  yes
Warning: Permanently added '172.30.1.222' (ED25519) to the list of known hosts.
kkongnyang2@172.30.1.222's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1079-raspi aarch64)
```

---
### ip 주소 고정 - desktop 버전

*ubuntu desktop에서는 netplan보다 networkmanager우선

networkmanager 설정
```
$ ls ~$ nm-connection-editor
KT_GiGA_1D39 - IPv4 settings 들어가서
dhcp가 아닌 manual로 바꾸고
address: 170.30.1.210, netmask: 24, gateway: 172.30.1.254
DNS servers: 8.8.8.8, 1.1.1.1 입력
```

저장 후 wifi 껐다 키기

> ⚠️ 안될때 테스트:
```
ping 172.30.1.254     #안되면 로컬 공유기 통신 문제
ping 8.8.8.8          #안되면 게이트웨이 문제
ping google.com       #안되면 DNS 문제
```

여기까지 하면 내 기기의 ip주소를 하나로 고정한거임!

---
### Apache2 웹 서버 설치

Apache2는 가장 널리 쓰이는 웹 서버로, 브라우저에서 http://내IP로 접속하면 html페이지를 보여줌

설치
```
$ sudo apt update && sudo apt install apache2
$ apache2 -version
Server version: Apache/2.4.52 (Ubuntu)
Server built:   2025-04-03T09:05:48
```

확인
```
$ sudo timedatectl set-timezone Asia/Seoul     #서버 시간 업데이트
로컬 네트워크로 http://172.30.1.222 들어가보기
```

소유자 및 퍼미션 확인
```
$ ls -ld /var/www/html
drwxr-xr-x 2 root root 4096 Jun 14 04:55 /var/www/html      #이미 잘 되어있지만 안되어 있으면 아래대로
$ sudo chown root:root /var/www/html
$ sudo find /var/www/html -type d -exec chmod 755 {} \;
$ sudo find /var/www/html -type f -exec chmod 644 {} \;
```
* 소유자는 root:root, 755(디렉토리)임을 확인
* apache2 기본 페이지는 정적 사이트로 html만 업로드 할거기에 변경 필요 없음.
* 만일 쓰기가 필요한 동적 폴더가 있다면 따로 chown(소유자)를 www-data:www-data로 줘야함.

html 파일 업로드
```
$ sudo nano /var/www/html/index.html           #index 파일 수정
$ sudo mkdir /var/www/html/dev-environment     #새 디렉토리 생성
$ sudo touch /var/www/html/dev-environment/1-os.html   #파일 생성
$ sudo nano /var/www/html/dev-environment/1-os.html    #파일 수정
$ ls -ld /var/www/html/dev-environment/1-os.html       #권한 정상인지 확인
-rw-r--r-- 1 root root 7962 Jun 14 05:28 /var/www/html/dev-environment/1-os.html
$ sudo rm -r /var/www/html/dev-environment      # 폴더 삭제
```

---
### 포트포워딩

외부에서 접속 가능하도록.
외부 공인 ip인 U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용) 를 치고 들어오면 공유기(안내원)가 포트포워딩 규칙에 의해 172.30.1.222 서버로 전달되도록.

포트 연결
```
http://172.30.1.254 로 접속(공유기 페이지)
외부 포트 80과 U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용) -> 내부 포트 80과 172.30.1.222로 대응. 프로토콜은 TCP.
외부 포트 443과 U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용) -> 내부 포트 443과 172.30.1.222로 대응. 프로토콜은 TCP.
```

확인
```
외부 네트워크로 http://U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용) 들어가보기
```

방화벽 설정
```
$ sudo ufw status verbose
Status: inactive            # 방화벽 비활성화인거 확인
$ sudo ufw allow 22/tcp     # 얘넨 허용
$ sudo ufw allow 80,443/tcp # 얘넨 허용
$ sudo ufw enable           # 방화벽 활성화
$ sudo ufw status verbose
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                  
80,443/tcp                 ALLOW IN    Anywhere                  
22/tcp (v6)                ALLOW IN    Anywhere (v6)             
80,443/tcp (v6)            ALLOW IN    Anywhere (v6) 
```

---
### 도메인 및 DDNS

No-ip 사이트 가입
```
이메일: i.kkongnyang2@gmail.com
비번: U2FsdGVkX1+pMjybe35mgNJBuIV/tF+brc3SX+Rcqvw=(업로드용)
```

도메인 생성
```
도메인 http://i-kkongnyang2.myddns.me와 외부 공인 ip(현재는 U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용)) 연결.
```

> ⚠️ KT 공유기 NAT 문제로 이건 안해놓음. 그래서 외부 공인 ip 바뀌면 수동으로 바꿔줘야 함.
DNS 클라이언트 설치
```
$ wget https://www.noip.com/client/linux/noip-duc-linux.tar.gz
$ tar xzf noip-duc-linux.tar.gz
$ cd noip-2.1.9-1
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

DDNS 클라이언트 실행
```
$ sudo /usr/local/bin/noip2            #실행
$ sudo /usr/local/bin/noip2 -S         #실행 확인
```
* NAT=주소 변환
[Raspberry Pi] 172.30.1.220 ──
                           ├─ 포트포워딩 (80/443)
[공유기]        112.168.5.119 (1차 IP)
                           │  <─ 내부 NAT (KT 라우터/BRAS)
[KT망]          U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용) (2차 IP) ── Internet

---
### https로 리디렉션

apache2 설정파일에 도메인 추가
```
$ sudo nano /etc/apache2/sites-available/000-default.conf
ServerName i-kkongnyang2.myddns.me
ServerAdmin webmaster@localhost
DocumentRoot /var/www/html
```

certbot 설치
```
$ sudo apt install certbot python3-certbot-apache
$ sudo certbot --apache -d i-kkongnyang2.myddns.me
안내받을 이메일 i.kkongnyang2@gmail.com
약관동의 y
광고수신동의 N
Successfully deployed certificate for i-kkongnyang2.myddns.me to /etc/apache2/sites-available/000-default-le-ssl.conf
Congratulations! You have successfully enabled HTTPS on https://i-kkongnyang2.myddns.me
```

---
### 테스트

      테스트 환경                 로컬IP                 외부IP              도메인
KT_GiGA_1D39(동일 와이파이)          O                     X                X            #NAT loopback이 없어서 불가능
KT_GiGA_5G_1D39(다른 와이파이)       O                     O                O
LTE/5G 데이터                       X                     O                O           #인터넷에선 로컬 ip주소를 쳤을때 안내해줄 안내원이 없음
