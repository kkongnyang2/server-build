## Vscode 1.100.2 설치와 Github 연결

### 목표: 코딩 환경 세팅
작성자: kkongnyang2 작성일: 2025-06-04

---
### 1> vscode 설치

* snap으로 설치하면 ibus와의 호환 이슈. deb방식으로 설치 필요.

step 1. apt에 vscode 등록
```bash
~$ wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
~$ sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
~$ sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main" > /etc/apt/sources.list.d/vscode.list'
```

"vscode 인증키와 링크 받아와서 내 apt에 등록"

* `wget`: 인터넷에서 파일을 다운로드하는 도구
* `-q`: 출력없이 실행 -0-: 표준 출력으로(터미널 출력이 표준)
* `|`: 앞의 결과를 뒤로 넘김
* `gpg` : 보안 인증용 키를 다루는 도구
* `gpg --dearmor`: 키를 인증용으로 사용 가능하도록 .asc(아스키형식)에서 .gpg(이진형식)으로
* `>` : 앞의 결과를 뒤의 파일에 저장
* 즉 첫번째 코드는 키 파일을 인터넷에서 받아와서 형식바꿔 저장.
* `-o root` : 파일의 소유자를 root로 설정  -g root : 그룹도 root로 설정
* `-m 644` : 읽기/쓰기/읽기 허용
* `/etc/apt/trusted.gpg.d/` : 신뢰 가능한 키들 모인 디렉토리
* 즉 두번째 코드는 아까 저장한 키파일을 신뢰키 디렉토리에 설치.
* `sh -c '...'` : 하나의 문자열로 실행. echo가 sudo가 안되기 때문에 쓴거. 의미x
* `echo "..."` : 출력해라
* `deb` : 바이너리 패키지를 의미.
* `[arch=amd64]` : 64비트 아키텍처용 패키지만
* `main` : 패키지 카테고리 중 주요 소프트웨어 저장소
* `/etc/apt/sources.list.d/vscode.list` : apt 소스 리스트
* 즉 세번째 코드는 링크를 apt 소스 리스트에 등록해라.

step 2. vs code 설치
```bash
~$ sudo apt update
~$ sudo apt install code
```

step 3. 설치 위치 확인
```bash
~$ which code
/usr/bin/code                 #deb형식으로 잘 깔림
```

step 4. 실행
```bash
~$ code
```
* code는 그냥 런쳐실행, code .은 현재 폴더 실행, code ~/..는 해당 폴더를 프로젝트로 열기

---
### 2> vscode 폴더 이해

my-project/
├── .vscode/
│   ├── settings.json      ← 설정
│   ├── launch.json        ← 디버깅 설정
│   ├── tasks.json         ← 자동실행 설정
│   └── extensions.json    ← 추천 확장 목록
├── index.js
└── README.md

---
### 3> 워크스페이스 파일 이해

my-project.code-workspace  ← 여러 폴더 묶는 워크스페이스 파일 (선택적)
{
  "folders": [
    { "path": "frontend" },
    { "path": "backend" }
  ],
  "settings": {
    "editor.tabSize": 2
  }
}

---
### 4> 깃 설치

step 1. git 설치
```bash
~$ sudo apt install git
```

step 2. 사용자 정보 입력
```bash
~$ git config --global user.name "kkongnyang2"
~$ git config --global user.email "i.kkongnyang2@gmail.com"
```
* `config` : git의 설정 정보 수정
* `--global` : 모든 git 저장소에 적용
* `user.name` : 커밋할때 적히는 작성자 이름

---
### 5> 깃허브 연결

step 1. ssh 키 생성
```bash
~$ ssh-keygen -t ed25519 -C "i.kkongnyang2@gmail.com"
```

* `ssh-keygen` : 공개키/비밀키 쌍 생성. 각각 ~/.ssh/id_ed25519.pub과 ~/.ssh/id_ed25519에 저장
* `-t ed25519` : 키 타입을 ED25519 방식으로 지정 (현대식)
* `-C "..."` : 생성된 키에 구분용으로 이메일 주소 커멘트 붙임
* `passphrase` : 푸시할 때마다 암호 할거면 설정
* `SHA256` : 키 내용 압축 해시 알고리즘
* `randomart` : 키의 무작위 시각화

출력:
```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/kkongnyang2/.ssh/id_ed25519): 
Created directory '/home/kkongnyang2/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/kkongnyang2/.ssh/id_ed25519
Your public key has been saved in /home/kkongnyang2/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:BN5rnq1LOEF9Td6r9xN5/8ua11oIMDjt2FkYS40Y+iU i.kkongnyang2@gmail.com
The key's randomart image is:
+--[ED25519 256]--+
|      . .ooo.    |
|     . =.+.B..   |
|      + E O + .  |
|     . o X =   . |
|      . S + . . .|
|       = o   o +.|
|      o + . . o *|
|       o .   .o=o|
|        o.   o++*|
+----[SHA256]-----+
```

step 2. ssh 키 가져오기
```bash
~$ cat ~/.ssh/id_ed25519.pub
```
출력:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAEbkGW/CYdYgUkksfsXEMclE2CNaf1mkxH8NqE8/5oC i.kkongnyang2@gmail.com
```

step 3. ssh 키 입력

해당 키를 github들어가서 settings-ssh키에 입력

---
### 6> 깃허브 연동폴더 생성

ubuntu-set/
├── README.md         ← 작업 파일
├── main.py           ← 작업 파일
├── .git/             ← 추적 (숨김 폴더)
│   ├── config
│   ├── HEAD
│   ├── objects/
│   └── ...

방법 1. 가져오기
```bash
~$ git clone git@github.com:kkongnyang2/ubuntu-set.git
```

* `git clone` : github에서 받아와 로컬폴더 만들고 git 적용
* `git@github.com:kkongnyang2/ubuntu-set.git` : github의 ssh 주소

방법 2. 여기서 만들기
```bash
~$ mkdir ubuntu-set
~$ cd ubuntu-set
~$ git init
~$ git remote add origin git@github.com:kkongnyang2/ubuntu-set.git
```

* `git init` : 지금 있는 로컬폴더에 git을 적용하고 github 용으로 초기화
* `remote` : github 폴더 설정
* `origin` : github 폴더의 기본별명

---
### 7> 폴더 변경내용 반영

```bash
~$ git add .
~$ git commit -m "처음 업로드"
~$ git push -u origin main
```

* `add` : 변경 파일을 git에 추가
* `.` : 전부
* `commit` : 스냅샷 저장 
* `-m "..."` : 메시지 옵션
* `push` : github로 업로드
* `-u` : 로컬 폴더와 github 폴더 연결. 이후에는 안쳐도됨
* `origin` : github 폴더의 기본별명
* `main` : 로컬 폴더의 기본별명

> ⚠️ github 측에서 수정된 게 있다면 꼭 먼저
```bash
~$ git pull origin main
```

---
### 8> Github pages 블로그 만들기

step 1. 주의사항

your-repo/
├── _config.yml
├── index.md        ← ✅ 첫 화면에 렌더링됨
├── about.md        ← ❌ 직접 들어가야 접근됨 (/about)
├── tips.md         ← ❌ 직접 들어가야 접근됨 (/tips)

* 따라서 `[링크](/about)`으로 index파일에 모으기

step 2. 레지스토리 공개로 전환
* github - ubuntuset - settings - 공개로 전환

step 3. source 선택
* github - ubuntuset - settings - pages 에 들어가기

방법 1. deploy from a branch 선택 시 yml 파일을 직접 만들어서 넣어줘야 함. 특정 브랜치 파일을 그대로 페이지로 만들어줌. 브랜치 main, root로 save

_config.yml
```
title: 꽁냥2의 기록
theme: minima
```

방법 2. github actions 선택 시 yml 자동화 파일을 제공해줌. 복잡한 커스터마이징.

step 4. 주소로 들어가기

```
https://kkongnyang2.github.io/ubuntu-set/
```

---
### 9> 마크다운 문법

미리보기는 ctrl+shift+v

```
# header
## header2
### header3
text1
text2
**굵게**
*기울임*
**_굵고 기울임_**
1. 가나다
2. 가나
* 가나다
* 가나
- [x] 완료한 일
> 인용
---
`인라인 강조`
[링크](https://example.com)
![이미지](https://example.com/image.png)
```