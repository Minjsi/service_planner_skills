# 기획 문서 조회 Skill 시스템 설계서

## 1. 개요

GitHub에 있는 기획 문서 repository를 git pull로 최신화하고, 프로젝트 고유 코드(폴더명) 기반으로 문서를 탐색/조회/질의할 수 있는 Claude Code slash command 시스템입니다.

### 1.1 목적

- 기획 문서가 관리되는 git repository를 로컬에 연결
- 프로젝트 코드를 기반으로 해당 프로젝트의 문서를 빠르게 조회
- 특정 문서에 대해 자연어로 Q&A 수행

### 1.2 제약 조건

| 항목 | 내용 |
|------|------|
| 적용 범위 | 이 프로젝트(`skilltest`) 전용 |
| 대상 repository | 단일 git repository |
| 조회 대상 파일 | `.md` (마크다운) 파일만 |
| 문자 인코딩 | UTF-8 |
| 탐색 범위 | 프로젝트 코드 폴더 하위 재귀 탐색 |

### 1.3 전제: Repository 폴더 구조

```
project-docs/          ← git repository root
  PRJ-001/             ← 프로젝트 고유 코드 (폴더명)
    기획서.md
    요구사항.md
    phase2/
      설계문서.md
  PRJ-002/
    설계문서.md
```

---

## 2. 시스템 구성

### 2.1 파일 구조

```
skilltest/
  .claude/
    docs-config.md                # 공유 설정 파일
    commands/
      docs-setup.md               # /docs-setup  — repository 설정
      docs-browse.md              # /docs-browse — 문서 탐색
      docs-ask.md                 # /docs-ask    — 문서 Q&A
      docs.md                     # /docs        — 통합 진입점
```

### 2.2 구성 요소 관계

```
┌─────────────────────────────────────────────────┐
│                .claude/docs-config.md            │
│          (repo_url, clone_path 저장)             │
└──────────┬──────────┬──────────┬────────────────┘
           │          │          │
     ┌─────▼──┐  ┌────▼───┐  ┌──▼──────┐
     │ setup  │  │ browse │  │  ask    │
     │ (설정) │  │ (탐색) │  │ (Q&A)  │
     └────────┘  └────────┘  └─────────┘
           │          │          │
           └──────────┼──────────┘
                      │
              ┌───────▼───────┐
              │  /docs (통합) │
              └───────────────┘
```

- **docs-config.md**: 모든 skill이 참조하는 공유 설정
- **개별 skill 3개**: 각각 독립적으로 호출 가능
- **통합 skill 1개**: 서브커맨드 방식으로 3개 기능을 하나로 제공

---

## 3. 공유 설정 파일 (`docs-config.md`)

### 3.1 형식

```markdown
---
repo_url: https://github.com/org/project-docs.git
clone_path: /{workspace}/project-docs
---
```

### 3.2 상태 판단

| 상태 | 조건 |
|------|------|
| 미설정 | 파일 미존재, 또는 `repo_url`/`clone_path` 값이 비어있음 |
| 설정됨 | 두 값이 모두 존재 |

---

## 4. Skill 상세

### 4.1 `/docs-setup` — Repository 설정

| 항목 | 내용 |
|------|------|
| 파일 | `.claude/commands/docs-setup.md` |
| 역할 | 로컬 clone 경로 확인 또는 신규 clone, repo URL 설정 |
| 인자 | 없음 (대화형) |

**동작 흐름:**

```
기존 설정 확인
  ├─ 설정 있음 → 현재 설정 표시, 변경 여부 확인
  └─ 설정 없음 → 새로 설정 진행
       ↓
로컬 경로 확인 (이미 clone 되어있는지?)
  ├─ clone 있음 → 경로 입력 → git repo 확인 → remote URL 자동 추출
  └─ clone 없음 → repo URL 입력 → clone 경로 입력 → git clone
       ↓
git pull (최신화)
       ↓
최상위 폴더 목록 표시
       ↓
docs-config.md 저장
```

**설정 흐름 상세:**

1. **기존 설정 확인**: `docs-config.md`를 읽어 이미 설정된 값이 있으면 표시하고 변경 여부를 확인
2. **로컬 경로 확인**: 사용자에게 이미 clone된 로컬 경로가 있는지 물어봄
   - **있는 경우**: 경로를 입력받아 `.git` 폴더 존재 여부로 git repository인지 확인. git repository가 맞으면 `git remote -v`로 repo URL을 자동 추출하여 config에 반영
   - **없는 경우**: git repository URL과 clone 경로를 각각 입력받아 `git clone` 실행
3. **Git Pull**: clone 경로에서 `git pull`로 최신화
4. **구조 확인**: 최상위 폴더 목록을 표시하여 프로젝트 코드 폴더 확인
5. **저장**: `docs-config.md`에 `repo_url`, `clone_path` 저장

### 4.2 `/docs-browse` — 문서 탐색

| 항목 | 내용 |
|------|------|
| 파일 | `.claude/commands/docs-browse.md` |
| 역할 | 프로젝트 코드 기반 `.md` 문서 목록 조회 및 내용 표시 |
| 인자 | `$ARGUMENTS` — 프로젝트 코드 (선택) |

**동작 흐름:**

```
config 확인
  └─ 미설정 → "/docs-setup 실행 안내" 후 중단

clone 경로 존재 확인
  └─ 미존재 → "/docs-setup 재실행 안내" 후 중단

git pull (실패 시 경고 후 로컬로 진행)
       ↓
프로젝트 코드 확인 ($ARGUMENTS 또는 대화형 입력)
       ↓
해당 폴더 하위 .md 파일 재귀 탐색
  ├─ 폴더 없음  → 에러
  ├─ 파일 0개   → "문서 없음" 안내
  ├─ 파일 1개   → 바로 내용 표시
  └─ 파일 여러개 → 목록 표시 → 선택 요청 → 내용 표시
```

### 4.3 `/docs-ask` — 문서 Q&A

| 항목 | 내용 |
|------|------|
| 파일 | `.claude/commands/docs-ask.md` |
| 역할 | 특정 문서를 로드하여 자연어 Q&A |
| 인자 | `$ARGUMENTS` — 프로젝트 코드 (필수), 문서명 (선택) |

**동작 흐름:**

```
config 확인
  └─ 미설정 → "/docs-setup 실행 안내" 후 중단

clone 경로 존재 확인
  └─ 미존재 → "/docs-setup 재실행 안내" 후 중단

git pull (실패 시 경고 후 로컬로 진행)
       ↓
프로젝트 코드 확인 ($ARGUMENTS 또는 대화형 입력)
       ↓
문서 선택
  ├─ 인자에 문서명 있음 → 파일 존재 확인 (없으면 에러)
  └─ 인자에 문서명 없음
       ├─ 파일 0개   → "문서 없음" 안내 후 중단
       ├─ 파일 1개   → 자동 선택
       └─ 파일 여러개 → 목록 표시 → 선택 요청
       ↓
문서 로드 → "질문을 입력해주세요" 안내
       ↓
자연어 Q&A (문서 내용 기반, 대화형으로 계속)
```

### 4.4 `/docs` — 통합 진입점

| 항목 | 내용 |
|------|------|
| 파일 | `.claude/commands/docs.md` |
| 역할 | 서브커맨드 방식으로 setup/browse/ask 기능 통합 제공 |
| 인자 | `$ARGUMENTS` — 서브커맨드 + 추가 인자 |

**서브커맨드 분기:**

| 명령어 | 동작 |
|--------|------|
| `/docs setup` | `/docs-setup`과 동일 |
| `/docs browse [코드]` | `/docs-browse`와 동일 |
| `/docs ask [코드] [문서명]` | `/docs-ask`와 동일 |
| `/docs` (인자 없음) | 사용 가능한 명령어 도움말 표시 |

통합 skill은 개별 skill을 참조하지 않고 각 섹션 내에 동일한 로직을 독립적으로 기술합니다.

---

## 5. 공통 규칙

### 5.1 Config 미설정 시

모든 조회/Q&A skill에서 `docs-config.md`가 미설정 상태이면 `/docs-setup` 실행을 안내하고 즉시 중단합니다.

### 5.2 Clone 경로 미존재 시

config에 `clone_path`가 설정되어 있지만 해당 디렉토리가 없으면 `/docs-setup` 재실행을 안내하고 즉시 중단합니다.

### 5.3 Git Pull

문서 조회 전 항상 `git pull`로 최신화합니다. 네트워크 오류 등으로 실패 시 경고 메시지를 표시하고 로컬 데이터로 계속 진행합니다.

### 5.4 에러 메시지

| 상황 | 메시지 |
|------|--------|
| 폴더 미존재 | `{프로젝트코드}` 폴더를 찾을 수 없습니다. 프로젝트 코드를 확인해주세요. |
| 문서 0개 | `{프로젝트코드}` 프로젝트에 문서(.md)가 없습니다. |
| 문서 미존재 | `{문서명}` 파일을 찾을 수 없습니다. 파일명을 확인해주세요. |
| git pull 실패 | ⚠ git pull에 실패했습니다. 로컬에 저장된 버전으로 진행합니다. |
| clone 경로 없음 | clone 경로(`{clone_path}`)가 존재하지 않습니다. `/docs-setup`을 다시 실행하여 설정해주세요. |

---

## 6. 사용 예시

```bash
# 1-a. 최초 설정 (이미 clone 되어있는 경우)
/docs-setup
# → "이미 clone 있음" → 경로 입력 → remote URL 자동 추출 → git pull → config 저장

# 1-b. 최초 설정 (새로 clone 하는 경우)
/docs-setup
# → "없음" → repo URL 입력 → clone 경로 입력 → git clone → config 저장

# 2. 문서 탐색
/docs-browse PRJ-001
# → git pull → PRJ-001/ 하위 .md 파일 목록 → 선택 → 내용 표시

# 3. 문서 Q&A
/docs-ask PRJ-001 기획서.md
# → git pull → 기획서.md 로드 → 자연어 Q&A 시작

# 4. 통합 명령어
/docs browse PRJ-001
/docs ask PRJ-001
/docs setup
```
