# 기획 문서 조회 Skill 시스템 설계

## 개요

GitHub에 있는 기획 문서 repository를 git pull로 최신화하고, 프로젝트 코드 기반으로 문서를 탐색/조회/질의할 수 있는 Claude Code skill 시스템.

## 제약 조건

- 프로젝트 전용 skill (`.claude/commands/` 하위)
- 단일 git repository만 관리
- 조회 대상: `.md` 파일만 (하위 폴더 포함 재귀 탐색)
- repository 구조: 프로젝트 고유 코드(폴더명) 하위에 `.md` 문서들이 존재
- 문자 인코딩: UTF-8

```
project-docs/
  PRJ-001/
    기획서.md
    요구사항.md
    phase2/
      설계문서.md
  PRJ-002/
    설계문서.md
```

## 파일 구조

```
skilltest/
  .claude/
    docs-config.md          # 공유 config (repo URL, clone 경로)
    commands/
      docs-setup.md         # Skill 1: 설정
      docs-browse.md        # Skill 2: 프로젝트 코드 기반 문서 탐색
      docs-ask.md           # Skill 3: 특정 문서 지정 Q&A
      docs.md               # Skill 4: 통합 진입점 (서브커맨드)
```

## 구성 요소 상세

### 1. `docs-config.md` — 공유 설정 파일

각 skill이 참조하는 공유 설정 파일. `/docs-setup` 실행 시 생성/갱신된다.

**형식:**

```markdown
---
repo_url: https://github.com/org/project-docs.git
clone_path: /{workspace}/project-docs
---
```

- `repo_url`: git repository URL
- `clone_path`: 로컬 clone 경로
- 파일이 존재하지 않거나 값이 비어있으면 "미설정" 상태

### 2. `/docs-setup` — 설정 skill

**역할:** repository URL과 로컬 clone 경로를 설정

**동작 흐름:**
1. 사용자에게 git repo URL 질문
2. 로컬 clone 경로 질문 (기본값 제안 가능)
3. 해당 경로에 clone이 안 되어 있으면 `git clone` 실행
   - clone 실패 시 (인증 오류, URL 오류 등) 에러 메시지를 표시하고 URL/경로 재입력 유도
4. 이미 clone 되어 있으면 `git pull`로 최신화
5. clone 성공 후 repository 내에 프로젝트 코드 폴더 구조가 있는지 간단히 확인 (최상위 폴더 목록 표시)
6. `.claude/docs-config.md`에 설정 저장

### 3. `/docs-browse` — 문서 탐색 skill

**역할:** 프로젝트 코드를 기반으로 하위 문서를 탐색/조회

**사용법:**
- `/docs-browse` — 프로젝트 코드를 대화형으로 질문
- `/docs-browse PRJ-001` — 프로젝트 코드를 인자로 직접 전달

**동작 흐름:**
1. `.claude/docs-config.md` 읽기 → 미설정 시 `/docs-setup` 실행 유도 메시지 출력 후 중단
2. clone 경로에서 `git pull` 실행하여 최신화
   - pull 실패 시 (네트워크 오류 등): 경고 메시지를 표시하되 로컬 데이터로 계속 진행
3. 인자에 프로젝트 코드가 있으면 사용, 없으면 사용자에게 프로젝트 코드 입력 요청 (예: `PRJ-001`)
4. 해당 프로젝트 코드 폴더 하위 `.md` 파일 목록 조회 (하위 폴더 포함 재귀 탐색)
   - 폴더가 없으면 에러 메시지 출력
   - `.md` 파일이 0개 → "해당 프로젝트에 문서가 없습니다" 안내
   - `.md` 파일이 1개 → 바로 내용 읽어서 표시
   - `.md` 파일이 여러 개 → 상대 경로 목록을 보여주고 어떤 문서를 볼지 선택 요청
5. 선택된 문서 내용 출력

### 4. `/docs-ask` — 특정 문서 Q&A skill

**역할:** 특정 프로젝트의 특정 문서를 지정하여 자연어 Q&A

**사용법:**
- `/docs-ask` — 프로젝트 코드, 문서명을 순차적으로 질문
- `/docs-ask PRJ-001` — 프로젝트 코드 전달 → 문서 선택 후 Q&A
- `/docs-ask PRJ-001 기획서.md` — 프로젝트 코드 + 문서명 직접 전달

**동작 흐름:**
1. `.claude/docs-config.md` 읽기 → 미설정 시 `/docs-setup` 실행 유도 메시지 출력 후 중단
2. clone 경로에서 `git pull` 실행하여 최신화
   - pull 실패 시: 경고 메시지를 표시하되 로컬 데이터로 계속 진행
3. 인자에서 프로젝트 코드와 문서명 파싱
   - 인자가 부족하면 순차적으로 질문 (프로젝트 코드 → 문서 선택)
   - 프로젝트 코드만 주어지고 문서가 1개면 자동 선택, 여러 개면 목록 표시 후 선택 요청
4. 해당 문서를 읽어 컨텍스트에 로드
5. "문서를 로드했습니다. 질문을 입력해주세요." 안내 메시지 출력
6. 사용자의 자연어 질문에 문서 내용 기반으로 응답 (대화형 Q&A — 사용자가 다른 주제로 전환할 때까지 계속)

### 5. `/docs` — 통합 진입점

**역할:** 서브커맨드 방식으로 위 3개 skill 기능을 하나로 통합

**사용법:**
- `/docs setup` → docs-setup 동작
- `/docs browse [프로젝트코드]` → docs-browse 동작
- `/docs ask [프로젝트코드] [문서명]` → docs-ask 동작
- `/docs` (인자 없음) → 사용 가능한 명령어 안내

**동작:** 인자의 첫 번째 단어로 분기하여 해당 skill과 동일한 로직을 각 섹션 내에 직접 기술한다 (개별 skill 파일을 참조하는 것이 아니라 독립적으로 동작).

## 공통 규칙

- **config 미설정 감지:** `/docs-browse`, `/docs-ask`, `/docs` (browse/ask) 실행 시 `.claude/docs-config.md`가 없거나 값이 비어있으면 `/docs-setup` 실행을 안내하고 중단
- **git pull:** 문서 조회 전 항상 `git pull`로 최신화. 실패 시 경고 후 로컬 데이터로 진행
- **에러 처리:** 프로젝트 코드 폴더 미존재, 문서 미존재, 문서 0개 시 명확한 안내 메시지 출력
- **파일 경로:** 하위 폴더 포함 재귀 탐색. 문서 목록/지정 시 프로젝트 코드 폴더 기준 상대 경로 사용 (예: `phase2/설계문서.md`)
