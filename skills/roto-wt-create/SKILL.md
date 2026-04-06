---
name: roto-wt-create
description: origin/main 기준으로 새로운 git worktree를 생성합니다. worktrunk(wt) CLI를 사용합니다. "워크트리 생성", "새 브랜치", "wt create" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# Worktree 생성

새로운 git worktree를 origin/main 기준으로 생성합니다. worktrunk(`wt`) CLI를 사용합니다.

## 사용 방법

```text
/roto-wt-create [브랜치명]
```

## 전제 조건

- worktrunk CLI 설치 필요 (`brew install worktrunk`)
- shell integration 설정 필요 (`wt config shell install`)

## 워크플로우

### 1. 인자 파싱

사용자가 제공한 브랜치명을 추출합니다. 없으면 질문합니다.

### 2. worktrunk으로 worktree 생성

```bash
wt switch --create <브랜치명> --yes
```

이 명령은 자동으로:
- origin/main 기준으로 새 브랜치 생성
- 설정된 경로 템플릿에 따라 worktree 디렉토리 생성
- `post-create` hooks 실행 (.env 파일 복사, yarn install 등)
- 생성된 worktree 디렉토리로 자동 이동

### 3. 결과 보고

```
Worktree 생성 완료:
- 브랜치: {BRANCH_NAME}
- 기준: origin/main
```

마지막으로 전체 worktree 목록을 보여줍니다:

```bash
wt list
```
