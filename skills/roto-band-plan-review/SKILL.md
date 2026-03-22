---
name: roto-band-plan-review
description: 기존 구현 계획을 5개 전문 서브에이전트(Architect, Security, Performance, QA, DX)가 병렬로 검토하여 누락된 고려사항, 리스크, 개선점을 찾아냅니다. "계획 검토", "plan review", "계획 리뷰", "설계 리뷰", "계획 점검" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 멀티 에이전트 구현 계획 검토

기존 구현 계획을 5개 전문 서브에이전트가 각자의 관점에서 검토하고, 누락된 사항/리스크/개선점을 통합 보고하는 스킬입니다.

**모든 출력은 한국어로 작성한다.**

## 사용 방법

```text
/roto-band-plan-review [계획 파일 경로]
```

| 인자 | 설명 |
|------|------|
| `계획 파일 경로` | 검토할 계획 파일 경로 (선택). 생략 시 대화에서 계획 내용을 직접 입력받음 |

### 예시

```text
/roto-band-plan-review docs/plan-ticket-system.md
/roto-band-plan-review                              # 대화에서 계획 입력
```

---

## 워크플로우

### Step 1: 계획 내용 수집

1. **파일 경로가 제공된 경우**: Read 도구로 파일을 읽는다.
2. **파일 경로가 없는 경우**: AskUserQuestion으로 계획 내용을 입력받는다:
   ```
   검토할 구현 계획을 붙여넣거나, 계획 파일의 경로를 알려주세요.
   ```
3. 계획 내용이 너무 짧으면(50자 미만) 추가 정보를 요청한다.

### Step 2: 관련 코드 수집

계획에 언급된 파일/모듈/컴포넌트를 기반으로 코드베이스를 탐색한다:
- Glob/Grep으로 관련 파일 식별
- 핵심 파일을 Read로 읽어 현재 상태 파악
- CLAUDE.md, `.claude/rules/` 규칙 확인

### Step 3: 5개 서브에이전트 병렬 실행

**반드시 Agent 도구를 사용하여 5개의 서브에이전트를 동시에 병렬 실행한다.**

각 에이전트에게 전달할 프롬프트:

1. **에이전트 역할 지침**: `agents/planning/<name>.md` 전체 내용
2. **구현 계획 전문**: 사용자가 제공한 계획 내용
3. **코드베이스 컨텍스트**: 관련 코드, 프로젝트 규칙
4. **검토 요청**: "위 구현 계획을 당신의 전문 관점에서 검토하고, 누락되거나 개선이 필요한 사항을 보고해주세요."

에이전트별 설정:

| 에이전트 | 파일 경로 | subagent_type | model |
|----------|----------|---------------|-------|
| Architect | `agents/planning/architect.md` | general-purpose | sonnet |
| Security Engineer | `agents/planning/security-engineer.md` | general-purpose | sonnet |
| Performance Engineer | `agents/planning/performance-engineer.md` | general-purpose | sonnet |
| QA Engineer | `agents/planning/qa-engineer.md` | general-purpose | sonnet |
| DX Engineer | `agents/planning/dx-engineer.md` | general-purpose | sonnet |

### Step 4: 결과 통합 및 검토 보고서 출력

#### 4-1: 에이전트 응답 파싱

1. 각 에이전트의 JSON 배열 응답을 파싱
2. 각 항목에 `agent` 필드 추가
3. 중복 통합: 같은 category + 유사 title → 최고 severity로 통합, 에이전트 병기

#### 4-2: 검토 결과 출력

```markdown
## 🔍 구현 계획 검토 결과

### 요약

| Severity | 개수 |
|----------|------|
| 🔴 Critical | N |
| 🟠 Major | N |
| 🟡 Minor | N |
| 🔵 Misc | N |

### 에이전트별 관점 요약

- **Architect**: 한 줄 요약
- **Security**: 한 줄 요약
- **Performance**: 한 줄 요약
- **QA**: 한 줄 요약
- **DX**: 한 줄 요약

### 상세 피드백

#### 🔴 Critical

| # | 카테고리 | 에이전트 | 제목 |
|---|----------|----------|------|
| 1 | 인증/인가 | Security | 관리자 API 인증 누락 |

**#1 상세**
- **설명**: ...
- **제안**: ...

#### 🟠 Major
(같은 형식)

#### 🟡 Minor
(같은 형식)

#### 🔵 Misc
(같은 형식)

### 권장 수정사항

계획에 반영을 권장하는 항목을 우선순위로 정리:
1. [Critical] ...
2. [Major] ...
3. [Major] ...
```

### Step 5: 계획 업데이트 제안

AskUserQuestion으로 사용자에게 후속 조치를 묻는다:

```
검토 결과를 확인해주세요.

- 계획을 수정하시겠습니까? (수정할 항목 번호를 지정)
- `/roto-band-plan`으로 새 계획을 세우시겠습니까?
- 현재 계획대로 진행하시겠습니까?
```

사용자가 수정을 요청하면:
- 기존 계획에 검토 결과를 반영한 **수정된 계획**을 출력한다
- 계획 파일이 제공된 경우, Edit 도구로 파일도 업데이트한다

---

## 도구 사용 규칙

| 작업 | 사용할 도구 |
|------|-----------|
| 계획 파일 읽기 | **Read** |
| 코드베이스 탐색 | **Glob**, **Grep**, **Read** |
| 에이전트 지침 읽기 | **Read** |
| 사용자 질문/확인 | **AskUserQuestion** |
| 서브에이전트 실행 | **Agent** (5개 병렬) |
| 계획 파일 수정 | **Edit** |

Bash는 git 상태 확인 등 시스템 명령에만 사용한다.

## 오류 처리

| 상황 | 처리 |
|------|------|
| 계획 내용 없음 | AskUserQuestion으로 입력 요청 |
| 파일 경로 잘못됨 | `[ERROR] 파일을 찾을 수 없습니다: <path>` |
| 서브에이전트 전원 실패 | `[ERROR] 모든 에이전트가 실패했습니다. 다시 시도해주세요.` |
| 일부 에이전트 실패 | 실패 에이전트 로그, 나머지 결과로 진행 |
| 이슈 0건 | `✅ 검토 결과 특별한 이슈가 발견되지 않았습니다. 계획이 잘 수립되어 있습니다!` |
