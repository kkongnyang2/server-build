## Ubuntu-22.04-desktop-amd64.iso를 설치해보자

### 목표: 듀얼 OS 설치 실습
작성자: kkongnyang2 작성일: 2025-06-04

---
### 0> 환경 점검

하드웨어: 삼성 노트북
├─cpu: i5-10th amd64
├─ram: 8GB
└─ssd: 256GB
운영체제: window 11 home
외부 저장장치: 32GB usb(임시용)

---
### 1> 부팅 usb 만들기
step 1. ubuntu-22.04-desktop-amd64.iso 다운받기
step 2. rufus 다운로드
step 3. rufus로 해당 iso 이미지를 usb에 굽기
* mbr/gpt. mbr은 파티션 적은 구버전(bios호환), gpt는 신버전(ufei호환).
* 영구저장공간. usb로 디스크 설치 없이 live 부팅할때 변경사항을 usb에 저장하는 할당 공간.
* FAT32. 32bit주소체계 파일 주소록. 대부분의 EFI System Partition이 사용.

---
### 2> 설치하기
step 1. F10으로 부팅 페이지 들어가기
step 2. usb 선택
step 3. try or install 우분투 클릭
* try : live 부팅. 디스크 설치 없이 usb에서 램에 올려서 사용.

step 4. 파티션 배분 후 설치 완료

----
### 3> 점검
step 1. 듀얼부팅에서 우분투 선택해서 키기
step 2. 파티션 확인
```bash
~$ lsblk -f
```
* `-f` : 파일 시스템 정보

```
step 3. 패키지 최신상태 확인
```bash
~$ sudo apt update && sudo apt upgrade -y
```
* &&는 명령어 연속. -y는 자동 yes

step 4. 드라이버 확인
```bash
~$ sudo ubuntu-drivers devices
```
* 설치가능한 목록. 아무것도 안뜨면 굿.

---
### 4> 기본 명령어좀요

ctrl+alt+t : 터미널

~ : home. /home의 줄임말
cd : 상위 디렉토리로
cd ~/... : 디렉토리 이동
ls -al : 디렉토리 목록 보기
pwd : 현재 디렉토리 경로 출력
mkdir : 디렉토리 생성
rm : 파일/디렉토리 삭제
which : 위치

cp .. .. : 복사
mv .. .. : 이동/이름변경
touch myfile.txt
echo "Hello, world!" > hello.txt
nano myfile.txt
cat > myfile.txt
Hello, this is my first file.
cat >> example.txt
cat example.txt

top : 실시간 자원 사용
ps : 실행중인 프로세스 보기

sudo apt update : APT 패키지 목록을 갱신
sudo apt upgrade : 설치된 패키지 업그레이드
sudo apt install : 패키지 설치
sudo apt remove : 패키지 삭제
sudo apt murge
sudo : 관리자 권한

명령어1 ; 명령어2 : 두개 실행
명령어1 && 명령어2 : 앞 명령어가 성공했을때만 실행
명령어1 || 명령어2 : 앞 명령어가 실패했을때만 실행