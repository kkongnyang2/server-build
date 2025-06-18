## Ubuntu-22.04-server-arm64에서 클라우드 서버를 구축해보자

### 목표: 클라우드 사용
작성자: kkongnyang2 작성일: 2025-06-05

---
### 0> 환경 점검

하드웨어: raspberrypi4 model B
├─cpu: arm cortex A72
├─ram: 4GB
└─sd카드: 32GB micro
운영체제: ubuntu-22.04-server-arm64
외부 저장장치: 256GB usb

---
### 1> mariaDB 설치

step 1. mariaDB 설치
```bash
~$ sudo apt install mariadb-server
~$ sudo systemctl enable --now mariadb
~$ sudo mysql --version
mysql  Ver 15.1 Distrib 10.6.22-MariaDB, for debian-linux-gnu (aarch64) using  EditLine wrapper
```
step 2. mariaDB 보안 설정
```bash
~$ sudo mysql_secure_installation
Enter password for root U2FsdGVkX19dkJUxkq0yzdg74lVWKoQtbJma/YUyJCs=(업로드용)                  #비번 설정
Switch to unix_socket authentication [Y/n] n  #우분투의 root 권한과 동일시. n하면 디폴트는 비밀번호
Change the root password? [Y/n] n     #비번 변경
Remove anonymous users? [Y/n] y       #익명 사용자 제거
Disallow root login remotely? [Y/n] y #원격 루트 계정 로그인 금지
Remove test database and access to it? [Y/n] y  #테스트 데베 삭제
Reload privilege tables now? [Y/n] y    #권한 테이블 재로딩
```

step 3. mariaDB 사용자 생성
```bash
~$ sudo mysql -u root -p
MariaDB [(none)]> CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
MariaDB [(none)]> CREATE USER 'kkongnyang2'@'localhost' IDENTIFIED BY 'U2FsdGVkX19dkJUxkq0yzdg74lVWKoQtbJma/YUyJCs=(업로드용)';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* TO 'kkongnyang2'@'localhost';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> SELECT User, Host FROM mysql.user;
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| kkongnyang2 | localhost |
| mariadb.sys | localhost |
| mysql       | localhost |
| root        | localhost |
+-------------+-----------+
MariaDB [(none)]> EXIT;
```

---
### 2> php 설치
php란 동적 웹페이지를 만드는 스크립트 언어. php 코드를 해석하여 html로 변환하는 모듈

```bash
~$ sudo apt install php libapache2-mod-php php-mysql php-xml php-mbstring php-curl php-zip php-gd php-intl php-bcmath php-imagick php-cli unzip
~$ sudo systemctl restart apache2        # 새 모듈 로드
~$ php -v                                # 버전 확인
PHP 8.1.2-1ubuntu2.21 (cli) (built: Mar 24 2025 19:04:23) (NTS)
```
* libapache2-mod-php: apache2와 php를 직접 연동
* php-mysql: mariaDB와 연결
* php-xml/mbstring/curl/zip/gd: 이미지 날짜 압축 등 nextcloud 같은 php앱이 요구하는 표준 기능
* php-cli: 터미널에서 php script.php 실행
* unzip: 플러그인 압축 해제용

---
### 3> Nextcloud 설치
Nextcloud는 웹 기반의 파일 서버로, 아까 Apache 환경 위에 설치되는 핵심 애플리케이션.

step 1. nextcloud 설치
```bash
~$ wget https://download.nextcloud.com/server/releases/latest.zip
~$ unzip latest.zip
```
step 2. apache 경로에 넣기
```bash
~$ sudo mv nextcloud /var/www/html/
```
* 이러면 폴더대로 http://ip주소를 입력했을 때는 apache 화면을, http://ip주소/nextcloud를 입력햇을 때는 nextcloud화면을 띄어준다.

> ⚠️ 만약 기본 경로를 nextcloud로 설정하고 싶다면:
```bash
~$ sudo nano /etc/apache2/sites-available/nextcloud.conf
<VirtualHost *:80>
    ServerAdmin i.kkongnyang2@gmail.com
    DocumentRoot /var/www/nextcloud
    ServerName 172.30.1.220

    <Directory /var/www/nextcloud/>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined

</VirtualHost>
~$ sudo a2dissite 000-default.conf      #원래 default 파일 비활성화
~$ sudo a2ensite nextcloud.conf         #새 설정파일 활성화
~$ sudo systemctl reload apache2
```

step 3. 퍼미션 설정
```bash
~$ ls -ld /var/www/html/nextcloud       #기본 퍼미션 확인 
drwxr-xr-x 15 kkongnyang2 kkongnyang2 4096 Jun 12 16:25 /var/www/html/nextcloud
~$ sudo chown -R www-data:www-data /var/www/html/nextcloud
~$ sudo find /var/www/html/nextcloud -type d -exec chmod 750 {} \;
~$ sudo find /var/www/html/nextcloud -type f -exec chmod 640 {} \;
~$ ls -ld /var/www/html/nextcloud      #다시 확인
drwxr-x--- 15 www-data www-data 4096 Jun 12 16:25 /var/www/html/nextcloud
```
* `chown`: 소유자 변경
* `chowd`: 폴더 접근 권한

---
### 4> 외장 저장장치 세팅

step 1. usb ext4으로 포맷
```bash
GUI환경에서 disks 들어가기
format disk 에서 erase는 overwirte with zeros 선택, partitioning은 gpt 선택하여 포맷
create partition 에서 파티션 사이즈 풀로, free space는 0으로, volume name은 NC_DATA로, ext4로 생성
```
* MBR은 예전에 많이 썼던 파티션 테이블 구조, 요즘은 GPT 많이 씀
* exFAT은 가장 무난한 파일시스템, ext4는 리눅스에서 서버용으로 많이 쓰는 파일시스템

step 2. 파일 수동 마운트
```bash
~$ lsblk -o NAME,UUID,SIZE,MOUNTPOINT | grep sd     #이름 확인
sda                                               233G 
└─sda1      82fa760d-f62e-4e16-b428-e78040b3980a  233G
~$ sudo mkdir -p /srv/nextcloud-data                #마운트 포인트 생성
~$ echo "UUID=82fa760d-f62e-4e16-b428-e78040b3980a  /srv/nextcloud-data  ext4  defaults,noatime,nofail,x-systemd.automount  0  2" \
 | sudo tee -a /etc/fstab       #fstab 등록
~$ sudo mount -a                #마운트 적용
```
* nofail은 이거 없이 부팅해도 켜지긴 한다는 뜻
* x-systemd.automount 필요해질 때 마운트
* umount는 sudo poweroff시 자동으로

step 3. 퍼미션 설정
```
~$ sudo chown -R www-data:www-data /srv/nextcloud-data
~$ sudo chmod 750 /srv/nextcloud-data
~$ ls -ld /srv/nextcloud-data
drwxr-x--- 3 www-data www-data 4096 Jun 16 03:48 /srv/nextcloud-data
```

---
### 5> Nextcloud 설정

nextcloud 설정:
```
https://i-kkongnyang2.myddns.me/nextcloud 페이지에 들어가
아이디: kkongnyang2
비밀번호: U2FsdGVkX1+pMjybe35mgNJBuIV/tF+brc3SX+Rcqvw=(업로드용)
Data folder: /srv/nextcloud-data
Database account: kkongnyang2
Database password: U2FsdGVkX19dkJUxkq0yzdg74lVWKoQtbJma/YUyJCs=(업로드용)
Database name: nextcloud
Database host: localhost
```
* 기본 파일은 데이터 폴더 경로와 연동, database는 그 데이터들의 메타데이터용.

---
### 6> 추가 백업 안정장치(선택)

서버 날라갈거 대비해 2차 외장 저장장치에 백업 rsync+cron