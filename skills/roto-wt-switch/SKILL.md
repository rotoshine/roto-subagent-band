---
name: roto-wt-switch
description: 사용 가능한 git worktree 목록을 보여주고 전환합니다. "워크트리 전환", "워크트리 목록", "wt switch" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# Worktree 전환

현재 존재하는 worktree 목록을 보여주고, 선택한 worktree로 전환합니다.

## 사용 방법

```text
/roto-wt-switch              # worktree 목록 표시 후 선택
/roto-wt-switch [브랜치명]    # 특정 worktree로 바로 전환
```

## 워크플로우

### 1. 현재 상태 파악

```bash
MAIN_REPO=$(git worktree list | head -1 | awk '{print $1}')
CURRENT_DIR=$(pwd)
git worktree list
```

### 2. Worktree 목록 표시

```
현재 Worktree 목록:

  1. /path/to/project           [main]              ← 메인 repo
  2. /path/to/project-quest     [quest]
* 3. /path/to/project.fix-...   [fix/search_console] ← 현재 위치
```

### 3. 대상 결정

- **인자가 있는 경우:** 해당 브랜치명의 worktree를 찾습니다.
- **인자가 없는 경우:** 사용자에게 번호 또는 브랜치명으로 선택하게 합니다.

### 4. 전환 실행

```bash
cd "$TARGET_WT_DIR"
pwd
git branch --show-current
```

**참고:** Claude Code 세션 내에서 cd로 이동하면 이후 모든 명령이 해당 디렉토리 기준으로 실행됩니다.
