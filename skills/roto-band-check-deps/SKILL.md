---
name: roto-band-check-deps
description: 서브에이전트 코드 리뷰에 필요한 외부 참조 스킬의 설치 여부를 확인하고, 미설치 스킬을 자동으로 설치합니다. "의존성 확인", "스킬 의존성 체크", "check deps", "install review skills" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 외부 참조 스킬 의존성 체크

서브에이전트들이 참조하는 외부 스킬의 설치 여부를 확인하고, 누락된 스킬을 자동 설치합니다.

**모든 출력은 한국어로 작성한다.**

## 사용 방법

```text
/roto-band-check-deps [--install] [--dry-run]
```

| 인자 | 설명 |
|------|------|
| `--install` | 미설치 스킬을 자동 설치 (기본값) |
| `--dry-run` | 설치 없이 상태만 확인 |

인자 없이 실행하면 `--install`과 동일하게 동작합니다.

---

## 참조 스킬 목록

서브에이전트들이 리뷰 품질을 높이기 위해 참조하는 외부 스킬 목록입니다.
이 스킬들이 설치되어 있으면 서브에이전트가 더 깊이 있는 리뷰를 수행합니다.

### 필수 (Recommended)

| 스킬 | 소스 | 참조 에이전트 |
|------|------|-------------|
| `web-design-guidelines` | `vercel-labs/skills` | UI Designer |
| `vercel-composition-patterns` | `vercel-labs/skills` | UI Designer, Senior FE |
| `tailwind-design-system` | `vercel-labs/skills` | UI Designer |
| `vercel-react-best-practices` | `vercel-labs/skills` | Senior FE |
| `next-best-practices` | `vercel-labs/next-skills` | Senior FE |
| `simplify` | `anthropics/claude-code-skills` | Junior FE |
| `react` | `vercel-labs/skills` | Junior FE |
| `seo-audit` | `vercel-labs/skills` | SEO Specialist |

### 구현 계획용 (Recommended)

| 스킬 | 소스 | 참조 에이전트 |
|------|------|-------------|
| `security-guidance` | `claude-plugins-official` | Security Engineer |

### 선택 (Optional, Vercel 플러그인 포함)

아래 스킬들은 Vercel 플러그인(`vercel@claude-plugins-official`)에 포함되어 있습니다.
Vercel 플러그인이 설치되어 있으면 별도 설치 없이 자동으로 사용됩니다.

| 스킬 | 참조 에이전트 |
|------|-------------|
| `vercel:shadcn` | UI Designer |
| `vercel:nextjs` | Senior FE, Architect, QA Engineer |
| `vercel:vercel-functions` | Senior FE, BE Engineer, Performance Engineer |
| `vercel:vercel-storage` | BE Engineer |
| `vercel:satori` | SEO Specialist |
| `vercel:vercel-firewall` | Security Engineer |
| `vercel:auth` | Security Engineer |
| `vercel:observability` | Performance Engineer |
| `turborepo` | BE Engineer, Architect, DX Engineer |

---

## 워크플로우

### Step 1: 현재 설치된 스킬 확인

프로젝트의 `skills-lock.json` 파일을 읽어 설치된 스킬 목록을 파악한다:

```
Read skills-lock.json
```

파일이 없으면 아무 스킬도 설치되지 않은 것으로 간주한다.

또한 `.claude/settings.json`의 `enabledPlugins`를 확인하여 Vercel 플러그인 설치 여부를 판단한다.

### Step 2: 미설치 스킬 목록 작성

필수 스킬 중 설치되지 않은 스킬을 표로 출력한다:

```markdown
## 📋 스킬 의존성 상태

### 필수 스킬
| 스킬 | 소스 | 상태 |
|------|------|------|
| web-design-guidelines | vercel-labs/skills | ✅ 설치됨 |
| vercel-react-best-practices | vercel-labs/skills | ❌ 미설치 |
| next-best-practices | vercel-labs/next-skills | ❌ 미설치 |
| simplify | anthropics/claude-code-skills | ✅ 설치됨 |
...

### Vercel 플러그인 스킬
| 스킬 | 상태 |
|------|------|
| vercel:shadcn | ✅ (Vercel 플러그인 활성) |
| vercel:nextjs | ✅ (Vercel 플러그인 활성) |
...
```

### Step 3: 자동 설치 (`--install` 모드)

`--dry-run`이 아닌 경우, 미설치 스킬을 소스별로 그룹핑하여 설치한다:

```bash
# vercel-labs/skills에서 여러 스킬 설치
npx claude-code skills add vercel-labs/skills --skill web-design-guidelines
npx claude-code skills add vercel-labs/skills --skill vercel-composition-patterns
npx claude-code skills add vercel-labs/skills --skill tailwind-design-system
npx claude-code skills add vercel-labs/skills --skill vercel-react-best-practices

# 다른 소스
npx claude-code skills add vercel-labs/next-skills --skill next-best-practices
npx claude-code skills add anthropics/claude-code-skills --skill simplify
```

> **참고**: 위 설치 명령어의 정확한 형식은 Claude Code 버전에 따라 다를 수 있습니다.
> 설치 실패 시 사용자에게 수동 설치 명령어를 안내합니다.

### Step 4: 결과 요약

```markdown
## ✅ 의존성 체크 완료

- 필수 스킬: N/8 설치됨
- 신규 설치: N건
- 설치 실패: N건
- Vercel 플러그인: 활성/비활성

### 설치 실패 항목 (수동 설치 필요)
- `skill-name`: 에러 메시지
  → 수동 설치: `npx claude-code skills add source --skill skill-name`
```

---

## Vercel 플러그인 미설치 시 안내

Vercel 플러그인이 설치되어 있지 않고 선택 스킬이 필요한 경우:

```
ℹ️ Vercel 플러그인(`vercel@claude-plugins-official`)이 설치되어 있지 않습니다.
설치하면 shadcn, Next.js, Functions, Storage, Satori 등의 스킬을 자동으로 사용할 수 있습니다.

설치 방법:
  claude plugins add vercel
```

## 오류 처리

| 상황 | 처리 |
|------|------|
| skills-lock.json 없음 | 아무 스킬도 설치되지 않은 것으로 간주 |
| 설치 명령 실패 | 에러 로그 + 수동 설치 명령 안내 |
| 네트워크 에러 | 재시도 1회, 실패 시 스킵 |
| 소스 레포 미존재 | 해당 스킬 스킵 + 사용자 안내 |
