# 메인 페이지 개편 설계

- 작성일: 2026-08-19
- 대상 저장소: `nameight88/github_page` (Jekyll + Minimal Mistakes 4.24.0 remote theme)
- 빌드 환경: `github-pages ~> 232` → **Jekyll 3.10** (Jekyll 4 전용 필터 사용 불가)

## 문제

메인 페이지의 "최근 포스트" 영역이 비어 있고, `/categories/` 페이지에도 카테고리별 글이 하나도 나오지 않는다.

### 근본 원인

`_posts` 안의 파일명에 날짜 접두사가 없다. Jekyll은 `_posts` 하위 파일이 `YYYY-MM-DD-제목.확장자` 형식일 때만 포스트 컬렉션에 넣는다. front matter에 `date:`가 있어도 파일명 규칙은 별도로 필수다.

- `_posts/database/sql-server-backup.md`
- `_posts/programing/effective_c++_chapter1.md`

두 파일 모두 규칙에 맞지 않아 `site.posts`가 비어 있다. 그 결과 메인, `/posts/`, `/categories/`, `/tags/` 네 화면이 동시에 비어 있다. 즉 "최근 포스트가 안 보임"과 "카테고리에 글이 안 생김"은 서로 다른 문제가 아니라 같은 원인의 두 증상이다.

### 두 번째 문제

`_data/navigation.yml`의 사이드바는 카테고리 앵커 10개(`#python`, `#java`, `#javascript`, `#mysql`, `#postgresql`, `#langchain`, `#ollama`, `#huggingface`, `#cpp`, `#sql-server`)를 가리키지만, 실제 포스트가 가진 카테고리는 `sql-server`, `database`, `cpp`, `programming` 넷뿐이다. Minimal Mistakes의 `layout: categories`는 글이 있는 카테고리만 `<h2 id="...">` 앵커를 만들므로, 파일명을 고쳐도 나머지 8개 링크는 여전히 아무 데도 닿지 않는다.

## 설계 원칙

**화면에 보이는 모든 링크는 실제로 존재하는 곳으로 간다.** 아직 글이 없는 분야를 로드맵으로 늘어놓는 대신, 글을 쓰면 메인과 사이드바가 따라오는 구조를 만든다.

## 변경 사항

### 1. 포스트 파일명 정규화

front matter의 `date` 값을 그대로 접두사로 쓴다.

| 현재 | 변경 후 |
|---|---|
| `_posts/database/sql-server-backup.md` | `_posts/database/2024-05-12-sql-server-backup.md` |
| `_posts/programing/effective_c++_chapter1.md` | `_posts/programming/2025-10-10-effective-cpp-chapter1.md` |

- 두 번째 파일의 `+` 문자는 제거한다. URL에 들어가면 인코딩 문제를 일으키고, 화면에 보이는 제목은 front matter의 `title`에서 오므로 표시에는 영향이 없다.
- 오타인 `programing/` 폴더는 `programming/`으로 고친다. `_posts` **아래**의 하위 폴더는 Jekyll에서 카테고리에 영향을 주지 않는 순수 정리용이므로 URL과 카테고리 모두 변하지 않는다.
- 기존에 이 글들이 포스트로 인식된 적이 없으므로 깨질 외부 링크는 없다.

### 2. 포스트에 `excerpt` 명시

현재 `_config.yml`의 `excerpt_separator: "\n\n"` 규칙상 본문 첫 블록이 발췌로 잡힌다. `sql-server-backup.md`는 본문 첫 줄이 `# SQL-Server_Backup_auto_System(DR.ver)`이라 발췌가 제목 반복이 된다.

두 포스트의 front matter에 `excerpt:` 한 줄을 추가한다. 내용은 각 글의 본문을 읽고 한 문장(40~80자)으로 요약한다. 앞으로 쓰는 글도 `excerpt`를 넣는 것을 규칙으로 삼는다.

### 3. 카테고리 표시 이름 데이터 파일 신설

`_data/category_names.yml`을 만든다. 파일명에 하이픈 대신 밑줄을 쓰는 이유는 Liquid에서 `site.data.category-names`처럼 점 표기로 접근할 수 없기 때문이다.

```yaml
- key: cpp
  name: "C/C++"
- key: programming
  name: "프로그래밍"
- key: sql-server
  name: "SQL Server"
- key: database
  name: "데이터베이스"
```

이 파일은 **표시 이름과 노출 순서만** 정의한다. 어떤 카테고리를 보여줄지는 결정하지 않는다 — 그건 실제 글의 존재 여부가 결정한다.

### 4. `index.md` 재작성

**레이아웃을 `home`에서 `archive`로 바꾼다.** Minimal Mistakes의 `home` 레이아웃은 `{{ content }}` 아래에 전체 포스트 목록을 자동으로 렌더링하므로, 최근 5개를 직접 그리면 같은 글이 두 번 나온다. `archive` 레이아웃은 같은 페이지 껍데기를 주면서 자동 목록만 없앤다.

제거할 것: 상단 `Devlog` 헤더, "개발자 소개" 문단, "모든 포스트 보기" 버튼, "주요 학습 분야" 텍스트 나열. (소개는 왼쪽 `author_profile`이, 사이트 제목은 상단 네비게이션이 이미 담당한다.)

남길 블록은 둘뿐이다.

**최근 포스트** — 최신 5개. 테마의 `archive-single.html` include를 재사용해 목록 스타일을 테마와 일치시킨다.

```liquid
{% for post in site.posts limit: 5 %}
  {% include archive-single.html %}
{% endfor %}
```

**카테고리 둘러보기** — `category_names.yml` 순서대로 순회하되, `site.categories`에 실제로 존재하는 키만 글 수와 함께 출력한다. 매핑에 없는 카테고리는 목록 끝에 원문 이름 그대로 덧붙여, 새 카테고리로 글을 써도 메인에서 누락되지 않게 한다.

```liquid
{% assign known_keys = site.data.category_names | map: "key" %}
<ul class="category-list">
{% for c in site.data.category_names %}
  {% assign posts_in_cat = site.categories[c.key] %}
  {% if posts_in_cat and posts_in_cat.size > 0 %}
  <li><a href="{{ '/categories/' | relative_url }}#{{ c.key | slugify }}">{{ c.name }}</a> ({{ posts_in_cat.size }})</li>
  {% endif %}
{% endfor %}
{% for cat in site.categories %}
  {% unless known_keys contains cat[0] %}
  <li><a href="{{ '/categories/' | relative_url }}#{{ cat[0] | slugify }}">{{ cat[0] }}</a> ({{ cat[1].size }})</li>
  {% endunless %}
{% endfor %}
</ul>
```

Jekyll 3.10 제약상 `push`/`pop` 같은 배열 필터는 쓰지 않는다. `known_keys`는 `map` 필터로 만들고, Liquid 배열의 `contains`는 정확히 일치하는 원소만 찾으므로 부분 문자열 오탐이 없다.

### 5. `_data/navigation.yml` 사이드바 정리

글이 있는 카테고리만 남긴다. 그룹 제목 자체도 해당 상위 카테고리 앵커로 연결한다.

```yaml
study-categories:
  - title: "프로그래밍"
    url: /categories/#programming
    children:
      - title: "C/C++"
        url: /categories/#cpp
  - title: "데이터베이스"
    url: /categories/#database
    children:
      - title: "SQL Server"
        url: /categories/#sql-server
```

`AI/머신러닝` 그룹은 소속된 글이 하나도 없으므로 통째로 제거한다. `main:` 네비게이션(Home / About / Posts)은 그대로 둔다.

사이드바는 정적 YAML이라 자동 생성되지 않는다. **새 분야의 글을 처음 쓸 때 `category_names.yml`과 `navigation.yml`에 각각 한 줄씩 추가한다**를 운영 규칙으로 삼는다.

## 검증

1. `bundle exec jekyll serve`로 로컬 빌드. 로컬에 Ruby/bundler가 없으면 이 단계는 건너뛰고 정적 검토로 대체한다.
2. 메인에 포스트 2개가 제목·날짜·발췌와 함께 표시되는지 확인.
3. 메인 카테고리 블록에 `C/C++ (1)`, `프로그래밍 (1)`, `SQL Server (1)`, `데이터베이스 (1)` 네 항목만 표시되는지 확인. 빈 카테고리 링크가 없어야 한다.
4. 각 카테고리 링크를 눌러 `/categories/` 의 해당 앵커로 실제 이동하고, 그 아래 글이 보이는지 확인.
5. 사이드바의 모든 링크가 실제 앵커로 이동하는지 확인.
6. `/posts/`, `/tags/` 페이지도 함께 채워졌는지 확인.

## 범위 밖

의도적으로 이번 작업에서 제외한다. 각각 별도 작업으로 다루는 편이 깔끔하다.

- `_config.yml`의 `defaults:` 키 중복. 45행 블록이 61행 블록에 덮여 무효이고, 61행 블록의 `path: "_pages"`는 실제 폴더명 `pages/`와 다르다. 현재는 각 파일이 front matter에 직접 값을 명시해 동작에 문제가 없다.
- 포스트 permalink 형식. `permalink` 미설정이라 기본값 `/:categories/:year/:month/:day/:title/`을 쓰는데, 카테고리가 2개인 글은 `/sql-server/database/2024/05/12/...`처럼 경로가 길어진다.
- 저장소 루트의 `README.md`가 아직 GitHub Skills 튜토리얼 템플릿 그대로인 점.
- `.github/workflows/`의 튜토리얼용 워크플로 6개.
