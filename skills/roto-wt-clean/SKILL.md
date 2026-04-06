---
name: roto-wt-clean
description: git worktree를 정리(삭제)합니다. 현재 worktree 또는 지정한 worktree를 제거합니다. "워크트리 삭제", "워크트리 정리", "wt clean" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# Worktree 정리

git worktree를 삭제하고 관련 브랜치를 정리합니다.

## 사용 방법

```text
/roto-wt-clean              # 현재 worktree 정리 (메인 repo에서 실행 시 목록에서 선택)
/roto-wt-clean [브랜치명]    # 특정 worktree 정리
```

## 워크플로우

### 1. 현재 상태 파악

```bash
MAIN_REPO=$(git worktree list | head -1 | awk '{print $1}')
CURRENT_DIR=$(pwd)
```

- `CURRENT_DIR == MAIN_REPO` → 메인 repo에서 실행 중
- `CURRENT_DIR != MAIN_REPO` → worktree에서 실행 중

### 2. 삭제 대상 결정

- **인자가 있는 경우:** 해당 브랜치명의 worktree를 찾습니다.
- **인자가 없고 worktree에서 실행 중:** 현재 worktree가 대상.
- **인자가 없고 메인 repo에서 실행 중:** 목록에서 선택.

**중요:** 메인 repo 자체는 삭제 대상에서 제외합니다.

### 3. 안전 확인

```bash
cd "$TARGET_WT_DIR" && git status --porcelain
```

커밋되지 않은 변경사항이 있으면 사용자에게 경고하고 확인을 받습니다.

### 4. Worktree 삭제

```bash
cd "$MAIN_REPO"
git worktree remove "$TARGET_WT_DIR"
```

커밋되지 않은 변경이 있어 실패 시, 사용자 확인 후:
```bash
git worktree remove --force "$TARGET_WT_DIR"
```

### 5. 브랜치 삭제 여부 확인

사용자에게 관련 브랜치도 삭제할지 물어봅니다.

### 6. 결과 보고

```
Worktree 정리 완료:
- 삭제된 경로: {TARGET_WT_DIR}
- 브랜치: {삭제됨 / 유지됨}
```
