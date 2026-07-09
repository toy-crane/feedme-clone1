# url-to-markdown 구현 계획

## 아키텍처 결정

| 결정 | 선택 | 이유 |
|---|---|---|
| 본문 추출 파이프라인 | Route Handler(`app/api/convert/route.ts`, Node.js 런타임)가 대상 URL을 서버에서 `fetch` → `linkedom`으로 DOM 생성 → `defuddle/node`(`{ markdown: true }`)로 파싱 | defuddle는 URL을 직접 fetch하지 않고 이미 만들어진 `Document`를 받는다(GitHub README 확인). linkedom은 네이티브 의존성이 없어 jsdom보다 가볍고 Node 런타임 서버리스 환경에 적합 |
| Markdown 렌더링 | `react-markdown` + `remark-gfm` | 결과 화면에서 defuddle가 반환한 markdown 문자열을 그대로 렌더링. React 생태계 표준, GFM(표 등) 지원. 스위칭 비용 낮아 초안에서 확정 |
| 복사/안내 피드백 | shadcn `sonner`(Toast) 추가 | "복사됨", "클립보드에 복사됨, 붙여넣으세요" 같은 일시적 안내에 적합, 기존 shadcn 스타일과 통일 |
| 프리셋/직접입력 선택 UI | shadcn `radio-group` 추가 | wireframe의 라디오 형태와 일치, 접근성 기본 제공 |
| 다크모드 | 기존 `app/layout.tsx`의 `ThemeProvider` 설정을 `defaultTheme="system"`, `enableSystem={true}`로 변경 | 시나리오 11(시스템 설정 기본 적용) 충족. 현재 코드는 `defaultTheme="light"`, `enableSystem={false}`로 고정되어 있어 수정 필요 |
| 상태 관리 | 최상위 클라이언트 컴포넌트의 로컬 React state (url, 결과, 에러, 선택된 프롬프트, 직접입력 문구) | 게스트 전용/무상태 서비스(범위에서 로그인·이력 저장 제외 확정). 새로고침 시 초기화되는 것이 오히려 "직접 입력 프롬프트 미저장" 불변 규칙과 자연히 일치 |
| 에러 처리 | API가 실패 원인(잘못된 형식/접근 불가/추출 실패)을 구분하지 않고 단일 공통 에러 메시지를 반환 | 불변 규칙: 원인 구분 없이 동일한 에러 메시지 노출 |
| 홈 페이지 교체 | `app/page.tsx`(현재 shadcn 컴포넌트 데모)를 이 feature의 UI로 교체 | 별도 라우트를 만들 근거가 spec에 없고, 현재 `app/page.tsx`는 임시 데모 스캐폴딩(`ComponentExample`)이라 대체 대상으로 적합 |

## 인프라 리소스

없음 — 게스트 전용, 영속 저장소 없음. `app/api/convert/route.ts`는 `export const runtime = "nodejs"`로 명시(linkedom이 Edge 런타임에서 미검증이므로 Node.js 런타임 고정).

## 데이터 모델

영속 엔티티 없음(로그인/이력 저장 제외 범위 확정). 클라이언트 로컬 상태로만 존재하는 타입:

### ConversionResult (`types/conversion.ts`)
- title (required)
- author (optional — 없으면 필드 자체를 렌더링하지 않음, Scenario 2)
- markdown (required — 순수 Markdown, 프롬프트 미포함)

## 필요 스킬

| 스킬 | 적용 Task | 용도 |
|---|---|---|
| shadcn | Task 3, 4 | `sonner`(Task 3), `radio-group`(Task 4) 컴포넌트를 레지스트리에서 추가(`components/ui/*` 직접 수정 금지, variant/semantic token 우선 규칙 준수) |
| next-best-practices | Task 1 | Route Handler 작성 규약, async API, 캐싱 기본값 확인 |
| vercel-react-best-practices | Task 2, 3, 4 | 클라이언트 컴포넌트 경계 최소화, 불필요 리렌더 방지 |
| web-design-guidelines | Task 5 | 다크모드 토글의 대비·접근성 점검 |

## 영향 받는 파일

| 파일 경로 | 변경 유형 | 관련 Task |
|---|---|---|
| `app/api/convert/route.ts` | New | Task 1 |
| `lib/extract-content.ts` | New | Task 1 |
| `types/conversion.ts` | New | Task 1 |
| `components/url-to-markdown/url-to-markdown-app.tsx` | New | Task 2 |
| `components/url-to-markdown/conversion-result.tsx` | New | Task 2 |
| `app/page.tsx` | Modify | Task 2 |
| `app/layout.tsx` | Modify | Task 2 (메타데이터), Task 5 (ThemeProvider 설정) |
| `e2e/smoke.spec.ts` | Modify | Task 2 |
| `lib/download-markdown-file.ts` | New | Task 3 |
| `components/ui/sonner.tsx` | New (shadcn add) | Task 3 |
| `components/url-to-markdown/export-actions.tsx` | New | Task 3 |
| `components/ui/radio-group.tsx` | New (shadcn add) | Task 4 |
| `lib/build-open-in-text.ts` | New | Task 4 |
| `components/url-to-markdown/prompt-selector.tsx` | New | Task 4 |
| `components/mode-toggle.tsx` | New | Task 5 |

변경 유형: New \| Modify \| Delete

## Tasks

### Task 1: URL을 서버에서 Markdown으로 추출하는 API

- **담당 시나리오**: Scenario 1 (partial — API 계약만, UI 제외), Scenario 2 (partial — 저자 없음 응답 형태), Scenario 3 (partial — API가 공통 에러를 반환하는지)
- **크기**: M (3-5 파일)
- **의존성**: None
- **참조**:
  - defuddle GitHub README(`kepano/defuddle`) — `defuddle/node` + `linkedom` 사용 패턴: `Defuddle(document, url, { markdown: true })`
  - next-best-practices — Route Handler 작성 규약
- **구현 대상**:
  - `lib/extract-content.ts` — URL을 받아 fetch → linkedom → defuddle 실행, `ConversionResult` 또는 실패 시 `null` 반환
  - `lib/extract-content.test.ts`
  - `app/api/convert/route.ts` — POST, `{ url }` → `{ result }` 또는 `{ error }` (HTTP 4xx/5xx 구분 없이 동일한 에러 메시지 문자열 사용)
  - `app/api/convert/route.test.ts`
  - `types/conversion.ts`
- **수용 기준**:
  - [ ] 저자 정보가 있는 실제 기사 URL로 POST 요청 → 응답 JSON에 title, author, markdown이 모두 채워짐
  - [ ] 저자 메타데이터가 없는 페이지 URL로 POST 요청 → 응답 JSON에 author 필드가 없거나 빈 값(falsy)
  - [ ] 형식이 잘못된 문자열을 URL로 POST 요청 → 공통 에러 메시지를 담은 응답, 200대 결과 필드 없음
  - [ ] 접근 불가능한 URL(예: fetch가 404/네트워크 에러를 던지는 목 서버)로 POST 요청 → 위와 동일한 공통 에러 메시지
  - [ ] 본문 추출이 불가능한 HTML(예: 빈 body)로 POST 요청 → 위와 동일한 공통 에러 메시지 (원인별로 메시지가 달라지지 않음을 3개 케이스 비교로 검증)
- **검증**: `bun run test -- extract-content route` (fetch는 vi.stubGlobal 또는 msw로 목킹, 실제 네트워크 호출 없이 결정적으로 검증)

---

### Task 2: 입력 화면에서 변환하고 결과 화면에서 확인한다

- **담당 시나리오**: Scenario 1 (full), Scenario 2 (full), Scenario 3 (full), Scenario 4 (full)
- **크기**: M (신규 3 파일 + 보조 수정 3건 — page.tsx/layout.tsx/e2e smoke는 한 줄 수준의 배선·기대값 변경)
- **의존성**: Task 1 (API 계약을 소비)
- **참조**:
  - wireframe.html screen-0(입력 화면), screen-1(결과 화면 좌측 본문 영역)
  - shadcn — 기존 `components/ui/input.tsx`, `components/ui/button.tsx` 재사용(직접 수정 금지)
  - `react-markdown`, `remark-gfm` — npm 패키지 설치 필요
- **구현 대상**:
  - `components/url-to-markdown/url-to-markdown-app.tsx` — URL 입력, "변환" 클릭 시 Task1 API 호출, 로딩/에러/결과 상태 관리, "지우기" 시 전체 초기화
  - `components/url-to-markdown/url-to-markdown-app.test.tsx`
  - `components/url-to-markdown/conversion-result.tsx` — 제목/저자(있을 때만)/react-markdown 렌더링
  - `app/page.tsx` — `<UrlToMarkdownApp />` 렌더링으로 교체
  - `app/layout.tsx` — metadata title/description을 "URL to Markdown"으로 변경
  - `e2e/smoke.spec.ts` — 변경된 title에 맞춰 기대값 수정
- **수용 기준**:
  - [ ] 빈 입력 화면에서 유효한 기사 URL을 붙여넣고 "변환" 클릭 → 결과 화면으로 전환되며 제목 헤더와 렌더링된 Markdown 본문이 보임
  - [ ] 저자 정보가 있는 페이지 변환 → 결과 헤더에 저자명이 표시됨
  - [ ] 저자 정보 없는 페이지 변환 → 결과 헤더에 저자 필드 자체가 렌더링되지 않음
  - [ ] 형식이 잘못된 문자열 입력 후 "변환" 클릭 → 공통 에러 메시지가 표시되고 결과 화면으로 전환되지 않음
  - [ ] 결과 화면이 표시된 상태에서 "지우기" 클릭 → URL 입력칸이 빈 값이 되고 결과 영역이 사라져 최초 화면으로 돌아감
- **검증**: `bun run test -- url-to-markdown-app`; Task 1 API는 `msw` 또는 `vi.fn` fetch 목으로 대체해 컴포넌트 단위에서 결정적으로 검증

---

### Checkpoint: Task 1-2 이후
- [ ] 모든 테스트 통과: `bun run test`
- [ ] 빌드 성공: `bun run build`
- [ ] 핵심 vertical slice(URL 붙여넣기 → 변환 → 결과 확인 → 지우기로 초기화)가 end-to-end로 동작

---

### Task 3: Markdown 복사하기 / .md 다운로드

- **담당 시나리오**: Scenario 5 (full), Scenario 6 (full)
- **크기**: M (신규 모듈 2 — sonner는 shadcn 레지스트리 산출물이라 직접 작성 분량에서 제외)
- **의존성**: Task 2 (결과 화면에 버튼을 부착)
- **참조**:
  - wireframe.html screen-1 우측 "내보내기" 패널
  - shadcn `sonner` — 복사/다운로드 완료 피드백에 사용(신규 추가)
- **구현 대상**:
  - `components/ui/sonner.tsx` — `bunx shadcn add sonner`로 추가(직접 작성 금지, 레지스트리 산출물 그대로 사용)
  - `lib/download-markdown-file.ts` — Blob + `<a download>`로 `.md` 파일 트리거
  - `lib/download-markdown-file.test.ts`
  - `components/url-to-markdown/export-actions.tsx` — 복사하기/다운로드 버튼, `navigator.clipboard.writeText`는 항상 순수 markdown만 사용(프롬프트 미포함)
  - `components/url-to-markdown/export-actions.test.tsx`
- **수용 기준**:
  - [ ] 프리셋 프롬프트가 선택된 상태에서 "복사하기" 클릭 → `navigator.clipboard.writeText`에 전달된 문자열이 프롬프트를 포함하지 않고 순수 Markdown과 일치, 복사 완료 토스트가 표시됨
  - [ ] 변환 결과 상태에서 ".md 파일 다운로드" 클릭 → 다운로드 트리거의 파일명이 `.md`로 끝나고 내용이 순수 Markdown과 일치
- **검증**: `bun run test -- export-actions download-markdown-file` (`navigator.clipboard`와 다운로드 앵커 클릭은 vi.fn으로 스파이/검증)

---

### Task 4: 프롬프트와 함께 ChatGPT·Claude로 열기

- **담당 시나리오**: Scenario 7 (full), Scenario 8 (full), Scenario 9 (full), Scenario 10 (full)
- **크기**: M (3-5 파일)
- **의존성**: Task 3 (내보내기 패널 옆에 배치, 순수 markdown 생성 로직 재사용)
- **참조**:
  - wireframe.html screen-1 우측 "프롬프트와 함께 열기" 패널
  - shadcn `radio-group` — 프리셋/직접입력 선택(신규 추가)
- **구현 대상**:
  - `lib/build-open-in-text.ts` — `(promptText: string | null, markdown: string) => string`, 프롬프트가 있으면 앞에 붙이고 없으면 markdown 그대로 반환
  - `lib/build-open-in-text.test.ts`
  - `components/url-to-markdown/prompt-selector.tsx` — "없음"/프리셋 3종/"직접 입력" radio-group, 직접 입력 선택 시에만 텍스트 인풋 노출, 선택 상태를 부모에 전달(값 자체는 세션 로컬 state에만 존재, 저장소 미사용)
  - `components/url-to-markdown/prompt-selector.test.tsx`
  - `components/url-to-markdown/url-to-markdown-app.tsx` (Modify) — "ChatGPT로 열기"/"Claude로 열기" 버튼이 `build-open-in-text` 결과를 클립보드에 복사하고 `window.open`으로 각각 chatgpt.com/claude.ai를 새 탭으로 열도록 연결
- **수용 기준**:
  - [ ] 프롬프트 미선택 상태에서 "ChatGPT로 열기" 클릭 → `window.open`이 chatgpt.com으로 호출되고 클립보드에는 Markdown 본문만 담김, 안내 토스트가 표시됨
  - [ ] 동일 조건에서 "Claude로 열기" 클릭 → `window.open`이 claude.ai로 호출되고 클립보드는 Markdown 본문만 포함
  - [ ] "요약해줘" 프리셋 선택 후 "ChatGPT로 열기" 클릭 → 클립보드 내용이 "요약해줘"로 시작하고 이어서 Markdown 본문이 포함됨
  - [ ] "직접 입력" 선택 후 임의 문구 입력, "ChatGPT로 열기" 클릭 → 클립보드가 해당 문구로 시작
  - [ ] 직접 입력 필드에 문구를 입력한 뒤 새 변환을 실행(입력 화면으로 돌아갔다가 다시 결과 화면 진입) → 직접 입력 필드가 항상 빈 값으로 시작함(이전 문구가 남아있지 않음, 컴포넌트 state나 storage 어디에도 값이 유지되지 않음)
  - [ ] 직접 입력 상태에서 다른 프리셋(예: "요약해줘")을 다시 선택 → 직접 입력 텍스트 인풋이 사라지고 이후 "열기" 동작에는 프리셋 문구가 적용됨
- **검증**: `bun run test -- prompt-selector build-open-in-text url-to-markdown-app`; `window.open`/`navigator.clipboard.writeText`는 vi.fn으로 스파이

---

### Checkpoint: Task 3-4 이후
- [ ] 모든 테스트 통과: `bun run test`
- [ ] 빌드 성공: `bun run build`
- [ ] 내보내기(복사/다운로드/AI로 열기) vertical slice가 end-to-end로 동작하며, 프롬프트가 복사/다운로드 결과물에는 절대 섞이지 않음(불변 규칙 재확인)

---

### Task 5: 다크모드 토글

- **담당 시나리오**: Scenario 11 (full)
- **크기**: S (1-2 파일 + 설정 수정)
- **의존성**: Task 2 (헤더 영역에 토글 버튼을 부착)
- **참조**:
  - wireframe.html 각 화면 헤더의 moon 아이콘 버튼 위치
  - `next-themes` 공식 문서 — `useTheme()` 훅
- **구현 대상**:
  - `app/layout.tsx` (Modify) — `ThemeProvider`를 `defaultTheme="system"`, `enableSystem={true}`로 변경
  - `components/mode-toggle.tsx` — `useTheme()`으로 라이트/다크 전환하는 아이콘 버튼
  - `components/mode-toggle.test.tsx`
  - `components/url-to-markdown/url-to-markdown-app.tsx` (Modify) — 헤더에 `<ModeToggle />` 배치
- **수용 기준**:
  - [ ] 시스템이 다크 모드로 설정된 환경(`prefers-color-scheme: dark` 목킹)에서 첫 로드 → 다크 테마 클래스가 적용된 상태로 렌더링됨
  - [ ] 다크모드 토글 버튼 클릭 → 라이트/다크 테마 클래스가 즉시 전환됨
- **검증**: `bun run test -- mode-toggle`(`matchMedia` 목으로 시스템 다크모드 시뮬레이션); Human review — 실제 브라우저에서 시스템 다크모드 전환 시 즉시 반영되는 느낌은 자동화 한계가 있어 리뷰어가 `bun run dev`로 육안 확인, 증거는 `artifacts/url-to-markdown/evidence/task-5-dark-mode.png`

---

### Checkpoint: Task 5 이후 (전체 완료)
- [ ] 모든 테스트 통과: `bun run test`
- [ ] E2E 스모크 통과: `bun run test:e2e`
- [ ] 빌드 성공: `bun run build`
- [ ] spec.md의 11개 시나리오 전부가 실제 브라우저에서 end-to-end로 동작

## 미결정 항목

없음 — 모든 항목을 초안에서 확정(스위칭 비용이 낮은 결정 위주). 리뷰 중 이견이 있으면 이 섹션에 기록한다.
