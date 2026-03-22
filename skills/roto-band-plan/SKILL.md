---
name: roto-band-plan
description: 구현 작업에 대해 5개 전문 서브에이전트(Architect, Security, Performance, QA, DX)가 병렬로 코드베이스를 분석하고, 다각도 관점이 반영된 구현 계획을 수립합니다. "구현 계획", "계획 세워줘", "plan", "implementation plan", "설계해줘" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 멀티 에이전트 구현 계획 수립

5개 전문 서브에이전트가 코드베이스를 분석하고, 각자의 관점에서 고려사항을 제시하면 이를 통합하여 구현 계획을 수립하는 스킬입니다.

**모든 출력은 한국어로 작성한다.**

## 사용 방법

```text
/roto-band-plan <작업 설명>
```

| 인자 | 설명 |
|------|------|
| `작업 설명` | 구현할 기능/작업에 대한 설명 (필수) |

### 예시

```text
/roto-band-plan 아티스트 프로필 페이지에 공연 히스토리 타임라인 추가
/roto-band-plan 티켓 예약 시스템의 동시성 처리 개선
/roto-band-plan 관리자 대시보드 페이지 구현
```

---

## 워크플로우

### Step 1: 작업 범위 파악

1. 사용자의 작업 설명을 분석한다.
2. 관련 코드를 탐색한다:
   - Glob/Grep으로 관련 파일 식별
   - 핵심 파일을 Read로 읽어 현재 구조 파악
   - CLAUDE.md, `.claude/rules/` 규칙 확인
3. 작업에 영향을 받는 영역을 정리한다:
   - 변경이 필요한 파일/모듈 목록
   - 의존하는 API/데이터 모델
   - 관련 기존 컴포넌트/유틸리티

### Step 2: 5개 서브에이전트 병렬 실행

**반드시 Agent 도구를 사용하여 5개의 서브에이전트를 동시에 병렬 실행한다.**

각 에이전트에게 전달할 프롬프트에는 반드시 아래 정보를 모두 포함한다:

1. **에이전트 역할 지침**: 이 플러그인의 `agents/planning/<name>.md` 파일 전체 내용
2. **작업 설명**: 사용자가 제공한 구현 작업 설명
3. **코드베이스 컨텍스트**: Step 1에서 수집한 관련 코드, 파일 구조, 프로젝트 규칙
4. **기존 패턴**: 유사한 기존 구현이 있으면 해당 코드

에이전트별 설정:

| 에이전트 | 파일 경로 | subagent_type | model |
|----------|----------|---------------|-------|
| Architect | `agents/planning/architect.md` | general-purpose | sonnet |
| Security Engineer | `agents/planning/security-engineer.md` | general-purpose | sonnet |
| Performance Engineer | `agents/planning/performance-engineer.md` | general-purpose | sonnet |
| QA Engineer | `agents/planning/qa-engineer.md` | general-purpose | sonnet |
| DX Engineer | `agents/planning/dx-engineer.md` | general-purpose | sonnet |

각 에이전트는 JSON 배열로 고려사항을 반환한다:

```
[
  {
    "category": "카테고리",
    "severity": "Critical|Major|Minor|Misc",
    "title": "제목",
    "description": "설명",
    "suggestion": "제안"
  }
]
```

### Step 3: 결과 통합 및 계획 수립

에이전트 결과를 통합하여 구현 계획을 수립한다.

#### 3-1: 에이전트 응답 파싱

1. 각 에이전트의 응답을 JSON 배열로 파싱
2. 실패 시 `[...]` 패턴 추출 재시도
3. 각 고려사항에 `agent` 필드 추가

#### 3-2: 고려사항 분류

모든 고려사항을 카테고리별로 정리:
- 🔴 Critical: 반드시 계획에 반영
- 🟠 Major: 가능한 반영
- 🟡 Minor: 참고 사항
- 🔵 Misc: 선택적 적용

#### 3-3: 구현 계획 작성

아래 형식으로 구현 계획을 사용자에게 출력한다:

```markdown
## 📋 구현 계획: <작업 제목>

### 개요
<작업 목표와 범위 1-2문장 요약>

### 아키텍처 결정
- <Architect 에이전트의 주요 결정 사항>
- <컴포넌트 구조, 데이터 흐름 등>

### 구현 단계

#### Phase 1: <단계 이름>
- [ ] 작업 1 — `파일경로`
- [ ] 작업 2 — `파일경로`

#### Phase 2: <단계 이름>
- [ ] 작업 3 — `파일경로`
- [ ] 작업 4 — `파일경로`

(필요한 만큼 Phase 추가)

### 보안 고려사항
- <Security 에이전트의 Critical/Major 이슈>

### 성능 고려사항
- <Performance 에이전트의 핵심 권고>

### 테스트 계획
- <QA 에이전트의 테스트 전략>
- 단위 테스트: ...
- 통합 테스트: ...
- 엣지 케이스: ...

### 주의사항
- <에이전트들이 제기한 리스크/주의점>

### 변경 영향 범위
- 수정 파일: N개
- 신규 파일: N개
- 영향받는 기존 기능: ...
```

### Step 4: 사용자 확인

AskUserQuestion 도구를 사용하여 사용자에게 계획을 확인받는다.

질문 예시:
```
위 구현 계획을 확인해주세요.

- 수정이 필요한 부분이 있으면 알려주세요
- 추가하거나 제거할 단계가 있으면 지정해주세요
- 확정되면 "확정" 또는 "OK"라고 답해주세요
```

사용자가 수정을 요청하면 계획을 업데이트하고, 확정하면 Step 5로 진행한다.

### Step 5: 계획 저장

확정된 계획을 markdown 파일로 저장한다.

**저장 경로**: `plans/plan-<slugified-title>.md`

Write 도구로 파일을 생성한다 (부모 디렉토리 자동 생성).

저장 후 사용자에게 경로를 알린다:

```
📋 구현 계획이 저장되었습니다: plans/plan-<title>.md
이 파일을 `/roto-band-plan-execute`에 전달하면 계획대로 구현할 수 있습니다.
```

---

## 도구 사용 규칙

| 작업 | 사용할 도구 |
|------|-----------|
| 코드베이스 탐색 | **Glob**, **Grep**, **Read** |
| 에이전트 지침 읽기 | **Read** |
| 사용자 질문 | **AskUserQuestion** |
| 서브에이전트 실행 | **Agent** (5개 병렬) |

Bash는 git 상태 확인 등 시스템 명령에만 사용한다.

## 오류 처리

| 상황 | 처리 |
|------|------|
| 작업 설명 없음 | `[ERROR] 구현할 작업을 설명해주세요.` |
| 관련 코드 없음 | 새 기능으로 간주하고 프로젝트 구조만으로 계획 수립 |
| 서브에이전트 전원 실패 | `[ERROR] 모든 에이전트가 실패했습니다. 다시 시도해주세요.` |
| 일부 에이전트 실패 | 실패 에이전트 로그, 나머지 결과로 진행 |
