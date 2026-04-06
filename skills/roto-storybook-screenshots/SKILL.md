---
name: roto-storybook-screenshots
description: 현재 브랜치의 변경된 스토리북 스토리를 Light/Dark 모드로 캡처하여 스크린샷을 생성하고, PR body에 테이블로 반영합니다. "스토리북 스크린샷", "UI 스크린샷", "screenshot", "스크린샷 찍어줘", "PR에 스크린샷 추가" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# 스토리북 스크린샷 (Light / Dark)

현재 브랜치에서 변경된 스토리북 스토리를 감지하고, Light/Dark 테마 모두에서 스크린샷을 캡처하여 PR body에 반영하는 스킬입니다.

**모든 출력은 한국어로 작성한다.**

## 사용 방법

```text
/roto-storybook-screenshots [--stories story-id-1,story-id-2]
```

| 인자 | 설명 |
|------|------|
| `--stories` | 특정 스토리 ID만 캡처 (선택). 생략 시 변경된 stories 파일을 자동 감지 |

## 전제 조건

- **Storybook**: 프로젝트에 Storybook이 설정되어 있어야 함
- **Playwright**: `npx playwright screenshot` 명령이 가능해야 함 (`npx playwright install chromium`)
- **http-server**: `npx http-server`로 정적 서버 실행 가능해야 함

## 워크플로우

### Step 1: 스토리북 빌드 명령 탐지

프로젝트 구조를 분석하여 스토리북 빌드 명령과 출력 디렉토리를 자동 탐지합니다.

탐지 순서:
1. `package.json`의 `scripts`에서 `storybook:build`, `build-storybook`, `build:storybook` 등 패턴 탐색
2. 모노레포인 경우 root `package.json`의 workspace 스크립트 탐색 (예: `yarn workspace @scope/storybook build`)
3. `.storybook/main.ts`의 위치로 스토리북 루트 판별

탐지 실패 시 사용자에게 빌드 명령을 질문합니다.

### Step 2: 테마 설정 탐지

스토리북의 테마 전환 방식을 자동 탐지합니다.

탐지 순서:
1. `.storybook/preview.tsx` (또는 `.ts`, `.js`)에서 `globalTypes` 확인
2. `data-theme`, `globals.theme`, `context.globals` 패턴 탐색
3. 테마 값 추출 (예: `winter` / `night`, `light` / `dark`)

탐지 결과:
- **light 테마 이름**: 기본값 (예: `winter`, `light`)
- **dark 테마 이름**: 다크 테마 (예: `night`, `dark`)
- **globals 키 이름**: URL 파라미터에 사용할 키 (예: `theme`)

탐지 실패 시 기본값: `globals=theme:light` / `globals=theme:dark`

### Step 3: 변경된 스토리 감지

```bash
git fetch origin
git diff origin/main...HEAD --name-only | grep '\.stories\.tsx$'
```

변경된 `.stories.tsx` 파일에서 export된 스토리를 추출합니다:

```bash
grep '^export const' path/to/Changed.stories.tsx
```

스토리 ID 변환 규칙:
- 파일의 `meta.title` (예: `"Ticket/Variants/B — Zine DIY"`) → kebab-case로 변환
- export 이름 (예: `StampEntered`) → kebab-case로 변환
- 결합: `ticket-variants-b-zine-diy--stamp-entered`

`--stories` 인자가 지정된 경우 자동 감지를 스킵하고 지정된 ID만 사용합니다.

### Step 4: 스토리북 빌드

Step 1에서 탐지한 빌드 명령을 실행합니다.

```bash
# 예시
yarn workspace @scope/storybook build
```

빌드 실패 시 에러를 사용자에게 보여주고 종료합니다.

### Step 5: 스크린샷 캡처

정적 서버를 실행하고 각 스토리에 대해 Light/Dark 모드 스크린샷을 캡처합니다.

```bash
# 서버 실행
npx http-server {storybook-output-dir} -p 6007 --silent &
sleep 2

# 각 스토리에 대해
for story_id in ${STORY_IDS}; do
  # Light 모드
  npx playwright screenshot \
    --viewport-size="500,800" \
    --wait-for-timeout=1500 \
    --full-page \
    "http://localhost:6007/iframe.html?id=${story_id}&viewMode=story" \
    "{screenshots-dir}/${story_name}-light.png"

  # Dark 모드
  npx playwright screenshot \
    --viewport-size="500,800" \
    --wait-for-timeout=1500 \
    --full-page \
    "http://localhost:6007/iframe.html?id=${story_id}&viewMode=story&globals=${theme_key}:${dark_theme_name}" \
    "{screenshots-dir}/${story_name}-dark.png"
done

# 서버 종료
kill $(lsof -ti:6007) 2>/dev/null
```

**스크린샷 저장 경로**: 스토리북 루트의 `.screenshots/` 디렉토리
**파일명 규칙**: `{story-kebab-name}-{light|dark}.png`

### Step 6: Git 스테이징

```bash
git add {screenshots-dir}/*.png
```

### Step 7: PR body 업데이트 (선택)

PR이 존재하면 `gh pr edit`으로 스크린샷 테이블을 body에 추가/업데이트합니다.

기존 body에 `## 📸 UI 스크린샷` 섹션이 있으면 해당 섹션만 교체합니다.
없으면 `## Test plan` 앞에 삽입합니다.

스크린샷 테이블 형식:

```markdown
## 📸 UI 스크린샷

### {컴포넌트/스토리 그룹명}
| 상태 | Light | Dark |
|------|-------|------|
| 기본 | ![light]({raw-url}-light.png) | ![dark]({raw-url}-dark.png) |
| 입장완료 | ![light]({raw-url}-light.png) | ![dark]({raw-url}-dark.png) |
```

이미지 URL 형식: `https://github.com/{owner}/{repo}/blob/{branch}/{path}?raw=true`

PR이 없으면 스크린샷 파일만 stage하고 사용자에게 알립니다.

## 출력

캡처 완료 후 아래를 출력합니다:
- 캡처된 스토리 수
- 생성된 스크린샷 파일 목록 (light/dark)
- PR 업데이트 여부

## 오류 처리

| 상황 | 처리 |
|------|------|
| Playwright 미설치 | `[ERROR] Playwright가 설치되어 있지 않습니다. npx playwright install chromium을 실행해주세요.` |
| 스토리북 빌드 실패 | 에러 로그 출력 후 종료 |
| 변경된 스토리 없음 | `변경된 stories 파일이 없습니다.` 출력 후 종료 |
| 포트 6007 사용 중 | 기존 프로세스 종료 후 재시도 |
| 스크린샷 캡처 실패 | 해당 스토리 스킵 + 사유 로그, 나머지 계속 |

## 참고

- 스토리북의 iframe URL에 `globals={key}:{value}` 파라미터를 추가하면 테마를 전환할 수 있습니다
- DaisyUI 프로젝트는 보통 `data-theme` 속성으로 테마를 전환합니다
- 스크린샷은 Playwright의 headless Chromium으로 캡처됩니다
