

# 🧭 Java 프로젝트 변경 이력 및 버전 관리 전략 가이드 (v6)

**환경: Git + GitLab + Maven + jgitver + commit-and-tag-version 기반 (릴리즈 태그는 병합 가능성 확인 후 생성)**

---

## ✅ 전략 개요

| 항목 | 사용 방식 |
|------|-----------|
| Git 호스팅 | GitLab |
| 브랜치 전략 | `master`, `feature/*`, `release/*` (버전 포함 ❌) |
| 버전 관리 | [`jgitver-maven-plugin`](https://github.com/jgitver/jgitver) 기반 자동 버전 채번 |
| 커밋 포맷 | [Conventional Commits](https://www.conventionalcommits.org/) |
| 릴리즈 자동화 | [`commit-and-tag-version`](https://github.com/absolute-version/commit-and-tag-version) 기반 changelog 생성 및 버전 태깅 자동화 |

---

## 📂 브랜치 전략

| 브랜치 | 목적 | 버전 형태 |
|--------|------|-----------|
| `feature/*` | 기능 단위 개발 | `1.3.1-feature-abc-SNAPSHOT` |
| `release/*` | 릴리즈 후보 준비, QA | `1.3.1-release-next-SNAPSHOT` |
| `master` | 운영 반영, 태그 대상 | `1.3.1` (릴리즈 확정 시점) |

---

## 🔢 버전 생성 방식 (jgitver 기준)

```yaml
# jgitver.config.yml
policy: SIMPLE
nonQualifierBranches:
  - master
useDistance: true
useGitCommitId: false
useDirty: true
autoIncrementPatch: true
mavenLike: true
regexVersionTag: ^   # 숫자 태그만 인식, v 접두사 없음
```

---

## 🔁 버전 흐름 예시

| Git 상태 | 생성 버전 |
|----------|-----------|
| `feature/123-login` | `1.3.1-feature-123-login-SNAPSHOT` |
| `release/next` | `1.3.1-release-next-SNAPSHOT` |
| `master` (태그: 1.3.1) | `1.3.1` |

---

## ✍️ 커밋 메시지 규칙 (Conventional Commits)

`Conventional Commits`는 커밋 메시지를 일관되게 작성하기 위한 규칙이며, **자동 버전 증가**, **CHANGELOG 생성**, **CI 릴리즈 자동화**를 위한 핵심 기반입니다.

### ✅ 기본 형식

```bash
<type>(<scope>): <subject>
```

| 구성 요소 | 설명 |
|-----------|------|
| `type` | 커밋의 목적 (예: `feat`, `fix`, `docs`, `refactor`, `chore` 등) |
| `scope` _(선택)_ | 영향을 받는 범위 또는 모듈 이름 (예: `auth`, `api`, `ui`) |
| `subject` | 간결한 변경 요약 (소문자로 시작하며 마침표 없이 작성) |

---

### ✅ 예시

```bash
feat(auth): 로그인 2FA 인증 추가 (#123)
fix(api): 응답 코드 수정 오류 해결 (#124)
docs(readme): 프로젝트 소개 갱신
refactor(service): 중복 코드 제거
chore(ci): GitLab 파이프라인 병렬 처리 추가
```

---

### 🧠 주요 `type` 목록

| type | 설명 | 버전 영향 |
|------|------|------------|
| `feat` | 새로운 기능 추가 | 🔼 minor (x.**y**.z) 증가 |
| `fix` | 버그 수정 | 🔼 patch (x.y.**z**) 증가 |
| `docs` | 문서 변경 | 없음 |
| `style` | 코드 스타일 변경 (기능 영향 없음) | 없음 |
| `refactor` | 리팩토링 (동작 변화 없음) | 없음 |
| `test` | 테스트 코드 추가/수정 | 없음 |
| `chore` | 빌드, 도구 설정 등 기타 변경 | 없음 |

---

### 🚨 Breaking Change 선언

기능이 하위 호환되지 않게 변경될 경우, 반드시 `BREAKING CHANGE:`를 **본문 또는 footer**에 명시해야 합니다.

```bash
feat(auth): 로그인 인증 구조 변경

BREAKING CHANGE: 인증 로직이 변경되어 기존 토큰이 호환되지 않습니다.
```

→ **자동으로 major 버전 (x.0.0) 증가**

---

### ✅ 자동화 연동 예

- `commit-and-tag-version`, `semantic-release` 같은 도구는 커밋 메시지를 분석하여:
  - 버전을 자동 증가시키고
  - `CHANGELOG.md`를 생성하며
  - 릴리즈 커밋 및 Git 태그를 만듭니다.

---

### 🧩 실전 팁

- 모든 커밋은 릴리즈 대상이 아니므로, 릴리즈 대상은 `feat:`와 `fix:` 위주로 작성하세요.
- GitLab MR 제목과 squash 커밋도 Conventional Commits 포맷을 따르게 설정하는 것을 추천합니다.

---

계속해서 **3단계부터 마지막 appendix(부록)** 까지 전체 출력해드리겠습니다.

---

### 3. 릴리즈 사전 검증 + CHANGELOG 생성 및 태깅 (GitLab 사용자 이벤트 기반)

사용자가 GitLab UI 또는 API로 `release/*` 브랜치에 대해 "릴리즈 확정" 버튼을 누르거나 Webhook으로 호출할 경우, GitLab CI가 자동으로 다음 단계를 수행합니다:

1. `master`와 병합 가능한지 확인  
2. 충돌이 없고 빌드 성공 시 `commit-and-tag-version` 실행

#### GitLab CI 예시

```yaml
release_prepare:
  stage: release
  script:
    - git fetch origin
    - git checkout $CI_COMMIT_REF_NAME
    - git merge --no-commit --no-ff origin/master
    - ./mvnw clean verify
    - npx commit-and-tag-version
    - git push origin HEAD --follow-tags
  only:
    - /^release\\/.*$/
  when: manual
  allow_failure: false
```

> - `when: manual` 설정으로 GitLab UI에서 릴리즈 작업을 수동 트리거 가능  
> - `commit-and-tag-version` 실행 전에 충돌 및 테스트 실패 시 병합 중단

---

### 4. 릴리즈 확정 (자동 연계)

3단계에서 CI를 통해 changelog 및 태그 생성이 완료되면, GitLab CI는 `master` 브랜치로 자동 병합하거나 후속 작업으로 릴리즈 확정을 이어서 실행할 수 있습니다.

#### GitLab CI 예시 (자동 병합)

```yaml
release_finalize:
  stage: release
  script:
    - git config user.email "ci@example.com"
    - git config user.name "CI Bot"
    - git checkout master
    - git merge --no-ff origin/$CI_COMMIT_REF_NAME
    - git push origin master --follow-tags
  only:
    - /^release\\/.*$/
  needs: [release_prepare]
  when: on_success
```

> - 위 job은 changelog/tagging 성공 시 자동으로 master로 병합  
> - `needs:` 키워드를 이용해 이전 단계와 연계

---

### 5. 다음 개발 시작

```bash
git checkout -b feature/next-feature master
```

---

## ⚙️ GitLab CI/CD 전체 구성 요약

```yaml
stages:
  - test
  - release
  - deploy

unit-test:
  stage: test
  script: ./mvnw clean test
  only:
    - merge_requests

release_prepare:
  stage: release
  script:
    - git fetch origin
    - git checkout $CI_COMMIT_REF_NAME
    - git merge --no-commit --no-ff origin/master
    - ./mvnw clean verify
    - npx commit-and-tag-version
    - git push origin HEAD --follow-tags
  only:
    - /^release\\/.*$/
  when: manual
  allow_failure: false

release_finalize:
  stage: release
  script:
    - git config user.email "ci@example.com"
    - git config user.name "CI Bot"
    - git checkout master
    - git merge --no-ff origin/$CI_COMMIT_REF_NAME
    - git push origin master --follow-tags
  only:
    - /^release\\/.*$/
  needs: [release_prepare]
  when: on_success

deploy_production:
  stage: deploy
  script:
    - ./mvnw clean deploy -DskipTests
  only:
    - master
    - tags
```


## 📦 commit-and-tag-version 의 주요 특징과 사용법

[`commit-and-tag-version`](https://github.com/absolute-version/commit-and-tag-version)은 Conventional Commits 기반으로 버전을 자동으로 올리고, changelog 작성, Git 커밋/태그 생성까지 자동 처리해주는 도구입니다.

### ✅ 주요 특징

- 커밋 메시지(`feat:`, `fix:` 등) 분석 → 자동 버전 증가  
- `CHANGELOG.md` 자동 작성  
- Git 태그 + 릴리즈 커밋 자동 생성 (`chore(release): 1.4.0`)  
- `npx` 명령어로 간편 실행  
- 사전/사후 스크립트 훅 지원 (`scripts.preversion`, `scripts.postversion`)  
- `.versionrc.json` 파일로 커스터마이징 가능  

---

### 🛠️ 사용법

```bash
npx commit-and-tag-version
```

---

### ✍️ `.versionrc.json` 예시

```json
{
  "types": [
    { "type": "feat", "section": "✨ Features" },
    { "type": "fix", "section": "🐛 Bug Fixes" },
    { "type": "chore", "hidden": true },
    { "type": "docs", "section": "📝 Documentation" }
  ],
  "skip": {
    "tag": false,
    "commit": false
  },
  "tagPrefix": ""
}
```

---

### 🔁 버전 증가 기준

| 커밋 메시지 | 버전 증가 |
|-------------|------------|
| `fix:`      | Patch (x.y.**z**) |
| `feat:`     | Minor (x.**y**.z) |
| `BREAKING CHANGE:` | Major (**x**.y.z) |

---

### ✅ CI 연동 예시

```yaml
release:
  stage: release
  script:
    - npx commit-and-tag-version
    - git push origin HEAD --follow-tags
  only:
    - master
```

> `standard-version`이나 `semantic-release`보다 간단하고 CI/CD에 바로 쓰기 좋음

---

📦 전체 내용 확인을 마치셨으면, 이 내용을 최종 파일로 다시 만들어드릴 수 있습니다. 원하시나요?