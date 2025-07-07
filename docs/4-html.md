## 서버에 html파일 자동 업로드툴을 만들자

### 목표: workflows 이해
작성자: kkongnyang2 작성일: 2025-06-17

---
### 0> 흐름

```
   로컬 폴더    -push->  github 레지스토리  -workflow->  github action  -mkdocs   -> -deploy-> github.io/server-build
server-build            server-build                   클라이언트       html로   └─ -ssh-> pi의 /var/www/html -apache2-> i-kkongnyang2.myddns.me/server-build
```

---
### 1> 폴더 구조

step 1. 폴더 구조 만들기
```
server-build/
├─ docs/
│   ├─ index.md
│   └─ 1.md
├─ mkdocs.yml
└─ .github/workflows/deploy.yml
```

step 2. mkdocs.yml
```
site_name: server-build
theme:
  name: material
nav:                    #사이드바 목차
  - pi: 1-pi.md
  - server: 2-server.md
  - cloud: 3-cloud.md
  - html: 4-html.md
docs_dir: docs          #기본값은 docs. 따라서 안써도 되지만 그냥 명시
site_dir: site          #기본값은 site. 따라서 안써도 되지만 그냥 명시
markdown_extensions:
  - nl2br       # ← 줄바꿈 자동 처리
```

step 3. .github/workflows/deploy.yml
```
~$ mkdir -p .github/workflows
```
```
name: Daily Deploy to GitHub Pages & Raspberry Pi       #이름

on:
  push:                 # 수동 push 시 즉시
#  schedule:             # 매일 18:00 UTC
#    - cron:  '0 18 * * *'

jobs:
  build:
    runs-on: ubuntu-latest      #github가 제공하는 우분투 러너 사용
    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-python@v5     #mkdocs가 필요로 하는 python 설치
      with: { python-version: '3.11' }

    - name: Install MkDocs & theme
      run: pip install mkdocs mkdocs-material     #추가 플러그인 있으면 스페이스로 이어서 적을것

    - name: Build MkDocs
      run: mkdocs build --strict             # mkdocs.yml을 읽어 site/ 폴더 생성

    # ---------- GitHub Pages 배포 ----------
    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v4
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir:  site                   # site/ → gh-pages 브랜치 푸시

    # ---------- Raspberry Pi 배포 ----------
    - name: Add Pi to known_hosts
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.PI_KNOWN_HOSTS }}" >> ~/.ssh/known_hosts      #pi의 호스트 키 사전 등록

    - name: Copy to Raspberry Pi
      uses: appleboy/scp-action@v0.1.3
      with:
        host:       ${{ secrets.PI_HOST }}          #pi의 ip 주소
        username:   ${{ secrets.PI_USER }}          #pi의 사용자
        key:        ${{ secrets.PI_SSH_KEY }}       #ssh로그인을 위한 private key
        source:     "site/*"                        #보낼것
        target:     "/var/www/html/${{ github.event.repository.name }}/"   #붙이는거 없이 루트에 두려면 /var/www/html로 바꾸기
        strip_components: 1                         # site/ 디렉터리 계층 제거
```

---
### 2> 내 컴퓨터에서

step 1. PI_KNOWN_HOSTS 알아내기
```bash
~$ ssh-keyscan -p22 172.30.1.222
```

step 2. 비밀번호 없는 ssh 키 쌍 생성
```bash
~$ ssh-keygen -t ed25519 -f id_ed25519_pi -N ""
Generating public/private ed25519 key pair.
Your identification has been saved in id_ed25519_pi       #private key는 id_ed25519_pi에 저장
Your public key has been saved in id_ed25519_pi.pub       #public kye는 id_ed25519_pi.pub에 저장
The key fingerprint is:
SHA256:sDGJubvrofTtJOfdXydN052qkDMxUIGuitUg1SaNJJ8 kkongnyang2@kkongnyang2-930XCJ-931XCJ-930XCR
The key's randomart image is:
+--[ED25519 256]--+
| ...+   .o.      |
|  o+.* o.        |
|  .E= *.         |
| . . . *.       +|
|  . + o So     o+|
|   . +    +   .o.|
|  + = o  =   .o o|
| o + O . .+ .. o |
|  ..=o+ . .o.    |
+----[SHA256]-----+
```

step 3. secret 입력

github - repo - settings - secrets and varables - actions 에 들어가 입력

```
PI_HOST : U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용)      #pi의 공인ip주소
PI_USER : deployer       #pi의 사용자 이름
PI_SSH_KEY : #아까 keygen한 private key
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACBC4ALhGcfEwRoT0Twq9YNL3XyE6PDY6qmK74qyZvatuAAAALCs5NegrOTX
oAAAAAtzc2gtZWQyNTUxOQAAACBC4ALhGcfEwRoT0Twq9YNL3XyE6PDY6qmK74qyZvatuA
AAAEDTndvXduj9G0yedw1h6LTUrKLFIOKEXbEGm/yE6ydTD0LgAuEZx8TBGhPRPCr1g0vd
fITo8NjqqYrvirJm9q24AAAALGtrb25nbnlhbmcyQGtrb25nbnlhbmcyLTkzMFhDSi05Mz
FYQ0otOTMwWENSAQ==(업로드용)
-----END OPENSSH PRIVATE KEY-----
PI_KNOWN_HOSTS : #아까 찾아놓은 known_hosts
172.30.1.222 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKsC8q8JdGM+d6RIkWQLKM2kG2PnUdiNrnM4swlYYbnh
```
* 입력하는 이유? github action이 클라이언트가 되어 내 pi 서버에 로그인해야함

---
### 3> pi에서

step 1. 업로드용 사용자 생성
```bash
~$ sudo adduser --disabled-password --gecos "" deployer    #비밀번호도 설명란도 없는 사용자 'deployer' 생성
~$ sudo usermod -aG www-data deployer         # www-data를 그룹에 추가
```

step 2. 업로드할 곳 퍼미션 설정
```bash
~$ sudo chown -R deployer:www-data /var/www/html       #/var/www/html의 소유자와 그룹 변경(퍼미션은 755니까 이러면 deployer만 write할수 있음)
~$ sudo chown -R www-data:www-data /var/www/html/nextcloud     #내부에 nextcloud의 소유자와 그룹도 바꼈을테니 다시 올바르게(퍼미션은 바꾼적 없으니 750 잘되어있음)
```

step 3. public key 등록
```bash
sudo -u deployer mkdir -p /home/deployer/.ssh       #해당 계정 home에서 .ssh 디렉토리 생성
sudo -u deployer chmod 700 /home/deployer/.ssh      #디렉토리 퍼미션을 700으로 해두기
sudo -u deployer nano /home/deployer/.ssh/authorized_keys   #아까 내 컴퓨터에서 생성한 ssh 키 중 public key 입력
sudo chmod 600 /home/deployer/.ssh/authorized_keys  #파일 퍼미션을 600으로 해두기
```

---
### 4> kkongnyang2.github.io 레포 추가

웹페이지 제일 첫 페이지가 될 루트이다. 직접 만든 index.html과 .github/workflows/deploy.yml을 넣어준다.

deploy.yml에서 mkdocs 부분만 빼고 target:     "/var/www/html/"으로만 수정하면 됨

---
### 5> 마지막 작업

step 1. 모든 레포 공개 public 으로 당연히 바꿔주기

step 2. repo - settings - actions - general 들어가서
workflow permissions 섹션에서 read and write permissions 로 변경

step 3. 공유기 페이지 들어가서 22번 포트 포트포워딩 해주기

step 4. 첫 업로드 이후 repo - settings → Pages 에서 Deploy from branch → gh-pages / (root) 설정     #얘는 github pages에 업로드하는 브랜치

---
### 6> 개인정보 암호화

```bash
~$ echo -n "원하는 문구" | openssl enc -aes-256-cbc -a -salt -pbkdf2 -pass pass:내비밀번호
암호 문구
~$ echo "암호 문구" | \
openssl enc -aes-256-cbc -a -d -pbkdf2 -pass pass:내비밀번호
원래 문구
```
```
비밀번호 -> U2FsdGVkX19dkJUxkq0yzdg74lVWKoQtbJma/YUyJCs=(업로드용)
공인ip주소 -> U2FsdGVkX19uR7WUsjBbuMdF4oG3heLi0O+nMKzum2Y=(업로드용)
비밀번호2 -> U2FsdGVkX1+pMjybe35mgNJBuIV/tF+brc3SX+Rcqvw=(업로드용)
개인키 -> U2FsdGVkX18oowmiNgvyUfNYrIgmveJ77XIBl8f6/CkWhmNdJTX1DtzJ0Ym5lgoE
QVcHLcQae5I9hnvEGdpE0wDyYmmhU9ToeUycUPYROGaGhO5PQC9XQQ0kZY7g5JR6
QmziVX/r4McqHpemsYihgTE3OZb4M9efkUgQDw5wpE5S5xg+YbgTYu/aJEfIpjA/
39LFLDy1BzcUr9pCjN4WZuzqCSMwquxFU/CNCwfLGy9bhHCNR/jn6H2gnMzZKeSj
j+hU3avIPgbwwl49TVYGyLVd6c2OglGOdzAK8HSJdJY/0BTan2R8NwILFugBqu5A
fCq9SCP7KQNydfEkfXKiUd2LGwcfw7j0NCDno8wK2BZse+xvC0Wjc1idIs80RL+n
sWuHI0zkxc0lEjTENS5AcSkcTtrcOLQOVYd39j8/T0nF1cF2rVCIJxlopV8cv533
WIGtKORT4nXF5q3qTERqBHvhrpoB5WopAIi33zERYHHHtkodf4cBQJo2fr0/AaiI
7tp8zAN8OX7pGcYku8v5ng==(업로드용)
```
