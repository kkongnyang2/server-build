## html 파일 업로드

### 1> 흐름

   로컬 폴더    -push->  github 레지스토리  -workflow->  github action  -mkdocs   -> -deploy-> github.io/server-build
server-build            server-build                   클라이언트       html로   └─ -ssh-> pi의 /var/www/html -apache2-> i-kkongnyang2.myddns.me/server-build


### 2> 폴더 구조

server-build/
├─ 1.md    
├─ 2.md
├─ 3.md
├─ mkdocs.yml
└─ .github/workflows/deploy.yml

mkdocs.yml
```
site_name: server-build
theme:
  name: material
nav:
  - 이론: 0-network.md
  - pi: 1-raspberrypi.md
  - server: 2-server.md
  - cloud: 3-cloud.md
  - html: 4-html.md
docs_dir: .             #기본값은 docs. 따로 문서 폴더 안만들거기에 루트로.
site_dir: site          #기본값은 site. 따라서 안써도 되지만 그냥 명시
markdown_extensions:
  - toc
  - codehilite
```

mkdir -p .github/workflows
.github/workflows/deploy.yml
```
name: Daily Deploy to GitHub Pages & Raspberry Pi       #이름

on:
  push:                 # 수동 push 시 즉시
  schedule:             # 매일 18:00 UTC
    - cron:  '0 18 * * *'

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
* (레포 Settings → Pages 에서 Deploy from branch → gh-pages / (root) 확인)

### 3> 내 컴퓨터에서
ssh-keyscan -p22 172.30.1.222                           #PI_KNOWN_HOSTS 알아내기
ssh-keygen -t ed25519 -f id_ed25519_pi -N ""            #비밀번호 없는 ssh 키 쌍 생성
Generating public/private ed25519 key pair.
Your identification has been saved in id_ed25519_pi
Your public key has been saved in id_ed25519_pi.pub
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

### 4> secret값 입력

github - repo - settings - secrets and varables - actions 에 들어가 입력

PI_HOST : 172.30.1.222      #pi의 ip주소
PI_USER : deployer       #pi의 사용자 이름
PI_SSH_KEY : b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACBC4ALhGcfEwRoT0Twq9YNL3XyE6PDY6qmK74qyZvatuAAAALCs5NegrOTX
oAAAAAtzc2gtZWQyNTUxOQAAACBC4ALhGcfEwRoT0Twq9YNL3XyE6PDY6qmK74qyZvatuA
AAAEDTndvXduj9G0yedw1h6LTUrKLFIOKEXbEGm/yE6ydTD0LgAuEZx8TBGhPRPCr1g0vd
fITo8NjqqYrvirJm9q24AAAALGtrb25nbnlhbmcyQGtrb25nbnlhbmcyLTkzMFhDSi05Mz
FYQ0otOTMwWENSAQ==       #내 컴퓨터에서 keygen하고 private key 입력
PI_KNOWN_HOSTS : 172.30.1.222 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKsC8q8JdGM+d6RIkWQLKM2kG2PnUdiNrnM4swlYYbnh  #내 컴퓨터에서 172.30.1.222 known_hosts 베끼기

### 5> pi에서

pi에서 업로드용 사용자 생성
sudo adduser --disabled-password --gecos "" deployer    #비밀번호도 설명란도 없는 사용자 'deployer' 생성
sudo usermod -aG www-data deployer         # www-data 그룹에 추가
sudo chown -R deployer:www-data /var/www/html       #/var/www/html의 소유자와 그룹 변경(퍼미션은 755니까 이러면 deployer만 쓸수 있음)
sudo chown -R www-data:www-data /var/www/html/nextcloud     #내부에 nextcloud의 소유자와 그룹도 바꼈을테니 다시 올바르게(퍼미션은 바꾼적 없으니 750 잘되어있음)
sudo -u deployer mkdir -p /home/deployer/.ssh       #해당 계정 home에서 .ssh 디렉토리 생성
sudo -u deployer chmod 700 /home/deployer/.ssh      #디렉토리 퍼미션을 700으로 해두기
sudo -u deployer nano /home/deployer/.ssh/authorized_keys   #아까 내 컴퓨터에서 생성한 ssh 키 중 public key 입력
sudo chmod 600 /home/deployer/.ssh/authorized_keys  #파일 퍼미션을 600으로 해두기

### 5> portal

제일 첫 페이지

portal/
├─ index.html    
└─ .github/workflows/deploy.yml

mkdocs 부분만 빼고 target:     "/var/www/html/"으로만 수정하면 됨