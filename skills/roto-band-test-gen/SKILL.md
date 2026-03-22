---
name: roto-band-test-gen
description: PR diff 또는 변경 파일을 분석하여 누락된 테스트를 자동 생성합니다. 7개 에이전트가 각자 관점에서 테스트가 필요한 영역을 식별하고, 통합된 테스트 계획에 따라 테스트 코드를 작성합니다. "테스트 생성", "테스트 만들어줘", "test gen", "테스트 추가", "테스트 작성" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 멀티 에이전트 테스트 생성 (Test Gen)

7개 전문 에이전트가 변경 코드를 분석하여 각자의 관점에서 테스트가 필요한 영역을 식별하고, 통합된 테스트 계획에 따라 테스트 코드를 자동 생성하는 스킬입니다.

**GitHub에 코멘트를 남기지 않습니다.** 모든 과정은 대화 내에서 이루어집니다.

**모든 출력은 한국어로 작성한다.** 테스트 코드의 describe/it 설명은 프로젝트 기존 컨벤션을 따른다 (영어 또는 한국어).

## 사용 방법

```text
/roto-band-test-gen [PR_NUMBER | 파일경로...] [--scope unit|integration|e2e|all]
```

| 인자 | 설명 |
|------|------|
| `PR_NUMBER` | GitHub PR 번호 (선택). 생략 시 현재 브랜치의 PR 또는 unstaged 변경 기준 |
| `파일경로...` | 특정 파일만 대상으로 테스트 생성 (공백 구분, 선택) |
| `--scope` | 테스트 범위 (선택, 기본값: `unit`). `unit`, `integration`, `e2e`, `all` |

### 예시

```text
/roto-band-test-gen                          # 현재 브랜치 PR 기준 단위 테스트
/roto-band-test-gen 64                       # PR #64 기준
/roto-band-test-gen src/lib/auth.ts          # 특정 파일에 대한 테스트
/roto-band-test-gen --scope all              # unit + integration + e2e
/roto-band-test-gen 64 --scope integration   # PR #64 통합 테스트
```

---

## 워크플로우

### Step 1: 대상 파악

#### 1-A: PR 번호가 주어진 경우

1. PR 메타데이터 수집:
   ```bash
   gh pr view <PR_NUMBER> --json number,title,additions,deletions,changedFiles,files,baseRefName,headRefName
   ```
2. Diff 수집:
   ```bash
   gh pr diff <PR_NUMBER>
   ```

#### 1-B: 파일 경로가 주어진 경우

1. 해당 파일을 Read 도구로 읽는다.
2. `git diff`로 해당 파일의 변경사항을 확인한다.

#### 1-C: 인자가 없는 경우

1. 현재 브랜치 PR 자동 탐지: `gh pr view --json number --jq .number`
2. PR이 없으면 unstaged 변경 기준: `git diff`
3. 변경사항이 없으면 에러: `[ERROR] 변경 사항이 없습니다.`

#### 자동 제외

테스트 대상에서 아래 파일은 제외한다:
- `*.test.*`, `*.spec.*`, `__tests__/` (기존 테스트 파일)
- `*.lock`, `*.min.*`, `*.map`, `*.generated.*`
- `dist/`, `build/`, `.next/`, `node_modules/`
- `*.snap`, `*.svg`, `*.json` (설정 파일 제외)
- `*.css`, `*.scss` (스타일 파일)

#### 기존 테스트 환경 파악

프로젝트의 테스트 프레임워크와 패턴을 자동 감지한다:

1. `package.json`에서 테스트 프레임워크 확인 (vitest, jest, playwright, cypress 등)
2. 기존 테스트 파일을 Glob으로 탐색: `**/*.test.{ts,tsx}`, `**/*.spec.{ts,tsx}`
3. 기존 테스트 파일 1~2개를 Read로 읽어 패턴 파악:
   - import 스타일 (`import { describe, it } from 'vitest'` 등)
   - 파일 위치 컨벤션 (같은 디렉토리 vs `__tests__/` 디렉토리)
   - mock 패턴 (`vi.mock`, `jest.mock` 등)
   - assertion 스타일 (`expect`, `assert` 등)
   - 테스트 설명 언어 (한국어 vs 영어)

### Step 2: 7개 서브 에이전트 병렬 분석

**반드시 Agent 도구를 사용하여 6개의 서브 에이전트를 동시에 병렬 실행한다.**
(Claude Config Reviewer는 테스트 생성에 기여하지 않으므로 제외한다.)

> **참고**: 에이전트 지침 파일은 코드 리뷰용으로 작성되어 있다. 각 에이전트 프롬프트에 "이번에는 코드 리뷰가 아닌 **테스트 분석**을 수행합니다"라고 명시하여 역할을 전환한다.

각 에이전트에게 전달할 프롬프트:

1. **에이전트 역할 지침**: `agents/<name>.md` 전체 내용
2. **변경 코드**: diff 또는 파일 내용
3. **기존 테스트 패턴**: Step 1에서 파악한 테스트 컨벤션
4. **테스트 분석 요청**:

```
변경된 코드를 분석하여, 당신의 전문 관점에서 테스트가 필요한 항목을 보고하세요.

각 항목에 대해:
1. 무엇을 테스트해야 하는지 (what)
2. 왜 테스트가 필요한지 (why) — 당신의 전문 관점에서
3. 어떤 종류의 테스트인지 (unit / integration / e2e)
4. 테스트 시나리오 (정상, 에러, 엣지 케이스)
5. 필요한 mock/fixture가 있으면 명시

응답은 반드시 JSON 배열이어야 합니다.
마크다운 코드블록(```)으로 감싸지 마세요. 앞뒤 설명 텍스트도 넣지 마세요.
순수한 JSON 배열만 반환하세요.

[
  {
    "file": "테스트 대상 파일 경로",
    "testFile": "테스트 파일 경로 (기존 컨벤션 따름)",
    "type": "unit|integration|e2e",
    "priority": "Critical|Major|Minor",
    "title": "테스트 항목 제목",
    "scenarios": [
      {
        "name": "시나리오 이름",
        "description": "시나리오 설명",
        "category": "normal|error|edge"
      }
    ],
    "mocks": ["필요한 mock 목록"],
    "reason": "이 테스트가 필요한 이유 (이 에이전트의 관점에서)"
  }
]

테스트가 필요 없으면 빈 배열 []을 반환하세요.
```

에이전트별 테스트 관점:

| 에이전트 | 테스트 관점 |
|----------|-----------|
| UI Designer | 접근성 테스트, 반응형 렌더링, 시각적 회귀 |
| Senior FE | 컴포넌트 렌더링, hook 동작, RSC/Client 경계, 상태 관리 |
| Junior FE | 기본 렌더링, props 전달, 조건부 렌더링 |
| BE Engineer | API 호출, 에러 핸들링, 데이터 변환, Server Actions |
| SEO Specialist | 메타데이터 생성, 구조화 데이터, OG 이미지 |
| i18n Reviewer | 다국어 렌더링, 로케일 전환, 날짜/숫자 포맷 |

에이전트별 설정:

| 에이전트 | 파일 | subagent_type | model |
|----------|------|---------------|-------|
| UI Designer | `agents/ui-designer.md` | general-purpose | sonnet |
| Senior FE | `agents/senior-fe-engineer.md` | general-purpose | sonnet |
| Junior FE | `agents/junior-fe-engineer.md` | general-purpose | sonnet |
| BE Engineer | `agents/backend-engineer.md` | general-purpose | sonnet |
| SEO Specialist | `agents/seo-specialist.md` | general-purpose | sonnet |
| i18n Reviewer | `agents/i18n-reviewer.md` | general-purpose | sonnet |

### Step 3: 테스트 계획 통합

#### 3-1: 에이전트 응답 파싱

1. 각 에이전트 응답을 JSON 배열로 파싱
2. 실패 시 `[...]` 패턴 추출 재시도
3. 각 항목에 `agent` 필드 추가

#### 3-2: 중복 통합 및 필터링

1. 같은 `file` + 유사한 `title`의 테스트 항목을 통합
   - 복수 에이전트가 보고한 항목은 priority 상향
   - 각 에이전트의 시나리오를 병합
2. `--scope` 필터 적용:
   - `unit`: type이 unit인 항목만
   - `integration`: type이 integration인 항목만
   - `e2e`: type이 e2e인 항목만
   - `all`: 전체
3. Priority 순 정렬: Critical → Major → Minor

#### 3-3: 테스트 계획 출력

사용자에게 테스트 계획을 보여준다:

```markdown
## 🧪 테스트 생성 계획

| # | 대상 파일 | 테스트 파일 | 종류 | 시나리오 수 | 에이전트 | Priority |
|---|----------|-----------|------|-----------|----------|----------|
| 1 | `src/lib/auth.ts` | `src/lib/auth.test.ts` | unit | 5 | Senior FE + BE | Critical |
| 2 | `src/components/Login.tsx` | `src/components/Login.test.tsx` | unit | 3 | UI + Junior FE | Major |

### 상세

#### 1. `src/lib/auth.ts` — 인증 로직 테스트
- ✅ 정상 로그인 → 토큰 반환
- ✅ 잘못된 비밀번호 → 에러
- ✅ 만료된 토큰 → 갱신 시도
- ✅ 네트워크 에러 → 재시도
- ✅ 동시 로그인 제한
```

### Step 4: 사용자 확인

AskUserQuestion 도구로 확인한다:

```
위 테스트 계획을 확인해주세요.

- 제거할 항목이 있으면 번호를 알려주세요
- 추가할 테스트 시나리오가 있으면 알려주세요
- 확정되면 "시작" 또는 "OK"라고 답해주세요
```

### Step 5: 테스트 코드 생성

확정된 테스트 계획을 기반으로 테스트 파일을 생성한다.

**생성 절차** (테스트 파일 하나당):

1. 테스트 대상 파일을 Read 도구로 읽어 구현 코드 확인
2. 기존 테스트 파일이 있으면 Read로 읽어 기존 테스트 확인 (중복 방지)
3. Step 1에서 파악한 테스트 패턴(import, mock, assertion 스타일)을 따라 테스트 작성
4. Write/Edit 도구로 테스트 파일 생성/수정

**생성 원칙**:
- 프로젝트의 기존 테스트 컨벤션을 엄격히 따른다
- 테스트 파일 위치는 기존 패턴에 따라 결정 (co-located vs `__tests__/`)
- 각 테스트는 독립적으로 실행 가능해야 한다 (테스트 간 상태 공유 금지)
- mock은 최소한으로 사용, 실제 동작에 가까운 테스트 우선
- describe/it 네스팅은 2단계 이내
- 테스트 설명은 기존 테스트의 언어 컨벤션을 따른다

각 파일 생성 시 사용자에게 진행 상황 출력:

```
🧪 [2/5]: ✅ `src/lib/auth.test.ts` — 5개 시나리오 생성
```

### Step 6: 테스트 실행 및 수정

생성된 테스트를 실행한다:

```bash
npx vitest run --reporter=verbose <생성된 테스트 파일들>
```

또는 프로젝트의 테스트 커맨드 사용.

**결과 처리**:
- 전체 통과: Step 7로 진행
- 실패 테스트 발견:
  1. 실패 원인 분석 (테스트 코드 오류 vs 실제 버그 발견)
  2. **테스트 코드 오류**: 테스트 수정 후 재실행 (최대 3회)
  3. **실제 버그 발견**: 테스트는 유지하고, 발견된 버그를 사용자에게 보고
     ```
     ⚠️ 테스트 실행 중 버그 발견:
     - `auth.test.ts` > "만료된 토큰 갱신" — refreshToken이 만료 체크를 하지 않음
     수정하시겠습니까?
     ```

### Step 7: 최종 요약

```markdown
## 🧪 테스트 생성 완료

| 항목 | 값 |
|------|-----|
| 생성된 테스트 파일 | N개 |
| 총 테스트 케이스 | N개 |
| 실행 결과 | ✅ N 통과 / ❌ N 실패 |
| 발견된 버그 | N건 |

### 생성된 파일
- `src/lib/auth.test.ts` (신규, 5 케이스)
- `src/components/Login.test.tsx` (3 케이스 추가)

### 에이전트별 기여
| 에이전트 | 식별한 테스트 항목 |
|----------|-----------------|
| Senior FE | auth 로직, hook 동작 |
| BE Engineer | API 에러 핸들링 |
| UI Designer | 접근성 테스트 |
| ... | ... |
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
- `gh` CLI 명령 (PR 조회, diff)
- `git diff` (변경사항 확인)
- 테스트 실행 명령 (`npx vitest run` 등)

## 오류 처리

| 상황 | 처리 |
|------|------|
| 변경사항 없음 | `[ERROR] 변경 사항이 없습니다.` |
| 테스트 프레임워크 미감지 | `[ERROR] 테스트 프레임워크를 감지하지 못했습니다. package.json을 확인해주세요.` |
| PR 미존재 | `[ERROR] PR #<NUMBER>을 찾을 수 없습니다.` |
| 서브 에이전트 전원 실패 | `[ERROR] 모든 에이전트가 실패했습니다. 다시 시도해주세요.` |
| 일부 에이전트 실패 | 실패 에이전트 로그, 나머지 결과로 진행 |
| 테스트 실행 실패 (환경 문제) | 에러 출력 + 수동 실행 명령어 안내 |
| 테스트 수정 3회 초과 실패 | 해당 테스트 스킵 + 사유 기록, 나머지 계속 |
