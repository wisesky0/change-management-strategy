좋습니다! 요청하신 대로 첨부된 문서를 다음 **새로운 목차 구성**에 따라 재구성하고, 부족한 부분은 보완하여 하나의 완성된 가이드로 출력하겠습니다.

---

# 🧭 Java 프로젝트 변경 이력 및 버전 관리 전략 가이드

**환경**: Git + GitLab + Maven + jgitver + commit-and-tag-version 기반  
**특징**: 릴리즈는 GitLab CI와 연계되며, 버전은 자동으로 채번되고 CHANGELOG는 Conventional Commits 기반으로 생성됩니다.

---

## 📌 목차

1. 전략 개요
2. 브랜치 전략
3. 버전 관리 설정 (jgitver)
4. 커밋 메시지 규칙 (Conventional Commits)
5. 릴리즈 흐름
    - 기능 개발
    - 릴리즈 브랜치 준비
    - CHANGELOG 생성 및 태깅
    - 릴리즈 확정
    - 다음 개발 시작
6. GitLab CI/CD 설정 요약
7. 운영 배포 전략 (Maven 기반)
8. commit-and-tag-version 사용법
9. 주의사항 및 전략적 이점

---

## 1. 전략 개요

| 항목 | 사용 방식 |
|------|-----------|
| Git 호스팅 | GitLab |
| 브랜치 전략 | `master`, `feature/*`, `release/*` (버전 포함 ❌) |
| 버전 관리 | [`jgitver-maven-plugin`](https://github.com/jgitver/jgitver) |
| 커밋 포맷 | [Conventional Commits](https://www.conventionalcommits.org/) |
| 릴리즈 자동화 | [`commit-and-tag-version`](https://github.com/absolute-version/commit-and-tag-version) 기반 changelog + 태깅 |

---

## 2. 브랜치 전략

| 브랜치 | 목적 | 버전 형태 |
|--------|------|-----------|
| `feature/*` | 기능 개발 | `1.3.1-feature-abc-SNAPSHOT` |
| `release/*` | QA 및 릴리즈 준비 | `1.3.1-release-next-SNAPSHOT` |
| `master` | 운영 반영 | `1.3.1` (릴리즈 확정 태그 기준) |

---

## 3. 버전 관리 설정 (jgitver)

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

## 4. 커밋 메시지 규칙 (Conventional Commits)

커밋 메시지를 구조화하여 자동 버전 증가 및 CHANGELOG 생성을 지원합니다.

### 형식

```bash
<type>(<scope>): <subject>
```

### 주요 타입

| type | 설명 | 버전 영향 |
|------|------|------------|
| feat | 기능 추가 | Minor 버전 증가 |
| fix | 버그 수정 | Patch 버전 증가 |
| chore, docs, style, refactor | 내부 변경 | 없음 |

### 예시

```bash
feat(auth): 로그인 기능 추가
fix(api): 예외 처리 개선
```

### Breaking Change

```bash
feat(auth): 토큰 기반 인증 변경

BREAKING CHANGE: 기존 세션 기반 인증 제거됨
```

---

## 5. 릴리즈 흐름

### 1) 기능 개발

```bash
git checkout -b feature/로그인개선 master
```

- SNAPSHOT 버전 자동 적용됨

---

### 2) 릴리즈 브랜치 준비

GitLab에서 `release/next` 브랜치 생성 후 MR로 여러 기능 병합

---

### 3) CHANGELOG 생성 및 태깅

GitLab UI에서 수동 실행 (when: manual)

```yaml
release_prepare:
  stage: release
  script:
    - git fetch origin
    - git checkout $CI_COMMIT_REF_NAME
    - git merge --no-ff origin/master
    - ./mvnw clean verify
    - npx commit-and-tag-version
    - git push origin HEAD --follow-tags
  only:
    - /^release\\/.*$/
  when: manual
```

---

### 4) 릴리즈 확정

태그 커밋이 포함된 release 브랜치를 master로 자동 병합

```yaml
release_finalize:
  stage: release
  script:
    - git checkout master
    - git merge --no-ff origin/$CI_COMMIT_REF_NAME
    - git push origin master --follow-tags
  needs: [release_prepare]
  only:
    - /^release\\/.*$/
```

---

### 5) 다음 개발 시작

```bash
git checkout -b feature/다음기능 master
```

---

## 6. GitLab CI/CD 전체 구성 요약

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

release_prepare: ...

release_finalize: ...

deploy_production:
  stage: deploy
  script:
    - ./mvnw clean deploy -Prelease -DskipTests
  only:
    - master
    - tags
  # 릴리즈 커밋 포함된 태그를 기준으로 Maven 배포 수행
```

---

## 7. 운영 배포 전략 (Maven 기반)

### 전제 조건

- `pom.xml` 내 `distributionManagement` 설정
- `settings.xml` 내 `<server>` 인증 정보
- 필요시 GPG 서명 또는 staging 설정

### 배포 명령

```bash
./mvnw clean deploy -Prelease -DskipTests
```

---

## 8. commit-and-tag-version 사용법

- Conventional Commits 기반 changelog 자동 생성 및 태깅 도구
- 릴리즈 자동화 핵심 도구

```bash
npx commit-and-tag-version
```

`.versionrc.json`으로 섹션/태그 포맷 등 커스터마이징 가능

---

## 9. 주의사항 및 전략적 이점

### 주의사항

- `commit-and-tag-version`은 병합 가능 검증 이후 실행할 것
- `release/*` 브랜치에는 버전명 포함하지 않음
- GitLab MR은 병합 전 기준으로 테스트됨

### 전략적 이점

| 항목 | 효과 |
|------|------|
| 자동 버전 관리 | 수동 실수 방지 |
| 커밋 기반 changelog | 문서화 자동화 |
| 병합 기준 버전 확정 | 릴리즈 안정성 확보 |
| GitLab CI 연계 | 운영 배포 자동화 |

---

원하신다면 이 내용을 `.md` 파일로 정리해드릴게요. 진행할까요?