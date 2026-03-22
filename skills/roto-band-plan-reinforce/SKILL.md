---
name: roto-band-plan-reinforce
description: 구현 계획을 5개 전문 서브에이전트가 반복 검토하여 품질을 강화합니다. N회 연속 검토를 수행하고, Critical/Major 발견 시 계획을 수정한 뒤 검토 카운터를 리셋하여 다시 N회 검토합니다. "계획 강화", "계획 반복 검토", "plan reinforce", "계획 완벽하게 검토해줘" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 계획 강화 검토 루프 (Plan Reinforce)

5개 전문 서브에이전트가 구현 계획을 반복 검토하고, Critical/Major 이슈 발견 시 계획을 수정한 뒤 검토를 처음부터 다시 시작하는 강화 검토 스킬입니다.

**모든 출력은 한국어로 작성한다.**

## 사용 방법

```text
/roto-band-plan-reinforce <계획 파일 경로> [--rounds N] [--max-resets M]
```

| 인자 | 설명 |
|------|------|
| `계획 파일 경로` | 검토할 계획 markdown 파일 경로 (필수). `/roto-band-plan`이 생성한 파일 또는 직접 작성한 계획 |
| `--rounds N` | 연속 검토 횟수 (선택, 기본값: 3, 최소: 2, 최대: 5) |
| `--max-resets M` | 최대 리셋 횟수 (선택, 기본값: 3, 최대: 5). 무한 루프 방지용 |

### 예시

```text
/roto-band-plan-reinforce plans/plan-timeline.md              # 3회 검토, 최대 3회 리셋
/roto-band-plan-reinforce plans/plan-auth.md --rounds 5       # 5회 검토
/roto-band-plan-reinforce plans/plan-dashboard.md --max-resets 2  # 최대 2회 리셋
```

---

## 핵심 개념

```
[검토 1/3] → [검토 2/3] → [검토 3/3] → ✅ 완료!
                 ↓ Critical/Major 발견
            [계획 수정] → 카운터 리셋
                 ↓
[검토 1/3] → [검토 2/3] → [검토 3/3] → ✅ 완료!
                              ↓ Critical/Major 발견
                         [계획 수정] → 카운터 리셋
                              ↓
                      [검토 1/3] → ...
```

- 매 라운드 Critical/Major 유무와 무관하게 N회 연속 검토를 목표로 한다
- 검토 중 Critical/Major가 발견되면 → 계획 수정 → 카운터 리셋 → 다시 1/N부터
- Critical/Major 없이 N회 연속 통과하면 → 강화 완료
- 리셋이 `--max-resets`에 도달하면 → 잔여 이슈 보고 후 종료

---

## 워크플로우

### Step 1: 인자 파싱 및 계획 로드

1. `--rounds N` 파싱. 없으면 기본값 3. 범위 2~5로 클램프.
2. `--max-resets M` 파싱. 없으면 기본값 3. 최대 5로 클램프.
3. 계획 파일을 Read 도구로 읽는다.
   - 파일 없으면 에러: `[ERROR] 계획 파일을 찾을 수 없습니다: <경로>`
4. 관련 코드베이스를 탐색한다:
   - 계획에 언급된 파일/모듈을 Glob/Grep으로 식별
   - 핵심 파일을 Read로 읽어 현재 상태 파악
   - CLAUDE.md, `.claude/rules/` 규칙 확인
5. 리포트 파일 경로 결정: `code-reviews/plan-reinforce-<slugified-title>.md`
6. 사용자에게 시작 알림:
   ```
   🔄 계획 강화 검토를 시작합니다. (연속 N회 검토, 최대 M회 리셋)
   ```

### Step 2: 검토 루프

카운터를 초기화한다: `pass = 0`, `reset_count = 0`

#### 2-A: 5에이전트 검토 실행

**반드시 Agent 도구를 사용하여 5개의 서브에이전트를 동시에 병렬 실행한다.**

각 에이전트에게 전달할 프롬프트:

1. **에이전트 역할 지침**: `agents/planning/<name>.md` 전체 내용
2. **구현 계획 전문**: 현재 계획 내용 (수정된 최신 버전)
3. **코드베이스 컨텍스트**: Step 1에서 수집한 관련 코드, 프로젝트 규칙
4. **이전 검토 이력**: 이전 라운드에서 보고된 이슈와 수정 내역 (있으면)
5. **검토 요청**:

```
위 구현 계획을 당신의 전문 관점에서 검토하세요.

이전에 수정된 이력이 있다면, 수정이 적절한지도 함께 확인하세요.
이전에 보고했던 이슈가 해소되었으면 다시 보고하지 마세요.
새로운 이슈만 보고하세요.

응답은 반드시 JSON 배열이어야 합니다.
마크다운 코드블록(```)으로 감싸지 마세요. 앞뒤 설명 텍스트도 넣지 마세요.
순수한 JSON 배열만 반환하세요.

[
  {
    "category": "카테고리",
    "severity": "Critical|Major|Minor|Misc",
    "title": "이슈 제목",
    "description": "상세 설명",
    "suggestion": "수정 제안",
    "planSection": "영향받는 계획 섹션 (예: Phase 2, 보안 고려사항 등)"
  }
]

이슈가 없으면 빈 배열 []을 반환하세요.
```

에이전트별 설정:

| 에이전트 | 파일 경로 | subagent_type | model |
|----------|----------|---------------|-------|
| Architect | `agents/planning/architect.md` | general-purpose | sonnet |
| Security Engineer | `agents/planning/security-engineer.md` | general-purpose | sonnet |
| Performance Engineer | `agents/planning/performance-engineer.md` | general-purpose | sonnet |
| QA Engineer | `agents/planning/qa-engineer.md` | general-purpose | sonnet |
| DX Engineer | `agents/planning/dx-engineer.md` | general-purpose | sonnet |

#### 2-B: 결과 통합

1. 에이전트 응답 파싱 (JSON → fallback `[...]` 추출 → 실패 시 스킵)
2. 각 항목에 `agent` 필드 추가
3. 중복 통합: 같은 category + 유사 title → 최고 severity로 통합, 에이전트 병기
4. Severity 순 정렬: Critical → Major → Minor → Misc

#### 2-C: 분기 판단

`pass`를 1 증가시킨다.

사용자에게 현재 상태 출력:

```
📋 Pass N/M (리셋 R회): 🔴 N / 🟠 N / 🟡 N / 🔵 N
```

**Case 1: Critical/Major가 0건**

→ 리포트에 이번 pass 결과를 기록하고 다음 pass로 진행.

`pass == rounds`이면 → Step 3(최종 요약)으로 진행.

**Case 2: Critical/Major가 있음**

→ 계획 수정 후 카운터 리셋.

1. 사용자에게 발견된 Critical/Major 이슈를 보여준다:

```markdown
⚠️ Critical/Major 이슈 발견 — 계획 수정 후 검토를 다시 시작합니다.

| # | Severity | 에이전트 | 제목 | 영향 섹션 |
|---|----------|----------|------|----------|
| 1 | 🔴 | Security | API 인증 누락 | Phase 2 |
| 2 | 🟠 | Architect | 데이터 모델 비효율 | Phase 1 |

<각 이슈 상세 + 수정 제안>
```

2. `reset_count`를 확인:
   - `reset_count >= max_resets` → Step 3으로 진행 (잔여 이슈 보고)
   - 그렇지 않으면 → 계획 수정 진행

3. 계획 수정 실행:
   - 각 Critical/Major 이슈의 `suggestion`과 `planSection`을 기반으로 계획 파일을 Edit 도구로 수정
   - 수정 원칙:
     - 이슈의 `suggestion`을 반영하되, 기존 계획의 구조와 톤을 유지
     - 한 이슈 수정이 다른 섹션에 영향을 줄 수 있으므로 관련 섹션도 확인
     - 수정 범위는 이슈 지적 사항으로 한정
   - Minor/Misc 이슈는 수정하지 않는다 (참고 사항으로 리포트에만 기록)

4. 수정 내역을 리포트와 사용자에게 출력:

```
🔧 계획 수정 완료 (N건). 검토 카운터를 리셋합니다. (리셋 R/M)
```

5. `pass = 0`, `reset_count += 1`로 카운터 리셋
6. → Step 2-A로 돌아가 다시 검토 시작

### Step 3: 최종 요약 및 리포트 저장

#### 3-1: 리포트 저장

리포트 파일(`code-reviews/plan-reinforce-<title>.md`)에 전체 과정을 기록한다.

리포트 템플릿:

```markdown
# 계획 강화 검토 리포트

## 메타데이터

| 항목 | 값 |
|------|-----|
| 계획 파일 | `<파일 경로>` |
| 검토 일시 | <YYYY-MM-DD HH:MM> |
| 검토 스킬 | `/roto-band-plan-reinforce` |
| 설정 | 연속 N회, 최대 리셋 M회 |
| 실제 리셋 | R회 |
| 총 검토 | N회 |
| 최종 상태 | ✅ N회 연속 통과 / ⚠️ 최대 리셋 도달 |

## 검토 이력

### 초기 검토 (리셋 0)

#### Pass 1/3
| Severity | 개수 |
|----------|------|
| 🔴 Critical | 0 |
| 🟠 Major | 0 |
| 🟡 Minor | N |
| 🔵 Misc | N |

#### Pass 2/3
(같은 형식)

⚠️ Critical/Major 발견 → 계획 수정

#### 수정 내역
| # | 이슈 | 에이전트 | 수정 내용 |
|---|------|----------|----------|
| 1 | API 인증 누락 | Security | Phase 2에 인증 미들웨어 단계 추가 |

### 리셋 1 검토

#### Pass 1/3
(같은 형식)

#### Pass 2/3
(같은 형식)

#### Pass 3/3
(같은 형식)

✅ 3회 연속 통과

## 최종 결과

### 수정된 항목
- ✅ [에이전트명] 이슈 제목 (리셋 N에서 수정)

### 잔여 이슈 (Minor/Misc — 수정하지 않음)
- 🟡 [에이전트명] 이슈 제목

### 잔여 이슈 (미해소 Critical/Major — max-resets 도달 시만)
- 🔴 [에이전트명] 이슈 제목

---
🤖 Generated by `/roto-band-plan-reinforce` | iterative plan review with auto-correction
```

#### 3-2: 대화 창 최종 요약

```markdown
## 🏁 계획 강화 검토 완료

| 항목 | 값 |
|------|-----|
| 총 검토 | N회 |
| 리셋 | R회 |
| 계획 수정 | N건 |
| 잔여 이슈 | 🟡 N / 🔵 N |
| 계획 파일 | `<파일 경로>` (수정 반영됨) |
| 리포트 | `code-reviews/plan-reinforce-<title>.md` |
```

수정 후 계획 파일로 구현을 시작할 수 있음을 안내한다:

```
💡 `/roto-band-plan-execute <파일 경로>`로 이 계획을 구현할 수 있습니다.
```

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
| 리포트 저장 | **Write** | `echo >` |

Bash는 git 상태 확인 등 시스템 명령에만 사용한다.

## 오류 처리

| 상황 | 처리 |
|------|------|
| 계획 파일 없음 | `[ERROR] 계획 파일을 찾을 수 없습니다: <경로>` |
| 인자 없음 | `[ERROR] 계획 파일 경로를 입력해주세요.` |
| 서브에이전트 전원 실패 | `[ERROR] 모든 에이전트가 실패했습니다. 다시 시도해주세요.` |
| 일부 에이전트 실패 | 실패 에이전트 로그, 나머지 결과로 진행 |
| max-resets 도달 | 잔여 이슈 보고 후 종료 |
| 동일 이슈 반복 | 2회 리셋 연속 같은 category+title 이슈 시 해당 이슈 스킵 (무한 루프 방지) |
| 이슈 0건 (전 pass) | 모든 pass에서 이슈 0이면 `✅ 계획이 매우 잘 수립되어 있습니다!` 출력 |
