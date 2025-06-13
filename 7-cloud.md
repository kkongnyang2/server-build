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
~$ sudo apt install mariadb-server    #10.6.22-0ubuntu0.22.04.1 버전
```
step 2. mariaDB 보안 설정
```bash
~$ sudo mysql_secure_installation
Switch to unix_socket authentication [Y/n] n  #디폴트는 비밀번호
Change the root password? [Y/n] n     #비밀번호 없이 가도 ok
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
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
### 2> Nextcloud 설치
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

### 3> Nextcloud 설정

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
* 기본 파일은 데이터 폴더 경로와 연동, database는 그 데이터들의 메타데이터용.

확인:
```
http://172.30.1.220/nextcloud 들어가기
```

---
### 4> 저장장치의 이해

ssd는 크게 

나는 외장 ssd는 휴대로 불편하여 usb를 사용하기로 했다.

---
### 5> 외장 저장장치 세팅

step 1. usb exFAT으로 포맷
```bash
~$ lsblk                            #연결 드라이브 확인
~$ sudo umount /dev/sda1            #파일 접속 해제
~$ sudo mkfs.vfat -F 32 /dev/sda1   #exFAT으로 포맷
```
* exFAT은 가장 무난한 파일시스템, ext4는 리눅스에서 서버용으로 많이 쓰는 파일시스템
* MBR은 예전에 많이 썼던 파티션 테이블 구조, 요즘은 GPT 많이 씀

step 2. 파일 수동 마운트
```bash
~$ sudo mount /dev/sda1 /mnt/myusb  #파일 접속 연결
```

step 3. 파일 자동 마운트 설정(선택)
```bash
~$ sudo blkid       #UUID 확인
/dev/sda1: UUID="CDDA-2FBB" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="eceab828-01"
~$ sudo mkdir -p /mnt/usbdrive    #마운트 포인트 생성
~$ sudo nano /etc/fstab           #자동 마운트 추가
UUID=CDDA-2FBB  /mnt/usbdrive   vfat    defaults,uid=www-data,gid=www-data,umask=0022   0       2
```
* CDDA-2FBB: 파일 시스템 UUID
* eceab828-01: 파티션 테이블 ID
* `uid=www-data,gid=www-data`: 웹서버 계정이 usb에 접근 가능하도록
* `umask=0022`: 퍼미션 기본값 설정

step 4. nextcloud 파일 디렉토리 변경
```bash
sudo mkdir -p /mnt/usbdrive/nextcloud-data
sudo nano /var/www/html/nextcloud/config/config.php
'datadirectory' => '/mnt/usbdrive/nextcloud-data',
```


---
### 6> 퍼미션의 이해


---
### 7> 추가 백업 안정장치(선택)

서버 날라갈거 대비해 2차 외장 저장장치에 백업 rsync+cron



신뢰 도메인 추가:
```bash
~$ sudo nano /var/www/html/nextcloud/config/config.php
'trusted_domains' => 
array (
  0 => 'localhost',
  1 => '172.30.1.210',
  2 => 'i-kkongnyang2.myddns.me',
),
로 수정.
```
확인:
```
http://kkongnyang2.ddns.net/nextcloud 접속해보기
```







kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
kkongnyang2@172.30.1.220's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1078-raspi aarch64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Jun  7 18:47:50 KST 2025

  System load:  1.7                Temperature:            47.2 C
  Usage of /:   19.7% of 28.68GB   Processes:              166
  Memory usage: 7%                 Users logged in:        0
  Swap usage:   0%                 IPv4 address for wlan0: 172.30.1.220

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

7 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

5 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


Last login: Fri Jun  6 02:12:58 2025 from 172.30.1.54
kkongnyang2@raspberrypi:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0 59.8M  1 loop /snap/core20/2321
loop1         7:1    0 59.6M  1 loop /snap/core20/2585
loop2         7:2    0 77.4M  1 loop /snap/lxd/29353
loop3         7:3    0 79.5M  1 loop /snap/lxd/31335
loop4         7:4    0 33.7M  1 loop /snap/snapd/21761
loop5         7:5    0 44.3M  1 loop /snap/snapd/24509
sda           8:0    1  233G  0 disk 
└─sda1        8:1    1  233G  0 part 
mmcblk0     179:0    0 29.7G  0 disk 
├─mmcblk0p1 179:1    0  512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 29.2G  0 part /
kkongnyang2@raspberrypi:~$ sudo umount /dev/sda1
[sudo] password for kkongnyang2: 
umount: /dev/sda1: not mounted.
kkongnyang2@raspberrypi:~$ sudo mkfs.vfat -F 32 /dev/sda1
mkfs.fat 4.2 (2021-01-31)
kkongnyang2@raspberrypi:~$ sudo blkid
[sudo] password for kkongnyang2: 
/dev/mmcblk0p1: LABEL_FATBOOT="system-boot" LABEL="system-boot" UUID="9FBA-406B" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="ef507fe6-01"
/dev/mmcblk0p2: LABEL="writable" UUID="f7380f37-4684-4f77-abe0-219400572e43" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ef507fe6-02"
/dev/sda1: UUID="CDDA-2FBB" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="eceab828-01"
/dev/loop1: TYPE="squashfs"
/dev/loop4: TYPE="squashfs"
/dev/loop2: TYPE="squashfs"
/dev/loop0: TYPE="squashfs"
/dev/loop5: TYPE="squashfs"
/dev/loop3: TYPE="squashfs"
kkongnyang2@raspberrypi:~$ sudo mkdir -p /mnt/usbdrive
kkongnyang2@raspberrypi:~$ sudo nano /etc/fstab
kkongnyang2@raspberrypi:~$ sudo mkdir -p /mnt/usbdrive/nextcloud-data
kkongnyang2@raspberrypi:~$ sudo chown -R www-data:www-data /mnt/usbdrive/nextcloud-data
[sudo] password for kkongnyang2: 
kkongnyang2@raspberrypi:~$ sudo nano /var/www/html/nextcloud/config/config.php
kkongnyang2@raspberrypi:~$ sudo touch /mnt/usbdrive/nextcloud-data/textfile.txt
kkongnyang2@raspberrypi:~$ sudo touch /mnt/usbdrive/nextcloud-data/.ocdata
kkongnyang2@raspberrypi:~$ sudo ls /mnt/usbdrive/
nextcloud-data
kkongnyang2@raspberrypi:~$ sudo ls /mnt/subdirve/nextcloud-data
ls: cannot access '/mnt/subdirve/nextcloud-data': No such file or directory
kkongnyang2@raspberrypi:~$ sudo ls /mnt/usbdrive/nextcloud-data
appdata_ocn62ffkl7fk  nextcloud.log  textfile.txt
kkongnyang2@raspberrypi:~$ exit
logout
Connection to 172.30.1.220 closed.
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
^C
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
kkongnyang2@172.30.1.220's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1078-raspi aarch64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Jun  7 21:18:08 KST 2025

  System load:  2.38               Temperature:            48.2 C
  Usage of /:   19.8% of 28.68GB   Processes:              166
  Memory usage: 7%                 Users logged in:        0
  Swap usage:   0%                 IPv4 address for wlan0: 172.30.1.220

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

9 updates can be applied immediately.
1 of these updates is a standard security update.
To see these additional updates run: apt list --upgradable

5 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm

New release '24.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sat Jun  7 18:47:52 2025 from 172.30.1.54
kkongnyang2@raspberrypi:~$ exit
logout
Connection to 172.30.1.220 closed.
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
kkongnyang2@172.30.1.220's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1078-raspi aarch64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Jun  7 21:20:25 KST 2025

  System load:  0.44               Temperature:            44.8 C
  Usage of /:   19.8% of 28.68GB   Processes:              164
  Memory usage: 8%                 Users logged in:        0
  Swap usage:   0%                 IPv4 address for wlan0: 172.30.1.220

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

9 updates can be applied immediately.
1 of these updates is a standard security update.
To see these additional updates run: apt list --upgradable

5 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm

New release '24.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sat Jun  7 21:18:11 2025 from 172.30.1.54
kkongnyang2@raspberrypi:~$ sudo touch /mnt/usbdrive/nextcloud-data/.ncdata
[sudo] password for kkongnyang2: 
kkongnyang2@raspberrypi:~$ ls -l /mnt/usbdrive
total 32
drwxr-xr-x 2 www-data www-data 32768 Jun  7 21:21 nextcloud-data
kkongnyang2@raspberrypi:~$ ls -l /mnt/usbdrive/nextcloud-data
total 0
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:20 nextcloud.log
kkongnyang2@raspberrypi:~$ ls -l /mnt/usbdrive/nextcloud-data/testfile.txt
ls: cannot access '/mnt/usbdrive/nextcloud-data/testfile.txt': No such file or directory
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/testfile.txt
ls: cannot access '/mnt/usbdrive/nextcloud-data/testfile.txt': No such file or directory
kkongnyang2@raspberrypi:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0 59.8M  1 loop /snap/core20/2321
loop1         7:1    0 59.6M  1 loop /snap/core20/2585
loop2         7:2    0 77.4M  1 loop /snap/lxd/29353
loop3         7:3    0 79.5M  1 loop /snap/lxd/31335
loop4         7:4    0 33.7M  1 loop /snap/snapd/21761
loop5         7:5    0 44.3M  1 loop /snap/snapd/24509
sda           8:0    1  233G  0 disk 
└─sda1        8:1    1  233G  0 part /mnt/usbdrive
mmcblk0     179:0    0 29.7G  0 disk 
├─mmcblk0p1 179:1    0  512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 29.2G  0 part /
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/
nextcloud.log
kkongnyang2@raspberrypi:~$ sudo umount /dev/sda1
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/
ls: cannot open directory '/mnt/usbdrive/nextcloud-data/': Permission denied
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/
nextcloud-data
kkongnyang2@raspberrypi:~$ sudo umount /dev/sda1
umount: /dev/sda1: not mounted.
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/
ls: cannot open directory '/mnt/usbdrive/nextcloud-data/': Permission denied
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/
nextcloud-data
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/
ls: cannot open directory '/mnt/usbdrive/nextcloud-data/': Permission denied
kkongnyang2@raspberrypi:~$ sudo mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,nosuid,relatime,size=1887120k,nr_inodes=471780,mode=755,inode64)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,noexec,relatime,size=388092k,mode=755,inode64)
/dev/mmcblk0p2 on / type ext4 (rw,relatime,discard,errors=remount-ro)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,inode64)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k,inode64)
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)
pstore on /sys/fs/pstore type pstore (rw,nosuid,nodev,noexec,relatime)
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=29,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=14538)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=2M)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
debugfs on /sys/kernel/debug type debugfs (rw,nosuid,nodev,noexec,relatime)
tracefs on /sys/kernel/tracing type tracefs (rw,nosuid,nodev,noexec,relatime)
configfs on /sys/kernel/config type configfs (rw,nosuid,nodev,noexec,relatime)
fusectl on /sys/fs/fuse/connections type fusectl (rw,nosuid,nodev,noexec,relatime)
none on /run/credentials/systemd-sysusers.service type ramfs (ro,nosuid,nodev,noexec,relatime,mode=700)
/var/lib/snapd/snaps/core20_2321.snap on /snap/core20/2321 type squashfs (ro,nodev,relatime,errors=continue,x-gdu.hide)
/var/lib/snapd/snaps/core20_2585.snap on /snap/core20/2585 type squashfs (ro,nodev,relatime,errors=continue,x-gdu.hide)
/var/lib/snapd/snaps/lxd_29353.snap on /snap/lxd/29353 type squashfs (ro,nodev,relatime,errors=continue,x-gdu.hide)
/var/lib/snapd/snaps/lxd_31335.snap on /snap/lxd/31335 type squashfs (ro,nodev,relatime,errors=continue,x-gdu.hide)
/var/lib/snapd/snaps/snapd_21761.snap on /snap/snapd/21761 type squashfs (ro,nodev,relatime,errors=continue,x-gdu.hide)
/var/lib/snapd/snaps/snapd_24509.snap on /snap/snapd/24509 type squashfs (ro,nodev,relatime,errors=continue,x-gdu.hide)
/dev/mmcblk0p1 on /boot/firmware type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,errors=remount-ro)
binfmt_misc on /proc/sys/fs/binfmt_misc type binfmt_misc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /run/snapd/ns type tmpfs (rw,nosuid,nodev,noexec,relatime,size=388092k,mode=755,inode64)
nsfs on /run/snapd/ns/lxd.mnt type nsfs (rw)
tmpfs on /run/user/1000 type tmpfs (rw,nosuid,nodev,relatime,size=388088k,nr_inodes=97022,mode=700,uid=1000,gid=1003,inode64)
kkongnyang2@raspberrypi:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0 59.8M  1 loop /snap/core20/2321
loop1         7:1    0 59.6M  1 loop /snap/core20/2585
loop2         7:2    0 77.4M  1 loop /snap/lxd/29353
loop3         7:3    0 79.5M  1 loop /snap/lxd/31335
loop4         7:4    0 33.7M  1 loop /snap/snapd/21761
loop5         7:5    0 44.3M  1 loop /snap/snapd/24509
sda           8:0    1  233G  0 disk 
└─sda1        8:1    1  233G  0 part 
mmcblk0     179:0    0 29.7G  0 disk 
├─mmcblk0p1 179:1    0  512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 29.2G  0 part /
kkongnyang2@raspberrypi:~$ sudo mount /dev/sda1 /mnt/usbdrive
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/
nextcloud.log
kkongnyang2@raspberrypi:~$ sudo touch /mnt/usbdrive/nextcloud-data/textfile.txt
kkongnyang2@raspberrypi:~$ ls /mnt/usbdrive/nextcloud-data/
nextcloud.log  textfile.txt
kkongnyang2@raspberrypi:~$ ls -l /mnt/usbdrive/nextcloud-data
total 0
-rwxr-xr-x 1 root root 0 Jun  7 21:20 nextcloud.log
-rwxr-xr-x 1 root root 0 Jun  7 21:38 textfile.txt
kkongnyang2@raspberrypi:~$ sudo chown -R www-data:www-data /mnt/usbdrive/nextcloud-data
chown: changing ownership of '/mnt/usbdrive/nextcloud-data/nextcloud.log': Operation not permitted
chown: changing ownership of '/mnt/usbdrive/nextcloud-data/.ncdata': Operation not permitted
chown: changing ownership of '/mnt/usbdrive/nextcloud-data/textfile.txt': Operation not permitted
chown: changing ownership of '/mnt/usbdrive/nextcloud-data': Operation not permitted
kkongnyang2@raspberrypi:~$ exit
logout
Connection to 172.30.1.220 closed.
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: No route to host
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
ssh: connect to host 172.30.1.220 port 22: Connection refused
kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR:~$ ssh kkongnyang2@172.30.1.220
kkongnyang2@172.30.1.220's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1078-raspi aarch64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Jun  7 21:47:38 KST 2025

  System load:  1.28               Temperature:            51.6 C
  Usage of /:   20.0% of 28.68GB   Processes:              164
  Memory usage: 7%                 Users logged in:        0
  Swap usage:   0%                 IPv4 address for wlan0: 172.30.1.220

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

9 updates can be applied immediately.
1 of these updates is a standard security update.
To see these additional updates run: apt list --upgradable

5 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm

New release '24.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sat Jun  7 21:20:26 2025 from 172.30.1.54
kkongnyang2@raspberrypi:~$ sudo ls /mnt/usbdrive/nextcloud-data
[sudo] password for kkongnyang2: 
nextcloud.log  textfile.txt
kkongnyang2@raspberrypi:~$ sudo -l ls /mnt/usbdrive/nextcloud-data
/usr/bin/ls /mnt/usbdrive/nextcloud-data
kkongnyang2@raspberrypi:~$ ls -l /mnt/usbdrive/nextcloud-data/testfile.txt
ls: cannot access '/mnt/usbdrive/nextcloud-data/testfile.txt': No such file or directory
kkongnyang2@raspberrypi:~$ ls -l /mnt/usbdrive/nextcloud-data/
total 0
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:20 nextcloud.log
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:38 textfile.txt
kkongnyang2@raspberrypi:~$ sudo ls -l /mnt/usbdrive/nextcloud-data
total 0
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:20 nextcloud.log
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:38 textfile.txt
kkongnyang2@raspberrypi:~$ sudo nano /etc/fstab
kkongnyang2@raspberrypi:~$ sudo mount -o remount /mnt/usbdrive
kkongnyang2@raspberrypi:~$ sudo ^[[200~ls -ld /mnt/usbdrive/nextcloud-data
sudo: ls: command not found
kkongnyang2@raspberrypi:~$ sudo ls -ld /mnt/usbdrive/nextcloud-data
drwxr-xr-x 2 www-data www-data 32768 Jun  7 21:38 /mnt/usbdrive/nextcloud-data
kkongnyang2@raspberrypi:~$ sudo ls -l /mnt/usbdrive/nextcloud-data
total 0
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:20 nextcloud.log
-rwxr-xr-x 1 www-data www-data 0 Jun  7 21:38 textfile.txt
kkongnyang2@raspberrypi:~$ sudo nano /etc/fstab
kkongnyang2@raspberrypi:~$ exit
logout
Connection to 172.30.1.220 closed.
