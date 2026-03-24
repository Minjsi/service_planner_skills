# Service Planner Skills

기획 문서 repository를 Claude Code에서 탐색/조회/Q&A 할 수 있는 slash command 스킬 세트입니다.

## 목적

- 기획 문서가 관리되는 git repository를 로컬에 연결
- 프로젝트 코드(폴더명) 기반으로 해당 프로젝트의 문서를 빠르게 조회
- 특정 문서에 대해 자연어 Q&A 수행

## 제공 스킬

| 명령어 | 설명 |
|--------|------|
| `/docs setup` | 기획 문서 git repository 설정 (clone/연결) |
| `/docs browse [프로젝트코드]` | 프로젝트 코드 기반 문서 탐색 및 조회 |
| `/docs ask [프로젝트코드] [문서명]` | 특정 문서를 로드하여 자연어 Q&A |
| `/docs` | 사용 가능한 명령어 도움말 |

개별 명령어로도 사용 가능합니다: `/docs-setup`, `/docs-browse`, `/docs-ask`

## 프로젝트 구조

```
service_planner_skills/
  .claude/
    docs-config.md              # 공유 설정 파일 (repo URL, clone 경로)
    commands/
      docs.md                   # /docs — 통합 진입점
      docs-setup.md             # /docs-setup — repository 설정
      docs-browse.md            # /docs-browse — 문서 탐색
      docs-ask.md               # /docs-ask — 문서 Q&A
  docs/                         # 스킬 시스템 설계 문서
  samples/                      # 테스트용 샘플 기획 문서
```

## 사용 방법

### 1. 타 프로젝트에 스킬 설치

대상 프로젝트의 `.claude/` 디렉토리에 스킬 파일을 복사합니다.

```bash
# commands 디렉토리가 없으면 생성
mkdir -p /path/to/your-project/.claude/commands

# 스킬 파일 복사 (4개)
cp .claude/commands/docs.md        /path/to/your-project/.claude/commands/
cp .claude/commands/docs-setup.md  /path/to/your-project/.claude/commands/
cp .claude/commands/docs-browse.md /path/to/your-project/.claude/commands/
cp .claude/commands/docs-ask.md    /path/to/your-project/.claude/commands/

# 설정 파일 복사
cp .claude/docs-config.md          /path/to/your-project/.claude/
```

### 2. 기획 문서 repository 연결

대상 프로젝트에서 Claude Code를 실행한 뒤:

```
/docs setup
```

대화형으로 기획 문서 git repository URL과 로컬 clone 경로를 설정합니다.

- 이미 clone된 로컬 경로가 있으면 해당 경로를 입력
- 없으면 repo URL을 입력하여 새로 clone

### 3. 문서 조회

```bash
# 프로젝트 문서 목록 확인
/docs browse PRJ-001

# 특정 문서에 대해 Q&A
/docs ask PRJ-001 기획서.md
```

## 기획 문서 repository 폴더 규칙

기획 문서 repository는 아래 구조를 따라야 합니다:

```
project-docs/                 ← git repository root
  PRJ-001/                    ← 프로젝트 고유 코드 (폴더명)
    기획서.md
    요구사항.md
    phase2/                   ← 하위 폴더 가능
      설계문서.md
  PRJ-002/
    설계문서.md
```

- 최상위에 **프로젝트 코드를 폴더명**으로 생성
- 폴더 안에 `.md` (마크다운) 파일로 문서 작성
- 하위 폴더 재귀 탐색 지원

## 샘플 문서로 테스트

이 레포의 `samples/` 디렉토리에 테스트용 기획 문서가 포함되어 있습니다.

```
samples/
  S04-5076/
    해피봇(챗봇) 마이그레이션 정책.md
```

`/docs setup` 시 clone 경로로 이 레포의 `samples/` 경로를 지정하면 바로 테스트할 수 있습니다.
