---
name: roto-band-code-review
description: GitHub PR에 대한 멀티 에이전트 코드 리뷰 수행 (한국어). 7개의 전문 서브 에이전트(UI Designer, Senior FE, Junior FE, BE Engineer, SEO Specialist, i18n Reviewer, Claude Config Reviewer)가 각각의 관점에서 리뷰하고 인라인 코멘트 + PR 요약을 남깁니다.
user-invocable: true
---

# 멀티 에이전트 코드 리뷰

7개의 전문화된 서브 에이전트(UI Designer, Senior FE, Junior FE, BE Engineer, SEO Specialist, i18n Reviewer, Claude Config Reviewer)가 각각의 관점에서 PR을 리뷰하고, 결과를 markdown 보고서로 통합한 뒤 인라인 코멘트 + PR 요약 코멘트를 남기는 스킬입니다.

**모든 출력은 한국어로 작성한다.** PR 코멘트, 요약, 대화 창 출력 등 사용자에게 보이는 모든 텍스트는 한국어를 사용할 것.

## 사용 방법

```text
/roto-band-code-review [PR_NUMBER] [--silent]
```

| 인자 | 설명 |
|------|------|
| `PR_NUMBER` | GitHub PR 번호 (선택). 생략 시 현재 브랜치의 PR을 자동 탐지 |
| `--silent` | PR에 코멘트를 남기지 않고 현재 대화에 요약만 출력 (선택) |

기본 동작은 PR에 인라인 코멘트 + 요약 코멘트를 게시합니다.
`--silent` 플래그를 붙이면 PR에는 아무것도 남기지 않고, 리뷰 결과를 현재 대화 창에만 출력합니다.

### 예시

```text
/roto-band-code-review          # 현재 브랜치의 PR을 자동 탐지하여 리뷰
/roto-band-code-review 64       # PR #64 리뷰
/roto-band-code-review --silent # 현재 브랜치 PR을 silent 모드로 리뷰
/roto-band-code-review 64 --silent
```

## 절대 금지 사항

- 이 스킬은 **리뷰만** 수행합니다. 코드 수정, 자동 수정(autofix), 리팩터링은 절대 수행하지 않습니다.
- 수정 권장안은 제시하지만 직접 적용하지 않습니다.

---

## 워크플로우

### Step 1: 인자 파싱 및 PR 번호 결정

1. `--silent` 플래그 유무를 확인한다.
2. 인자에서 숫자가 있으면 `PR_NUMBER`로 사용한다.
3. **PR 번호가 생략된 경우**, 현재 브랜치 기준으로 자동 탐지한다:
   ```bash
   gh pr view --json number --jq .number
   ```
   - 성공 시: 해당 PR 번호를 사용하고, 사용자에게 `현재 브랜치(branch-name)의 PR #N을 리뷰합니다.`라고 알린다.
   - 실패 시 (현재 브랜치에 PR이 없음): 에러 메시지 출력 후 종료:
     ```
     [ERROR] 현재 브랜치에 연결된 PR이 없습니다. PR 번호를 직접 지정해주세요.
     ```
4. PR 번호가 양의 정수가 아닌 값이면 에러:
   ```
   [ERROR] PR 번호는 양의 정수여야 합니다. 입력값: '<VALUE>'
   ```

### Step 2: PR 존재 여부 및 스킵 조건 확인

아래 명령어로 PR 메타데이터를 수집한다:

```bash
gh pr view <PR_NUMBER> --json number,title,body,additions,deletions,changedFiles,files,baseRefName,headRefName,author,createdAt,labels,isDraft,headRefOid
```

#### 스킵 조건 체크

아래 조건에 해당하면 사용자에게 알리고 진행 여부를 확인받는다:

- **bot PR**: author.login이 `dependabot`, `renovate`, `github-actions` 등 bot인 경우
- **draft PR**: `isDraft`가 true인 경우
- **skip 라벨/제목**: labels에 `skip review`가 포함되거나 title에 `[skip review]`가 포함된 경우

사용자가 진행을 원하면 계속, 아니면 종료.

### Step 3: Diff 수집 및 파일 필터링

```bash
gh pr diff <PR_NUMBER>
```

#### 자동 제외 파일 (리뷰 대상에서 제거)

아래 패턴에 해당하는 파일은 diff에서 제외한다:

- `*.lock`, `*-lock.*` (yarn.lock, package-lock.json, pnpm-lock.yaml 등)
- `*.min.js`, `*.min.css`
- `*.map` (sourcemap)
- `*.generated.*`, `*.g.*`
- `dist/`, `build/`, `.next/`, `node_modules/`
- `*.snap` (jest snapshot)
- `*.svg` (단, JSX/TSX 내 인라인 SVG는 제외하지 않음)

#### 대규모 PR 처리

- `additions + deletions >= 15,000`: 파일 목록 기반 리뷰로 전환, 사용자에게 경고
- `additions + deletions >= 10,000`: diff 수집하되 경고 마커 포함

### Step 4: 서브 에이전트 병렬 실행

**반드시 Agent 도구를 사용하여 7개의 서브 에이전트를 병렬로 실행한다.**
각 에이전트는 하나의 Agent 도구 호출로 spawn하며, 7개를 동시에 보낸다.

각 에이전트에게 전달할 프롬프트에는 반드시 아래 정보를 모두 포함한다:

1. **에이전트 역할 지침**: 이 플러그인의 `agents/` 디렉토리에서 해당 에이전트의 `.md` 파일 내용 전체를 읽어서 프롬프트 앞부분에 삽입. 파일 경로는 이 스킬 파일(`SKILL.md`)의 위치를 기준으로 `../../agents/<name>.md`로 접근하거나, Glob 도구로 `**/agents/<name>.md` 패턴을 사용하여 찾는다.
2. **PR 메타데이터**: title, body, author, base/head branch
3. **필터링된 Diff**: 자동 제외 파일을 제거한 diff 전문. 단, diff가 매우 길면(8,000줄+) 해당 에이전트의 우선 파일 패턴에 해당하는 파일만 전달하고, 나머지는 파일명 목록만 전달한다.
4. **변경된 파일 목록**: 파일명과 additions/deletions
5. **라인 번호 규칙**: "line은 반드시 diff가 아닌 **변경 후 파일의 실제 라인 번호**를 보고하세요. diff의 `@@ -a,b +c,d @@` 헤더에서 `+` 쪽 번호 기준입니다."

각 에이전트 프롬프트는 아래 형식의 결과를 JSON 배열로 반환하도록 요청한다:

```
응답은 반드시 JSON 배열이어야 합니다.
마크다운 코드블록(```)으로 감싸지 마세요. 앞뒤 설명 텍스트도 넣지 마세요.
순수한 JSON 배열만 반환하세요.

[
  {
    "file": "src/components/Button.tsx",
    "line": 42,
    "severity": "Major",
    "title": "이슈 제목",
    "description": "상세 설명",
    "suggestion": "수정 권장안 (선택)"
  }
]

이슈가 없으면 빈 배열 []을 반환하세요.
```

에이전트별 설정:

| 에이전트 | 파일 경로 | subagent_type | model |
|----------|----------|---------------|-------|
| UI Designer | `agents/ui-designer.md` | general-purpose | sonnet |
| Senior FE Engineer | `agents/senior-fe-engineer.md` | general-purpose | sonnet |
| Junior FE Engineer | `agents/junior-fe-engineer.md` | general-purpose | sonnet |
| Backend Engineer | `agents/backend-engineer.md` | general-purpose | sonnet |
| SEO Specialist | `agents/seo-specialist.md` | general-purpose | sonnet |
| i18n Reviewer | `agents/i18n-reviewer.md` | general-purpose | sonnet |
| Claude Config Reviewer | `agents/claude-config-reviewer.md` | general-purpose | sonnet |

### Step 5: 결과 통합 및 리뷰 보고서 저장 (Pass 1)

이 단계에서 에이전트 결과를 통합하여 **markdown 리뷰 보고서**로 저장한다.
이 보고서가 모든 후속 작업(코멘트 게시, 대화 출력)의 Single Source of Truth이다.

#### 5-1: 에이전트 응답 파싱

1. 각 에이전트의 응답을 파싱한다:
   - 먼저 순수 JSON 배열로 파싱 시도
   - 실패 시, 응답에서 `[...]` 패턴을 추출하여 재시도 (마크다운 코드블록으로 감싼 경우 대응)
   - 그래도 실패 시 해당 에이전트 결과를 스킵하고 보고서에 `⚠️ {에이전트명} 에이전트 응답 파싱 실패` 기록
2. 파싱된 각 이슈에 `agent` 필드를 추가한다.

#### 5-2: 이슈 통합 및 정렬

1. 모든 이슈를 하나의 리스트로 합친다.
2. **중복 통합**: 같은 `file` + `line` (±3 라인 이내)에 2개 이상의 에이전트가 이슈를 보고한 경우:
   - 가장 높은 severity로 통합 (Critical > Major > Minor > Misc)
   - 각 에이전트의 관점을 description에 병기
3. **Severity 순으로 정렬**: Critical → Major → Minor → Misc

#### Severity 레벨

| 레벨 | 의미 | 이모지 |
|------|------|--------|
| Critical | 보안/데이터 손실/서비스 중단 | 🔴 |
| Major | 기능 오작동/회귀 리스크 | 🟠 |
| Minor | 코드 품질/성능 개선 | 🟡 |
| Misc | 스타일/가독성/제안 | 🔵 |

#### 5-3: 리뷰 보고서 markdown 저장

**저장 경로**: `code-reviews/PR-<PR_NUMBER>.md`

Write 도구로 파일을 생성하면 부모 디렉토리가 자동으로 생성된다. 별도의 `mkdir` 명령이 필요 없다.

보고서 템플릿:

```markdown
# PR-<NUMBER> Agent Code Review

## 메타데이터

| 항목 | 값 |
|------|-----|
| PR | #<NUMBER> |
| 리뷰 일시 | <YYYY-MM-DD HH:MM> |
| 리뷰어 | AI Multi-Agent (`/roto-band-code-review`) |
| 변경 규모 | +<ADDITIONS> / -<DELETIONS> lines, <CHANGED_FILES> files |
| 베이스 브랜치 | <BASE_REF> |
| 소스 브랜치 | <HEAD_REF> |
| 모드 | <기본 / silent> |

## 리뷰 요약

| Severity | 개수 |
|----------|------|
| 🔴 Critical | N |
| 🟠 Major | N |
| 🟡 Minor | N |
| 🔵 Misc | N |

## 에이전트별 관점 요약

- **UI Designer**: 한 줄 요약
- **Senior FE**: 한 줄 요약
- **Junior FE**: 한 줄 요약
- **BE Engineer**: 한 줄 요약
- **SEO Specialist**: 한 줄 요약
- **i18n Reviewer**: 한 줄 요약
- **Claude Config**: 한 줄 요약

## 인라인 코멘트 목록

> 아래 목록의 각 항목은 PR 코드의 특정 라인에 남길 코멘트입니다.
> `commented` 컬럼은 실제 PR에 코멘트가 게시되었는지 여부입니다.

### 🔴 Critical

| # | file | line | agent | title | commented |
|---|------|------|-------|-------|-----------|
| 1 | `path/to/file.tsx` | 42 | Senior FE | 이슈 제목 | ⬜ |

**#1 상세**
- **설명**: 상세 설명
- **권장안**: 수정 권장안

### 🟠 Major
(같은 형식)

### 🟡 Minor
(같은 형식)

### 🔵 Misc
(같은 형식)

## 코멘트 게시 로그

| 시각 | 이슈 # | 상태 | 비고 |
|------|--------|------|------|
| (Step 6 완료 후 기록) | | | |

---
🤖 Generated by `/roto-band-code-review` | 7-agent multi-perspective review
```

**핵심 규칙**:
- `commented` 컬럼은 초기값 `⬜` (미게시)
- 이슈가 0건이면 각 severity 섹션에 "None found."를 기록하고, 에이전트별 관점 요약만 작성한다.
- 보고서 저장 후 사용자에게 저장 경로를 알린다.

### Step 6: 코멘트 게시 (Pass 2)

이 단계는 Step 5에서 저장한 markdown 보고서를 읽고, 각 이슈를 PR에 코멘트로 게시한 뒤, 보고서를 업데이트한다.

#### 6-A: `--silent` 모드

PR에 코멘트를 남기지 않는다.
1. 보고서 markdown의 내용을 대화 창에 요약 출력한다.
2. 보고서의 `commented` 컬럼은 모두 `➖` (silent)로 업데이트한다.
3. 코멘트 게시 로그에 `silent 모드 — 코멘트 게시 생략`을 기록한다.

#### 6-B: 기본 모드 (PR에 코멘트 게시)

##### 6-B-1: 인라인 코멘트 게시

보고서의 인라인 코멘트 목록을 순회하며, severity 높은 순서대로 PR에 인라인 코멘트를 게시한다.

```bash
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments -X POST \
  -f body='<comment_body>' \
  -f commit_id='<HEAD_SHA>' \
  -f path='<file>' \
  -F line=<line> \
  -f side='RIGHT' \
  -f subject_type='line'
```

인라인 코멘트 본문 형식:

```
[<이모지> <Severity>] [<에이전트명>] <제목>

<설명>

<수정 권장안이 있으면>
**권장안**: <수정 권장안>

---
🤖 AI 자동 리뷰 (`/roto-band-code-review`)
```

**게시 규칙**:
- 인라인 코멘트는 최대 15개까지만 게시한다. 초과 시 나머지는 PR 요약 코멘트에 포함.
- 각 코멘트 게시 성공 시: 보고서의 해당 이슈 `commented` 컬럼을 `✅`로 업데이트
- 422 에러 시:
  1. 인접 line(±2)으로 재시도
  2. 그래도 실패 시: `commented` 컬럼을 `❌`로 업데이트하고, 코멘트 게시 로그에 실패 사유 기록
  3. 해당 이슈는 PR 요약 코멘트에 포함
- rate limit 에러 시: 남은 이슈는 모두 `⏸️`로 표시하고 PR 요약 코멘트에 포함

##### 6-B-2: PR 요약 코멘트 게시

모든 인라인 코멘트 처리 후, PR에 요약 코멘트를 남긴다.

```bash
gh pr comment <PR_NUMBER> --body "$(cat <<'EOF'
## 🤖 AI 멀티 에이전트 코드 리뷰

### 리뷰 요약
| Severity | 개수 |
|----------|------|
| 🔴 Critical | N |
| 🟠 Major | N |
| 🟡 Minor | N |
| 🔵 Misc | N |

### 이슈 목록
(severity별 정렬, 각 이슈에 **[에이전트명]** 포함)
(형식: `N. **[에이전트명] 이슈 제목** — \`파일:라인\``)
(중복 통합된 이슈는 `[Senior FE + BE]`처럼 모든 에이전트 병기)
(인라인 코멘트 게시에 실패한 이슈는 여기에 상세 내용 포함)

### 에이전트별 관점 요약
- **UI Designer**: 한 줄 요약
- **Senior FE**: 한 줄 요약
- **Junior FE**: 한 줄 요약
- **BE Engineer**: 한 줄 요약
- **SEO Specialist**: 한 줄 요약
- **i18n Reviewer**: 한 줄 요약
- **Claude Config**: 한 줄 요약

---
🤖 Generated by `/roto-band-code-review` | 7-agent multi-perspective review
EOF
)"
```

요약 코멘트 게시 후, 보고서의 코멘트 게시 로그에 최종 결과를 기록한다:

```markdown
## 코멘트 게시 로그

| 시각 | 이슈 # | 상태 | 비고 |
|------|--------|------|------|
| 14:30:01 | #1 | ✅ 성공 | |
| 14:30:02 | #2 | ❌ 실패 | 422: line not part of diff |
| 14:30:03 | #3 | ✅ 성공 | 인접 라인(44→45)으로 재시도 |
| 14:30:03 | — | ✅ 성공 | PR 요약 코멘트 게시 |
```

##### 6-B-3: 대화 창 출력

사용자에게 간략한 요약을 출력한다:
- 보고서 저장 경로
- 총 이슈 수와 severity별 분포
- 인라인 코멘트 게시 결과: `✅ N개 성공 / ❌ N개 실패 / ⏸️ N개 스킵`
- PR 요약 코멘트 URL

## 에이전트별 파일 할당 가이드

변경 파일이 많을 때(20개+), 각 에이전트에게 관련도 높은 파일을 우선 할당한다:

| 에이전트 | 우선 파일 패턴 |
|----------|---------------|
| UI Designer | `*.tsx`, `*.css`, `*.scss`, `*.module.*`, `components/` |
| Senior FE | `*.tsx`, `*.ts`, `app/`, `pages/`, `lib/`, `hooks/`, `middleware.*` |
| Junior FE | 모든 변경 파일 (가독성 관점이므로 제한 없음) |
| BE Engineer | `*.ts`, `api/`, `server/`, `lib/`, `services/`, `actions/`, `types/` |
| SEO Specialist | `app/**/page.tsx`, `app/**/layout.tsx`, `**/metadata.*`, `**/sitemap.*`, `lib/slug.*` |
| i18n Reviewer | `*.tsx`, `messages/`, `**/i18n.*`, `**/locale*`, `app/**/page.tsx` |
| Claude Config Reviewer | `CLAUDE.md`, `.claude/rules/**`, `.claude/skills/**`, `.claude/settings*`, `.agents/**` |

diff가 컨텍스트에 충분히 들어가는 크기(8,000줄 미만)면 모든 에이전트에게 전체 diff를 전달한다. 8,000줄 이상이면 Step 4의 규칙에 따라 우선 파일의 diff만 전달하고, 나머지는 파일명+변경 라인 수 목록으로 대체한다.

## 도구 사용 규칙

Bash 대신 Claude 전용 도구를 우선 사용한다:

| 작업 | 사용할 도구 | Bash 사용 금지 |
|------|-----------|---------------|
| 에이전트 지침 파일 읽기 | **Read** | `cat`, `head` |
| 리뷰 보고서 저장 | **Write** (디렉토리 자동 생성) | `mkdir -p` + `echo >` |
| 보고서 업데이트 | **Edit** | `sed` |
| 파일 검색 | **Glob** | `find`, `ls` |
| 코드 내용 검색 | **Grep** | `grep`, `rg` |

Bash는 아래 경우에만 사용한다:
- `gh` CLI 명령 (PR 조회, diff, 코멘트 게시 등)
- `npx` / `yarn` 명령 (타입 체크, 테스트 실행)

## 오류 처리

| 상황 | 메시지 |
|------|--------|
| GitHub 인증 실패 | `[ERROR] GitHub authentication required. Run: gh auth login` |
| PR 미존재 | `[ERROR] PR #<NUMBER>을 찾을 수 없습니다.` |
| 서브 에이전트 전원 실패 | `[ERROR] 모든 에이전트가 실패했습니다. 다시 시도해주세요.` |
| 일부 에이전트 실패 | 실패한 에이전트 이름을 로그하고, 성공한 에이전트 결과로 진행 |
