---
name: roto-band-self-dev
description: PR의 코드를 7개 전문 에이전트로 리뷰한 뒤, 사용자와 대화하며 직접 수정까지 수행하는 자기 개발(self-dev) 스킬. GitHub에 코멘트를 남기지 않고 대화 내에서 모든 과정이 완결됩니다. "셀프 리뷰", "자체 리뷰 후 수정", "리뷰하고 고쳐줘", "self dev", "코드 점검하고 수정" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 에이전트 셀프 개발 (Self-Dev)

7개 전문 에이전트가 PR을 리뷰하고, 결과를 사용자에게 보여준 뒤, 사용자의 판단에 따라 코드를 직접 수정하는 올인원 스킬입니다.

**GitHub에 코멘트를 남기지 않습니다.** 모든 상호작용은 대화 내에서 이루어집니다.

**모든 출력은 한국어로 작성한다.** 이슈 테이블, 질문, 수정 요약 등 사용자에게 보이는 모든 텍스트는 한국어를 사용할 것. 코드 자체는 기존 코드베이스의 언어 규칙을 따른다.

## 사용 방법

```text
/roto-band-self-dev [PR_NUMBER]
```

| 인자 | 설명 |
|------|------|
| `PR_NUMBER` | GitHub PR 번호 (선택). 생략 시 현재 브랜치의 PR을 자동 탐지 |

---

## 워크플로우

### Phase 1: 리뷰 (agent-code-review --silent과 동일)

이 단계는 `/roto-band-code-review`의 Step 1~5를 silent 모드로 실행하는 것과 동일하다. 에이전트 지침 파일은 이 플러그인의 `agents/` 디렉토리에서 찾는다.

#### Step 1: PR 번호 결정

1. 인자에서 숫자가 있으면 `PR_NUMBER`로 사용.
2. 생략 시 `gh pr view --json number --jq .number`로 자동 탐지.
3. 실패 시 에러: `[ERROR] 현재 브랜치에 연결된 PR이 없습니다.`

#### Step 2: PR 메타데이터 수집

```bash
gh pr view <PR_NUMBER> --json number,title,additions,deletions,changedFiles,files,baseRefName,headRefName,headRefOid
```

#### Step 3: Diff 수집 및 파일 필터링

```bash
gh pr diff <PR_NUMBER>
```

자동 제외: `*.lock`, `*.min.*`, `*.map`, `*.generated.*`, `dist/`, `build/`, `.next/`, `*.snap`, `*.svg`

#### Step 4: 7개 서브 에이전트 병렬 실행

**반드시 Agent 도구를 사용하여 7개 서브 에이전트를 동시에 병렬 실행한다.**

각 에이전트에게 전달할 프롬프트 구성:
1. 에이전트 역할 지침 (이 플러그인의 `agents/<name>.md` 전체 내용)
2. PR 메타데이터
3. 필터링된 Diff (8,000줄+ 시 에이전트 우선 파일만 전달)
4. 변경 파일 목록
5. 라인 번호 규칙: "변경 후 파일의 실제 라인 번호 기준"

응답 형식: JSON 배열 `[{file, line, severity, title, description, suggestion}]`

에이전트 목록:

| 에이전트 | 파일 | model |
|----------|------|-------|
| UI Designer | `agents/ui-designer.md` | sonnet |
| Senior FE | `agents/senior-fe-engineer.md` | sonnet |
| Junior FE | `agents/junior-fe-engineer.md` | sonnet |
| BE Engineer | `agents/backend-engineer.md` | sonnet |
| SEO Specialist | `agents/seo-specialist.md` | sonnet |
| i18n Reviewer | `agents/i18n-reviewer.md` | sonnet |
| Claude Config | `agents/claude-config-reviewer.md` | sonnet |

#### Step 5: 결과 통합

1. 에이전트 응답 파싱 (JSON → fallback `[...]` 추출 → 실패 시 스킵)
2. 각 이슈에 `agent` 필드 추가
3. 중복 통합: 같은 file + line (±3) → 최고 severity 채택, 에이전트 병기
4. Severity 순 정렬: 🔴 Critical → 🟠 Major → 🟡 Minor → 🔵 Misc

### Phase 2: 사용자와 수정 방안 협의

#### Step 6: 리뷰 결과 출력

통합된 이슈를 severity별 테이블로 사용자에게 보여준다:

```markdown
## 🔍 코드 리뷰 결과

| Severity | 개수 |
|----------|------|
| 🔴 Critical | N |
| 🟠 Major | N |
| 🟡 Minor | N |
| 🔵 Misc | N |

### 에이전트별 관점 요약
- **UI Designer**: 한 줄 요약
- **Senior FE**: 한 줄 요약
...

### 이슈 목록

| # | Severity | 파일 | 라인 | 에이전트 | 제목 |
|---|----------|------|------|----------|------|
| 1 | 🔴 | actions.ts | 132 | Senior FE + BE | Open Redirect |
| 2 | 🟠 | ProfileStep.tsx | 146 | Senior FE | canSubmit 허점 |
...

<각 이슈 상세: 설명 + 권장안>
```

#### Step 7: 사용자에게 수정 방안 질문

AskUserQuestion 도구를 사용하여 사용자에게 어떤 이슈를 어떻게 수정할지 묻는다.

질문 예시:
```
위 리뷰 결과를 확인해주세요.

- 수정할 이슈 번호를 지정해주세요 (예: "1,2,5,7" 또는 "Critical+Major 전부" 또는 "전부")
- 특정 이슈에 대해 다른 수정 방향이 있으면 알려주세요
- 스킵할 이슈와 사유가 있으면 함께 알려주세요
```

사용자가 응답하면:
- 수정 대상 이슈와 수정 방향을 확정
- 사용자가 지정한 스킵 사유 기록

### Phase 3: 코드 수정

#### Step 8: 코드 수정 실행

사용자가 승인한 이슈를 severity 순서(Critical → Major → Minor → Misc)로 수정한다.

**수정 절차** (이슈 하나당):
1. 해당 파일을 Read 도구로 읽어 현재 코드 확인
2. 리뷰 권장안 + 사용자 지시사항을 반영하여 Edit/Write 도구로 수정
3. 수정이 다른 이슈에 영향을 줄 수 있으므로 후속 이슈의 코드를 다시 확인

**수정 원칙**:
- 수정 범위는 리뷰 지적 사항으로 한정. 주변 코드를 불필요하게 리팩터링하지 않는다.
- import 추가/제거가 필요하면 함께 처리
- 사용자가 별도 수정 방향을 지시한 이슈는 권장안 대신 사용자 지시를 따른다

#### Step 9: 수정 후 검증

모든 수정이 완료되면:
- TypeScript 타입 체크 실행 (`npx tsc --noEmit`)
- 테스트 실행 (`npx vitest run` 등 프로젝트의 테스트 커맨드)
- 새로운 에러가 발견되면 수정

### Phase 4: 최종 요약 출력

#### Step 10: 결과 요약

사용자에게 전체 과정의 요약을 출력한다:

```markdown
## 🔧 Self-Dev 완료

### 리뷰 결과
- 총 이슈: N건 (🔴 Critical N / 🟠 Major N / 🟡 Minor N / 🔵 Misc N)

### 수정된 항목
- ✅ [에이전트명] 이슈 제목 — `파일:라인` (수정 내용 요약)
- ✅ [에이전트명] 이슈 제목 — `파일:라인` (수정 내용 요약)

### 미반영 항목
- ⏭️ [에이전트명] 이슈 제목 — `파일:라인` (사유)

### 변경 통계
- 수정: N건
- 스킵: N건

### 검증 결과
- TypeScript: ✅ 통과 (기존 에러 N건 제외)
- 테스트: ✅ 223개 통과

### 수정된 파일
- `path/to/file1.ts`
- `path/to/file2.tsx`
```

마지막으로 커밋 여부를 사용자에게 확인한다.

---

## 도구 사용 규칙

Bash 대신 Claude 전용 도구를 우선 사용한다:

| 작업 | 사용할 도구 | Bash 사용 금지 |
|------|-----------|---------------|
| 파일 읽기 | **Read** | `cat`, `head`, `tail` |
| 파일 수정 | **Edit** | `sed`, `awk` |
| 파일 생성 | **Write** (디렉토리 자동 생성) | `mkdir -p` + `echo >` |
| 파일 검색 | **Glob** | `find`, `ls` |
| 코드 내용 검색 | **Grep** | `grep`, `rg` |

Bash는 아래 경우에만 사용한다:
- `gh` CLI 명령 (PR 조회, diff 수집)
- `npx tsc --noEmit` (타입 체크)
- `npx vitest run` 등 프로젝트 테스트 명령

## 오류 처리

| 상황 | 처리 |
|------|------|
| GitHub 인증 실패 | `[ERROR] GitHub authentication required. Run: gh auth login` |
| PR 미존재 | `[ERROR] PR #<NUMBER>을 찾을 수 없습니다.` |
| 서브 에이전트 전원 실패 | `[ERROR] 모든 에이전트가 실패했습니다. 다시 시도해주세요.` |
| 일부 에이전트 실패 | 실패 에이전트 이름 로그, 나머지 결과로 진행 |
| 이슈 0건 | `리뷰 결과 이슈가 없습니다. 코드가 양호합니다!` 출력 후 종료 |
| 수정 중 에러 | 해당 이슈 스킵 + 사유 기록, 나머지 계속 |
| 타입 체크/테스트 실패 | 실패 내용을 사용자에게 보여주고 수정 시도 |
