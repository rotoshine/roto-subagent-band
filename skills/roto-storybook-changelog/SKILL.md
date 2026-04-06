---
name: roto-storybook-changelog
description: 현재 브랜치의 UI 변경사항을 분석하여 Storybook 패치노트(MDX + stories)를 자동 생성합니다. "패치노트", "changelog", "스토리북 변경 기록", "UI 변경 요약" 등을 요청하면 이 스킬을 사용하세요.
user-invocable: true
---

# Storybook Changelog 생성

현재 브랜치의 UI 변경사항을 분석하여 Storybook 패치노트(MDX + stories 파일)를 생성합니다.

## 사용 방법

```text
/roto-storybook-changelog
```

## 파일명 규칙

- **날짜**: `YYYY-MM-DD` (오늘 날짜)
- **branch-slug**: 브랜치명에서 `/` → `-`, `_` → `-`로 치환
- **경로**: 프로젝트의 스토리북 `stories/changelog/` 디렉토리 (자동 탐지)

## 워크플로우

### Step 1: 스토리북 구조 탐지

프로젝트의 스토리북 설정을 자동 탐지합니다:

1. `.storybook/main.ts` 위치로 스토리북 루트 판별
2. `stories` 디렉토리 경로 확인
3. changelog 디렉토리 존재 여부 확인 (없으면 생성)

### Step 2: 브랜치 변경사항 분석

```bash
git fetch origin
git diff origin/main...HEAD --name-status
```

분석 대상:
- 새로 추가된 `.stories.tsx` 파일
- 수정된 컴포넌트 파일 (`*.tsx`)
- i18n 변경 (`messages/*.json`)
- CSS 변경 (`*.css`)
- 삭제된 파일

### Step 3: 변경 유형 분류

| 카테고리 | 설명 |
|----------|------|
| 새 컴포넌트 | 새로 추가된 컴포넌트 + 스토리 |
| UI 변경 | 기존 컴포넌트의 시각적 변경 |
| 리팩터링 | 컴포넌트 분리, 이동 등 구조 변경 |
| 삭제 | 제거된 컴포넌트 |

### Step 4: Stories 파일 생성

새로 추가/변경된 stories의 스토리를 re-export하는 전용 stories 파일을 생성합니다.

**IMPORTANT**: Storybook 10의 MDX 제약으로 인해 반드시 아래 패턴을 따를 것:
- `<Meta of={...}>` 사용 (`<Meta title="...">` 사용 금지)
- 외부 stories의 스토리를 보여주려면 전용 stories 파일에서 re-export

```tsx
import type { Meta, StoryObj } from "@storybook/react";
import ComponentA from "path/to/ComponentA";
import { Default as CompADefault } from "path/to/ComponentA.stories";

const meta: Meta = {
  title: "Changelog/YYYY-MM-DD 변경 설명",
};
export default meta;

export const ComponentA_Default: StoryObj = {
  ...CompADefault,
  name: "ComponentA — 기본",
  render: (args) => <ComponentA {...(CompADefault.args as any)} {...args} />,
};
```

**주의**: 원본 stories 파일의 export 이름을 반드시 Grep으로 확인 후 import.

### Step 5: MDX 패치노트 생성

```mdx
import { Meta, Canvas } from "@storybook/addon-docs/blocks";
import * as ChangelogStories from "./{filename}.stories";

<Meta of={ChangelogStories} />

# {브랜치 설명} — {YYYY.MM.DD}

## 요약
{변경사항 1~3줄 요약}

## 변경 내용
### 새 컴포넌트
#### ComponentName
{설명}
<Canvas of={ChangelogStories.StoryName} meta={ChangelogStories} />

### UI 변경
{기존 → 신규 변경사항 설명}
```

### Step 6: 검증

- stories 파일의 import 경로가 유효한지 확인
- export 이름이 원본과 일치하는지 확인
- MDX에서 참조하는 스토리가 stories 파일에 모두 존재하는지 확인

### Step 7: 변경 파일 스테이징

```bash
git add {changelog-dir}/*.stories.tsx {changelog-dir}/*.mdx
```

## 스토리가 없는 경우

변경된 stories 파일이 없으면 빈 stories 파일을 생성하고 `<Meta of={ChangelogStories} />`를 사용합니다.

## 참고

- Storybook 10 MDX 제약: unattached MDX에서 `<Canvas of={...}>`를 사용하면 `of={undefined}` 에러 발생
- 반드시 전용 stories 파일을 만들고 `<Meta of={...}>`로 첨부할 것
