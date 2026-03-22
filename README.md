# roto-subagent-band

멀티 서브에이전트 기반 코드 리뷰 + 구현 계획 플러그인 for Claude Code.

## 포함 스킬 (10개)

### 코드 리뷰 계열

| 스킬 | 설명 |
|------|------|
| `/roto-band-code-review` | PR을 7개 에이전트가 병렬 리뷰 → 인라인 코멘트 + PR 요약 게시 |
| `/roto-band-self-dev` | 리뷰 후 사용자와 협의하며 직접 코드 수정 (GitHub 코멘트 없음) |
| `/roto-band-reinforce` | self-dev 리뷰→수정을 자동 반복, Critical/Major 0이 될 때까지 강화 |
| `/roto-band-test-gen` | 변경 코드를 7에이전트가 분석 → 누락 테스트 식별 → 자동 생성 |
| `/roto-band-review-apply` | PR에 남겨진 리뷰 코멘트를 수집 → 타당성 검토 → 코드 반영 |

### 구현 계획 계열

| 스킬 | 설명 |
|------|------|
| `/roto-band-plan` | 5개 에이전트가 코드베이스 분석 → 다각도 구현 계획 수립 |
| `/roto-band-plan-execute` | 구현 계획 markdown 기반 Phase별 구현 + 5에이전트 검증 루프 |
| `/roto-band-plan-reinforce` | 계획을 N회 반복 검토, Critical/Major 시 수정 후 카운터 리셋 재검토 |
| `/roto-band-plan-review` | 기존 구현 계획을 5개 에이전트가 검토 → 누락/리스크/개선점 보고 |

### 유틸리티

| 스킬 | 설명 |
|------|------|
| `/roto-band-check-deps` | 서브에이전트가 참조하는 외부 스킬 설치 여부 확인 + 자동 설치 |

## 서브에이전트

### 코드 리뷰용 (7명)

| 에이전트 | 관점 | 참조 스킬 |
|----------|------|-----------|
| UI Designer | 웹 접근성(WCAG), UX 패턴, 반응형 디자인, 시각적 일관성 | `web-design-guidelines`, `vercel-composition-patterns`, `tailwind-design-system`, `vercel:shadcn` |
| Senior FE Engineer | 아키텍처, 성능, 보안, 프로젝트 규칙 준수 | `vercel-react-best-practices`, `next-best-practices`, `vercel-composition-patterns`, `vercel:nextjs`, `vercel:vercel-functions` |
| Junior FE Engineer | 가독성, 네이밍, 복잡성, 중복 패턴 | `react`, `simplify` |
| Backend Engineer | API 데이터 흐름, 에러 핸들링, 타입 안전성, 비동기 처리 | `turborepo`, `vercel:vercel-storage`, `vercel:vercel-functions` |
| SEO Specialist | 메타데이터, URL 구조, 구조화 데이터, Core Web Vitals | `seo-audit`, `vercel:satori` |
| i18n Reviewer | 번역 누락, 하드코딩 문자열, 날짜/숫자 포맷 | — |
| Claude Config Reviewer | CLAUDE.md, rules, skills 설정 파일 품질 | — |

### 구현 계획용 (5명)

| 에이전트 | 관점 | 참조 스킬 |
|----------|------|-----------|
| Architect | 시스템 설계, 컴포넌트 경계, 데이터 흐름, 확장성 | `vercel:nextjs`, `vercel-composition-patterns`, `turborepo` |
| Security Engineer | 인증/인가, 데이터 보호, 입력 검증, 공격 표면 | `security-guidance`, `vercel:vercel-firewall`, `vercel:auth` |
| Performance Engineer | 렌더링 전략, 번들 사이즈, 캐싱, Core Web Vitals | `vercel-react-best-practices`, `vercel:observability`, `vercel:vercel-functions` |
| QA Engineer | 테스트 가능성, 엣지 케이스, 에러 시나리오, 회귀 리스크 | `vercel:nextjs`, `next-best-practices` |
| DX Engineer | API 인체공학, 코드 컨벤션, 유지보수성, 개발 워크플로우 | `simplify`, `vercel-composition-patterns`, `turborepo` |

## 설치

```bash
# 1. 마켓플레이스 등록
/plugin marketplace add rotoshine/roto-subagent-band

# 2. 플러그인 설치
/plugin install roto-subagent-band@rotoshine-roto-subagent-band

# 3. 플러그인 리로드 (설치 직후 또는 업데이트 반영 시)
/reload-plugins
```

## 설치 후 의존성 체크

```bash
/roto-band-check-deps
```

외부 참조 스킬(web-design-guidelines, react-best-practices 등)이 설치되어 있으면
서브에이전트가 더 깊이 있는 리뷰를 수행합니다.

## 사용 예시

```bash
# 코드 리뷰
/roto-band-code-review              # 현재 브랜치 PR 리뷰
/roto-band-code-review 64 --silent  # 특정 PR silent 리뷰
/roto-band-self-dev                 # 리뷰 + 직접 수정
/roto-band-reinforce                # 반복 리뷰+수정 (Critical/Major 0까지)
/roto-band-reinforce --max-rounds 5 # 최대 5회 반복
/roto-band-test-gen                 # 변경 코드 기반 테스트 자동 생성
/roto-band-test-gen --scope all     # unit + integration + e2e 전체
/roto-band-review-apply             # 리뷰 코멘트 반영

# 구현 계획
/roto-band-plan 사용자 프로필에 활동 타임라인 추가
/roto-band-plan-execute docs/plan-timeline.md  # 계획 파일로 구현
/roto-band-plan-execute 관리자 대시보드 구현    # 계획 수립→구현 한번에
/roto-band-plan-reinforce plans/plan-auth.md     # 계획 강화 검토 (3회 반복)
/roto-band-plan-reinforce plans/plan.md --rounds 5  # 5회 반복 검토
/roto-band-plan-review docs/plan.md             # 기존 계획 1회 검토
```

## 라이선스

MIT
