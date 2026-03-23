---
description: 기획 문서 repository 설정 (git repo URL, clone 경로)
---

# docs-setup: 기획 문서 Repository 설정

이 skill은 기획 문서가 저장된 git repository를 설정합니다.

## 실행 절차

아래 절차를 순서대로 수행하세요.

### 1단계: 기존 설정 확인

`.claude/docs-config.md` 파일을 읽어 현재 설정 상태를 확인합니다.
- `repo_url`과 `clone_path`가 이미 설정되어 있으면 현재 설정을 보여주고, 변경할지 물어봅니다.
- 값이 비어있으면 새로 설정을 진행합니다.

### 2단계: 로컬 경로 확인

먼저 사용자에게 이미 clone된 로컬 경로가 있는지 물어봅니다.

> 기획 문서 repository가 이미 로컬에 clone 되어 있나요?
> - **있다면**: clone된 경로를 입력해주세요. (예: `/{workspace}/project-docs`)
> - **없다면**: `없음`이라고 입력해주세요.

**이미 clone 되어있는 경우:**
- 입력받은 경로가 git repository인지 확인합니다 (`.git` 폴더 존재 여부).
  - git repository가 맞으면 → `git remote -v`로 repo URL을 자동으로 읽어옵니다 → **4단계로 이동**
  - git repository가 아니면 → "해당 경로는 git repository가 아닙니다. 경로를 다시 확인해주세요." 안내 후 재입력 유도

**clone 되어있지 않은 경우 → 3단계로 진행**

### 3단계: 새로 Clone

사용자에게 git repository URL과 clone할 경로를 물어봅니다.

> 기획 문서가 저장된 git repository URL을 입력해주세요.
> 예: `https://github.com/org/project-docs.git`

> 로컬에 clone할 경로를 입력해주세요.
> 기본값: `~/project-docs`

입력받은 정보로 `git clone <repo_url> <clone_path>`를 실행합니다.
- 실패 시 에러 메시지를 표시하고 URL과 경로를 다시 확인하도록 안내

### 4단계: Git Pull (최신화)

clone 경로에서 `git pull`을 실행하여 최신 상태로 업데이트합니다.

### 5단계: Repository 구조 확인

clone된 경로의 최상위 폴더 목록을 표시하여 프로젝트 코드 폴더들이 있는지 사용자가 확인할 수 있게 합니다.

> Repository가 설정되었습니다. 다음 프로젝트 폴더들이 확인됩니다:
> - PRJ-001
> - PRJ-002
> - ...

### 6단계: 설정 저장

`.claude/docs-config.md` 파일을 아래 형식으로 저장합니다:

```markdown
---
repo_url: <입력받은 URL>
clone_path: <입력받은 경로>
---
```

설정 완료 후 안내:

> 설정이 완료되었습니다. `/docs-browse` 또는 `/docs-ask`로 문서를 조회할 수 있습니다.
