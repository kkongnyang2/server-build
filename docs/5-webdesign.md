## 서버에 페이지를 본격적으로 꾸며보자

### 목표: 웹 디자인
작성자: kkongnyang2 작성일: 2025-07-07

---
### > 메인 페이지

```
kkongnyang2.github.io/
├─ index.html
├─ profile.html
├─ pre-work.html
├─ hobby.html
├─ hyacine1.gif
├─ hyacine2.gif
├─ hyacine3.gif
├─ myphoto.jpg
└─ .github/workflows/deploy.yml
```

각 html 파일들을 만들어주면 된다.

---
### > 카테고리 페이지

우선 실시간으로 편하게 볼 수 있도록

```bash
~$ sudo apt install mkdocs
~$ pip install -U mkdocs-material mkdocs-awesome-pages-plugin \
  mkdocs-minify-plugin mkdocs-git-revision-date-localized-plugin
~$ cd server-build
~$ mkdocs serve
```

```
server-build/
├─ docs/
│   ├─ index.md
│   ├─ 1.md
│   ├─ 2.md
│   ├─ 3.md
│   ├─ 4.md
│   ├─ 5.md
│   ├─ logo.svg
│   └─ soft-era.css
├─ mkdocs.yml
└─ .github/workflows/deploy.yml
```

index 파일에는 맨 처음에 # Home를 적어준다.

mkdocs.yml은
```
site_name: Server-Build

theme:
  name: material
  logo: logo.svg
  favicon: logo.svg

  font:
    text: Inter
    code: "JetBrains Mono"

  palette:
    - scheme: apple          # 기본(라이트)
      primary: white
      accent: indigo
      toggle:
        icon: material/brush
        name: Soft Era
    - scheme: soft           # 커스텀 팔레트
      primary: custom        # custom → CSS 변수에서 덮어씀
      accent: custom
      toggle:
        icon: material/brush
        name: Apple Classic

  features:
    - navigation.tabs           # 상단 탭 내비게이션
    - content.code.copy         # 코드 블록 Copy 버튼
    - header.autohide           # 스크롤 시 헤더 숨김

extra:
  homepage: .. 

extra_css:
  - soft-era.css

markdown_extensions:
  - pymdownx.extra            # 유용 기능들 모음
  - pymdownx.highlight        # 코드 블럭 문법 강조
  - nl2br                     # 줄바꿈 자동 처리

```
meterial 테마를 베이스로 쓰되 두 팔레트를 토글할 수 있도록 만들어둔다. 그리고 사이드바가 아닌 상단 탭 내비게이션이 나아 features를 넣어주었다.

soft-era.css 파일에는
```
[data-md-color-scheme="soft"] {
  --md-default-bg-color:        #f9f5f4;     /* 전체 기본 배경 (editor background) */
  --md-default-fg-color:        #666d7dbb;   /* 기본 텍스트 색상 (editor foreground) */
  --md-typeset-color:           #c29ba3;     /* 문단/본문 색상 */

  --md-accent-fg-color:         #dd698c;     /* 강조 색상 (링크 등) */
  --md-primary-fg-color:        #f7dfe8;     /* 연한 핑크 포인트 컬러 */
  --md-typeset-a-color:         #dd698c;     /* 링크 색상 */

  --md-code-bg-color:           #fff7fa;     /* 코드 블럭 배경 */
  --md-code-fg-color:           #4a4543;     /* 코드 기본 텍스트 색상 */

  --md-code-hl-keyword-color:   #c678dd;     /* 코드 키워드 (ex. `if`, `return`) */
  --md-code-hl-function-color:  #61afef;     /* 함수 이름 등 */
  --md-code-hl-string-color:    #98c4ba;     /* 문자열 */
  --md-code-hl-constant-color:  #be88d9;     /* 숫자, 상수 등 */
  --md-code-hl-variable-color:  #db90a7;     /* 변수, 인자 등 */
  --md-code-hl-comment-color:   #948484;     /* 주석 */
  --md-code-hl-punctuation-color: #4a4543d2; /* 구두점 (괄호, 연산자 등) */

  --md-admonition-bg-color:     #e4bcbf33;   /* 참고/주의 박스 배경 */
  --md-border-color:            #e3e6ed;     /* 구분선, 테두리 등 */
  --md-selection-bg-color:      #c8bfff34;   /* 텍스트 드래그 시 배경 */

  .md-typeset h2 {
    color: #6f557388;
  }

  .md-typeset h3 {
    color: #98c4ba;
  }
}
```
로 지정해주었다.

참고로 더 색상을 고르고 싶으면 아래 표를 참고하여 한다.
```
[data-md-color-scheme="soft"] {
  --soft-era-color-01: #0000ff;
  --soft-era-color-02: #01a0e4;
  --soft-era-color-03: #252535;
  --soft-era-color-04: #25b7b8;
  --soft-era-color-05: #3b9191;
  --soft-era-color-06: #4a4543;
  --soft-era-color-07: #666d7d;
  --soft-era-color-08: #82b4e3;
  --soft-era-color-09: #8396d8;
  --soft-era-color-10: #87b6b6;
  --soft-era-color-11: #8c99d6;
  --soft-era-color-12: #905fa8;
  --soft-era-color-13: #948484;
  --soft-era-color-14: #958ac5;
  --soft-era-color-15: #9792c4;
  --soft-era-color-16: #979fd4;
  --soft-era-color-17: #988cca;
  --soft-era-color-18: #98c4ba;
  --soft-era-color-19: #9c9c9c;
  --soft-era-color-20: #Cb8dd7;
  --soft-era-color-21: #E9E4E1;
  --soft-era-color-22: #a29acb;
  --soft-era-color-23: #a474bc;
  --soft-era-color-24: #a67bbc;
  --soft-era-color-25: #b4addf;
  --soft-era-color-26: #b8bac9;
  --soft-era-color-27: #ba989c;
  --soft-era-color-28: #be88d9;
  --soft-era-color-29: #bfa4a4;
  --soft-era-color-30: #c2addf;
  --soft-era-color-31: #c7b8be;
  --soft-era-color-32: #c8b3b3;
  --soft-era-color-33: #cabf9a;
  --soft-era-color-34: #ce9ae8;
  --soft-era-color-35: #cfc9f4;
  --soft-era-color-36: #cfcccc;
  --soft-era-color-37: #d3a656;
  --soft-era-color-38: #d9c8c8;
  --soft-era-color-39: #db90a7;
  --soft-era-color-40: #dd698c;
  --soft-era-color-41: #ddbac5;
  --soft-era-color-42: #ddcdce;
  --soft-era-color-43: #e2d1d1;
  --soft-era-color-44: #e2e1df;
  --soft-era-color-45: #e3e6ed;
  --soft-era-color-46: #e4846f;
  --soft-era-color-47: #e4bcbf;
  --soft-era-color-48: #e7d59a;
  --soft-era-color-49: #e7d8d8;
  --soft-era-color-50: #e8dea2;
  --soft-era-color-51: #ea8297;
  --soft-era-color-52: #ebedf3;
  --soft-era-color-53: #ec57b4;
  --soft-era-color-54: #eceafa;
  --soft-era-color-55: #ede7c5;
  --soft-era-color-56: #eeaabe;
  --soft-era-color-57: #f1e48f;
  --soft-era-color-58: #f1ecee;
  --soft-era-color-59: #f3e6bd;
  --soft-era-color-60: #f5ebbb;
  --soft-era-color-61: #f5eded;
  --soft-era-color-62: #f6fffd;
  --soft-era-color-63: #f9f5f4;
  --soft-era-color-64: #faf5d7;
  --soft-era-color-65: #fdfbfb;
  --soft-era-color-66: #ffb4b8;
  --soft-era-color-67: #ffe8e3;
  --soft-era-color-68: #fff3f0;
  --soft-era-color-69: #fff5bf;
  --soft-era-color-70: #fffae0;
  --soft-era-color-71: #ffffff;
  --soft-era-color-72: #4a454366;
  --soft-era-color-73: #4a4543d2;
  --soft-era-color-74: #6f557388;
  --soft-era-color-75: #E9E4E18A;
  --soft-era-color-76: #c8bfff34;
  --soft-era-color-77: #cfc9f466;
  --soft-era-color-78: #cfc9f499;
  --soft-era-color-79: #e2e1dfdd;
  --soft-era-color-80: #e7d8d899;
}
```