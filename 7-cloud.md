## Ubuntu 22.04.5에서 클라우드 서버를 구축해보자

### 목표: 클라우드 사용
작성자: kkongnyang2 작성일: 2025-06-05

---
### 1> mariaDB 설치

step 1. mariaDB 설치
```bash
~$ sudo apt install mariadb-server    #10.6.22-0ubuntu0.22.04.1 버전
```
step 2. mariaDB 보안 설정
```bash
~$ sudo mysql_secure_installation
Switch to unix_socket authentication [Y/n] n  #디폴트는 비밀번호
 ... skipping.
Change the root password? [Y/n] n     #비밀번호 없이 가도 ok
 ... skipping.
Remove anonymous users? [Y/n] y
 ... Success!
Disallow root login remotely? [Y/n] y
 ... Success!
Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!
Reload privilege tables now? [Y/n] y
 ... Success!
```
* `unix_socket`: 우분투의 root 사용자 권한과 동일시

step 3. mariaDB 사용자 생성
```bash
~$ sudo mysql -u root -p
MariaDB [(none)]> CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
MariaDB [(none)]> CREATE USER 'kkongnyang2'@'localhost' IDENTIFIED BY '1234';
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
## 2> Nextcloud 설치
Nextcloud는 웹 기반의 파일 서버로, 아까 Apache 환경 위에 설치되는 핵심 애플리케이션.

step 1. PHP 및 모듈 설치
```bash
~$ sudo apt install php libapache2-mod-php php-mysql php-xml php-mbstring php-curl php-zip php-gd php-intl php-bcmath php-imagick php-cli unzip
```
step 2. nextcloud 설치
```bash
~$ wget https://download.nextcloud.com/server/releases/latest.zip
~$ unzip latest.zip
```
step 3. apache 경로에 넣기
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
```

step 4. 퍼미션 설정
```bash
~$ sudo chown -R www-data:www-data /var/www/html/nextcloud
~$ sudo chmod -R 755 /var/www/html/nextcloud
```
* `chown`: 소유자를 apache 사용자(www-data)로 변경
* `chowd`: 폴더 접근 권한 (소유자는 r/w, 다른 사람은 r)

step 5. 모듈 활성화 및 재시작
```bash
~$ sudo a2enmod rewrite
~$ sudo systemctl restart apache2
```
확인:
```
http://172.30.1.220/nextcloud 들어가보기
```

## 7> Nextcloud 설정

nextcloud 설정:
```
http://172.30.1.220/nextcloud 페이지에 들어가
아이디: kkongnyang2
비밀번호: naciwotfhr@8104
Data folder: /var/www/html/nextcloud/data
Database account: kkongnyang2
Database password: 1234
Database name: nextcloud
Database host: localhost
```
확인:
```
http://172.30.1.220/nextcloud 들어가기
```


///////여기부터 안함

## 8> Nextcloud 이용
서버 날라갈거 대비해 외장 저장장치에 백업 rsync+cron
클라이언트에서 앱 설치해서 클라우드처럼 이용. 폴더 하나랑 동기화도 가능.
mail 기능도 있음.
외부 스토리지 (google drive 등)과도 연결해서 nextcloud의 일부처럼 접근하고 관리 가능


신뢰 도메인 추가:
```bash
~$ sudo nano /var/www/html/nextcloud/config/config.php
'trusted_domains' => 
array (
  0 => 'localhost',
  1 => '172.30.1.210',s
  2 => 'kkongnyang2.ddns.net',
),
로 수정.
```
확인:
```
http://kkongnyang2.ddns.net/nextcloud 접속해보기
```

