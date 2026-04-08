# roto-claude-skills

개인 커스텀 스킬 모음 for Claude Code — 멀티 에이전트 코드 리뷰, 구현 계획, 스토리북 스크린샷 등.

## 포함 스킬 (22개)

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

### 스토리북

| 스킬 | 설명 |
|------|------|
| `/roto-storybook-screenshots` | 변경된 스토리를 Light/Dark 모드로 캡처 → PR에 스크린샷 테이블 반영 |
| `/roto-storybook-changelog` | 브랜치의 UI 변경사항을 분석하여 Storybook 패치노트(MDX + stories) 생성 |

### 테스트

| 스킬 | 설명 |
|------|------|
| `/roto-generate-tests` | 파일/컴포넌트에 대한 포괄적 테스트 스위트 자동 생성 |

### Git Worktree

| 스킬 | 설명 |
|------|------|
| `/roto-wt-create` | origin/main 기준으로 새 git worktree 생성 (worktrunk CLI) |
| `/roto-wt-switch` | worktree 목록 표시 및 전환 |
| `/roto-wt-clean` | worktree 삭제 및 브랜치 정리 |

### PR / 릴리즈

| 스킬 | 설명 |
|------|------|
| `/roto-pr-description` | 커밋 + diff 분석 → PR title/body 자동 생성/업데이트 |
| `/roto-release-notes` | 마지막 태그 이후 머지 PR 수집 → 릴리즈 노트 자동 생성 |

### 코드 품질

| 스킬 | 설명 |
|------|------|
| `/roto-dead-code` | 미사용 export, 미참조 파일, 순환 의존성 탐지 |
| `/roto-dep-audit` | outdated/vulnerable/deprecated 의존성 탐지 + 업그레이드 제안 |
| `/roto-a11y-audit` | WCAG 2.1 AA 기준 접근성 이슈 자동 검출 |

### 사고 도구

| 스킬 | 설명 |
|------|------|
| `/roto-ultra-think` | 다차원 분석과 깊은 문제 해결을 위한 구조화된 사고 프레임워크 |

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
# 1. 마켓플레이스 추가
/plugin marketplace add rotoshine/roto-claude-skills

# 2. 플러그인 설치
/plugin install roto-claude-skills@rotoshine-roto-claude-skills

# 3. 플러그인 리로드
/reload-plugins
```

또는 `/plugin` → **Marketplaces** 탭에서 `rotoshine/roto-claude-skills` 추가 → **Discover** 탭에서 설치할 수 있습니다.

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
/roto-band-review-apply             # 리뷰 코멘트 반영

# 구현 계획
/roto-band-plan 사용자 프로필에 활동 타임라인 추가
/roto-band-plan-execute docs/plan-timeline.md
/roto-band-plan-reinforce plans/plan.md
/roto-band-plan-review docs/plan.md

# 스토리북
/roto-storybook-screenshots                          # 변경된 스토리 자동 감지 + 캡처
/roto-storybook-screenshots --stories stamp,stamp-entered  # 특정 스토리만 캡처
/roto-storybook-changelog                            # UI 변경 패치노트 생성

# 테스트
/roto-generate-tests src/lib/auth.ts                 # 특정 파일 테스트 생성
/roto-generate-tests UserProfile                     # 컴포넌트 테스트 생성

# Git Worktree
/roto-wt-create fix/search-console                   # 새 worktree 생성
/roto-wt-switch                                      # worktree 목록 + 전환
/roto-wt-clean                                       # 현재 worktree 삭제

# PR / 릴리즈
/roto-pr-description                  # PR 자동 생성
/roto-pr-description --update         # 기존 PR body 업데이트
/roto-release-notes                   # 릴리즈 노트 생성
/roto-release-notes --since v1.2.0    # 특정 태그 이후

# 코드 품질
/roto-dead-code                       # dead code 탐지
/roto-dead-code --circular            # 순환 의존성만
/roto-dep-audit                       # 의존성 감사
/roto-dep-audit --security            # 보안 취약점만
/roto-a11y-audit                      # 접근성 감사
/roto-a11y-audit --pr                 # PR 변경 파일만

# 사고 도구
/roto-ultra-think 마이크로서비스 마이그레이션 vs 모놀리스 개선?
```

## 라이선스

MIT
