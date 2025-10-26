# VibeTunnel CSS 디자인 구조 및 인터렉션 플로우

> VibeTunnel의 CSS 아키텍처, UI 인터렉션 시그널 플로우, 반응형 디자인 전략에 대한 상세 문서

## 📐 CSS 아키텍처

### 1.1 조직 구조

**CSS 방법론**: **Tailwind CSS v4 + 커스텀 CSS 변수** 및 **LitElement를 통한 CSS-in-JS 패턴**

**주요 파일**:
- `web/src/client/styles.css` - 메인 스타일시트 (2,150+ 줄)
- 컴포넌트는 `html` 템플릿 리터럴을 통해 인라인 Tailwind 클래스 사용
- Shadow DOM을 비활성화하여 Tailwind와 호환

**파일 위치**: `web/src/client/styles.css`

### 1.2 디자인 시스템 레이어

#### CSS 변수 (Custom Properties)

**위치**: `styles.css:5-183`

카테고리별로 구성된 디자인 토큰:

```css
:root {
  /* 색상 팔레트 */
  --color-bg                    /* 배경색 */
  --color-bg-secondary          /* 보조 배경색 */
  --color-bg-tertiary           /* 삼차 배경색 */
  --color-bg-elevated           /* 강조 배경색 */
  --color-surface               /* 표면색 */
  --color-surface-hover         /* 표면 호버색 */
  --color-border                /* 테두리색 */
  --color-border-light          /* 연한 테두리색 */
  --color-border-focus          /* 포커스 테두리색 */
  --color-text                  /* 텍스트색 */
  --color-text-bright           /* 밝은 텍스트색 */
  --color-text-muted            /* 흐린 텍스트색 */
  --color-text-dim              /* 어두운 텍스트색 */
  --color-primary               /* 기본 강조색 */
  --color-primary-hover         /* 강조색 호버 */
  --color-primary-dark          /* 어두운 강조색 */
  --color-primary-light         /* 밝은 강조색 */
  --color-status-error          /* 에러 상태색 */
  --color-status-warning        /* 경고 상태색 */
  --color-status-success        /* 성공 상태색 */
  --color-status-info           /* 정보 상태색 */

  /* 그림자 */
  --shadow-glow                 /* 발광 효과 */
  --shadow-glow-sm              /* 작은 발광 */
  --shadow-glow-lg              /* 큰 발광 */
  --shadow-glow-intense         /* 강한 발광 */
  --shadow-card                 /* 카드 그림자 */
  --shadow-card-hover           /* 카드 호버 그림자 */
  --shadow-elevated             /* 강조 그림자 */

  /* 레이아웃 상수 */
  --vt-breakpoint-mobile: 768px
  --vt-breakpoint-tablet: 1024px
  --vt-breakpoint-desktop: 1280px
  --vt-sidebar-default-width: 420px
  --vt-sidebar-min-width: 240px
  --vt-sidebar-max-width: 600px

  /* Z-Index 레이어 */
  --vt-z-mobile-overlay: 20
  --vt-z-sidebar-mobile: 30
  --vt-z-session-exited-overlay: 25

  /* 전환 효과 */
  --vt-transition-sidebar: 200ms
  --vt-transition-mobile-slide: 200ms
}
```

#### 테마 시스템

**라이트 테마**: `styles.css:6-30` (RGB 값)
**다크 테마**: `styles.css:167-183` (`[data-theme="dark"]` 선택자)
**시스템 설정**: `styles.css:186-202` (`@media (prefers-color-scheme: dark)`)

```css
/* 다크 테마 예시 */
[data-theme="dark"] {
  --color-bg: rgb(23 23 23);
  --color-text: rgb(228 228 228);
  --color-primary: rgb(96 165 250);
}

/* 시스템 테마 우선순위 */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    /* 다크 테마와 동일한 값 */
  }
}
```

### 1.3 Tailwind 통합

**Tailwind v4 테마 설정** (`styles.css:32-83`):

- CSS 변수를 사용한 커스텀 색상 등록
- 커스텀 그림자 및 애니메이션 정의
- 애니메이션 키프레임: `pulsePrimary`, `slideInRight`, `slideInBottom`, `fadeIn`, `scaleIn`

**컴포넌트 클래스** (`styles.css:356-683`):

```css
.terminal-icon          /* 발광 터미널 아이콘 효과 */
.input-field           /* 통일된 입력 필드 스타일 */
.btn, .btn-sm, .btn-md, .btn-lg  /* 버튼 크기 변형 */
.btn-primary, .btn-secondary, .btn-ghost  /* 버튼 스타일 변형 */
.card, .card-elevated  /* 카드 스타일링 (호버 효과 포함) */
.status-badge          /* 상태 배지 컴포넌트 */
.quick-start-btn       /* 빠른 시작 버튼 */
.modal-backdrop, .modal-content  /* 모달 스타일링 */
.session-flex-responsive  /* 세션 그리드 레이아웃 */
```

### 1.4 글로벌 스타일 레이어

**베이스 레이어** (`styles.css:205-353`):

```css
html, body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100vh;
  height: 100dvh;  /* 동적 뷰포트 높이 */
}
```

- HTML/body 리셋
- 폼 요소: checkbox, select, textarea, radio, file input 스타일링
- iOS 노치 및 내비게이션 바를 위한 Safe Area 지원

**컴포넌트 레이어** (`styles.css:356-683`):
- 재사용 가능한 UI 컴포넌트
- 모든 버튼 변형 및 상태
- 카드 및 모달 컴포넌트

**유틸리티 레이어** (`styles.css:696-806`):

```css
.scrollbar-hide         /* 크로스 브라우저 스크롤바 숨기기 */
.interactive           /* 전환 효과 유틸리티 */
.interactive-fast      /* 빠른 전환 */
.interactive-slow      /* 느린 전환 */
.hover-lift           /* 호버 리프트 효과 */
.hover-glow           /* 호버 발광 효과 */
.pulse-slow           /* 느린 펄스 애니메이션 */
.slide-in-from-right  /* 오른쪽에서 슬라이드 인 */
.slide-in-from-bottom /* 아래에서 슬라이드 인 */
```

### 1.5 폰트 로딩

**Fira Code Variable Font** (`styles.css:809-818`):

```css
@font-face {
  font-family: 'Fira Code';
  font-weight: 300 700;
  font-display: swap;
  src: url('/fonts/FiraCode-VF.woff2') format('woff2-variations');
}
```

**Hack Nerd Font Mono** (`styles.css:820-835`):
- Regular 및 Bold 웨이트를 별도로 로드
- 터미널 렌더링에 사용 (Fira Code보다 우선)

**폰트 스택** (`styles.css:838-842`):

```css
.font-mono {
  font-family: 'Hack Nerd Font Mono', 'Fira Code', ui-monospace,
               'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono',
               Consolas, 'Courier New', monospace;
}
```

## 🔄 UI 인터렉션 시그널 플로우

### 2.1 LitElement 컴포넌트 아키텍처

**앱 진입점**: `web/src/client/app.ts:57-1920`

**LitElement 설정**:
- 모든 컴포넌트가 Shadow DOM 비활성화: `createRenderRoot() { return this; }`
- 이유: Tailwind CSS가 전역 범위를 필요로 함 (Shadow DOM은 스타일을 격리)

**반응형 상태 관리**:

```typescript
@state() private errorMessage = '';
@state() private successMessage = '';
@state() private sessions: Session[] = [];
@state() private currentView: 'list' | 'session' | 'auth' | 'file-browser' = 'auth';
@state() private mediaState: MediaQueryState = responsiveObserver.getCurrentState();
@state() private sidebarCollapsed = this.loadSidebarState();
```

### 2.2 사용자 인터렉션 → 이벤트 → 상태 → 렌더 플로우

#### 예시: 세션 네비게이션

```
사용자가 세션 카드 클릭
    ↓
session-card.ts:119-126: handleCardClick()
    ↓
CustomEvent('session-select') 디스패치 → bubbles: true
    ↓
app.ts:1814: @session-select 리스너
    ↓
app.ts:1005-1050: handleNavigateToSession(e: CustomEvent)
    ↓
상태 업데이트:
  - this.selectedSessionId = sessionId
  - this.currentView = 'session'
  - this.sidebarCollapsed = true (모바일에서)
    ↓
LitElement 재렌더링 (app.ts:1664-1919)
    ↓
Tailwind 클래스가 뷰를 동적으로 업데이트
```

### 2.3 이벤트 위임 계층

**전역 키보드 단축키** (`app.ts:188-402`):

애플리케이션 레벨에서 캡처 (`window.addEventListener('keydown', handleKeyDown)`):

**캡처된 단축키**:
- `Cmd/Ctrl+1-9`: 세션 전환
- `Cmd/Ctrl+B`: 사이드바 토글
- `Cmd/Ctrl+O`: 파일 브라우저 열기
- `Escape`: 세션 뷰 종료
- 터미널 컨트롤 키: `Cmd/Ctrl+A`, `Cmd/Ctrl+E`, `Cmd/Ctrl+W` 등

**이벤트 버블링 흐름**:

```
컴포넌트 이벤트 (click, input, change)
    ↓
bubbles: true → 부모 컴포넌트가 캡처
    ↓
컴포넌트 이벤트 핸들러 (예: session-select)
    ↓
setState() 트리거
    ↓
app.ts:1137-1151 willUpdate() → 상태 변경 처리
    ↓
app.ts:1664+ render() → DOM 업데이트
    ↓
LitElement가 새로운 Tailwind 클래스로 업데이트
```

### 2.4 터미널 입력 시그널 플로우

#### 데스크톱 입력

**위치**: `web/src/client/components/session-view/input-manager.ts:1-150`

```
사용자가 터미널에 입력
    ↓
keydown 이벤트 캡처
    ↓
입력 라우팅 로직:
  - 브라우저 단축키: 허용 (Ctrl+Tab, Ctrl+W 등)
  - 터미널 캡처: Ctrl+A/E/L/R/P/U/K, Alt+D (셸 편집용)
  - 기타 키: WebSocket 또는 HTTP로 전송
    ↓
서버로 WebSocket 연결: /api/sessions/{id}/input
    ↓
서버가 PTY 쓰기 처리
```

#### 모바일 입력

**위치**: `web/src/client/components/session-view/mobile-input-manager.ts`

```
사용자가 키보드 버튼 터치
    ↓
handleMobileInputToggle() / handleMobileInputSend()
    ↓
showMobileInput 상태 토글
    ↓
mobile-input-overlay가 textarea와 함께 렌더링
    ↓
사용자가 입력하고 전송 버튼 누름
    ↓
inputManager.sendInputText(text)
    ↓
서버로 WebSocket 메시지
```

#### IME (한중일) 입력

**위치**: `web/src/client/components/session-view/input-manager.ts:141-149`

- `navigator.language`를 통해 CJK 언어 감지
- `DesktopIMEInput` 컴포넌트 설정
- `compositionstart`, `compositionupdate`, `compositionend` 이벤트 캡처

### 2.5 터미널 출력 시그널 플로우

#### 데이터 플로우: 서버 → WebSocket → 터미널 렌더

```
서버가 PTY에 쓰기
    ↓
web/src/server/services/buffer-aggregator.ts가 데이터 그룹화
    ↓
WebSocket 메시지 (바이너리 0xBF magic byte)
    ↓
bufferSubscriptionService.ts가 메시지 수신
    ↓
vibe-terminal-binary.ts: writeBinary(data)
    ↓
XtermTerminal.write(data)
    ↓
XTerm이 DOM에 렌더링
    ↓
terminal-renderer.ts가 렌더 업데이트 큐에 추가
    ↓
renderBuffer()가 requestAnimationFrame 중에 실행
```

#### 활동 모니터링

**위치**: `web/src/client/components/session-card.ts:129-148`

```
터미널 콘텐츠 변경
    ↓
content-changed 이벤트 발생
    ↓
session-card.ts: handleContentChanged()
    ↓
isActive = true
    ↓
활동 표시기 렌더링 (시각적 피드백)
    ↓
500ms 비활성 후 지우기
```

### 2.6 뷰 전환

#### View Transition API

**위치**: `web/src/client/app.ts:767-815`

```typescript
if ('startViewTransition' in document) {
  document.startViewTransition(async () => {
    await performLoad();
    await this.updateComplete; // Lit 재렌더링 대기
  });
}
```

#### CSS 애니메이션

**위치**: `web/src/client/styles.css:1820-1941`

- 세션 로딩: `initialLoad` 페이드인 애니메이션
- 세션 숨기기: `sessionHide` 스케일아웃 애니메이션
- 세션 표시: `sessionFlow` 스케일인 애니메이션 (단계적 지연 포함)

## 📱 반응형 디자인

### 3.1 뷰포트 설정

**HTML 뷰포트 메타 태그** (`web/assets/index.html:4-8`):

```html
<meta name="viewport"
      content="width=device-width,
               initial-scale=1.0,
               viewport-fit=cover,
               user-scalable=no,
               interactive-widget=resizes-content" />
```

**주요 파라미터**:
- `viewport-fit=cover`: 노치를 포함한 전체 화면 사용 (iOS 11+)
- `user-scalable=no`: 핀치 줌 비활성화 (터미널 인터렉션 보호)
- `interactive-widget=resizes-content`: 가상 키보드에 맞춰 레이아웃 조정

### 3.2 중단점 및 미디어 쿼리

#### 3단계 중단점 시스템

**위치**: `web/src/client/utils/constants.ts:3-7`

```typescript
BREAKPOINTS = {
  MOBILE: 768,      // < 768px: 폰
  TABLET: 1024,     // 768-1023px: 태블릿
  DESKTOP: 1280,    // >= 1280px: 데스크톱
}
```

#### 반응형 옵저버

**위치**: `web/src/client/utils/responsive-utils.ts:1-105`

- `document.documentElement`에 `ResizeObserver` 사용
- 효율적인 뷰포트 추적 (여러 미디어 쿼리 대신 단일 옵저버)
- 중단점 변경 시 콜백 트리거
- `ResizeObserver` 미지원 시 `window.resize`로 폴백

**상태 결정**:

```typescript
getMediaQueryState(): MediaQueryState {
  const width = window.innerWidth;
  return {
    isMobile: width < 768,
    isTablet: width >= 768 && width < 1280,
    isDesktop: width >= 1280,
  };
}
```

### 3.3 모바일 vs 데스크톱 레이아웃

#### 세션 리스트 뷰

**위치**: `web/src/client/styles.css:537-615`

**데스크톱** (768px+):

```css
.session-flex-responsive {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  grid-auto-rows: 400px;
  gap: 1.5rem;
}
```
- 다중 열 그리드 레이아웃
- 280px 최소 카드 너비
- 400px 고정 행 높이

**태블릿** (540-768px):

```css
@media (max-width: 768px) {
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  grid-auto-rows: 380px;
  gap: 1rem;
}
```
- 약간 작은 카드
- 간격 감소

**모바일** (< 540px):

```css
@media (max-width: 540px) {
  grid-template-columns: minmax(0, 1fr);  /* 단일 열 */
  grid-auto-rows: auto;
  gap: 0.875rem;
}
```
- 단일 열 레이아웃
- 전체 뷰포트 너비
- 종횡비 4:3

**초소형** (< 380px):

```css
@media (max-width: 380px) {
  aspect-ratio: 16/11;  /* 약간 넓게 */
  padding: 0 0.625rem;
  gap: 0.625rem;
}
```

### 3.4 분할 뷰 (데스크톱 + 모바일)

#### 사이드바 동작

**위치**: `web/src/client/app.ts:1594-1643`

```typescript
private get sidebarClasses(): string {
  if (!this.showSplitView) {
    return 'w-full min-h-screen flex flex-col';
  }

  // 분할 뷰 활성화
  const mobileClasses = isMobile ? 'absolute left-0 top-0 bottom-0 flex' : '';
  const transitionClass = this.sidebarAnimationReady && !isMobile ? 'sidebar-transition' : '';

  const collapsedClasses = this.sidebarCollapsed
    ? isMobile ? 'hidden' : 'flex'
    : isMobile ? 'expanded' : 'flex';
}

private get sidebarStyles(): string {
  if (this.sidebarCollapsed) return 'width: 0px;';

  if (isMobile) {
    // 오른쪽 80px 마진을 뺀 100vw (탭-투-클로즈용)
    return `width: calc(100vw - 80px); z-index: 30;`;
  }

  // 데스크톱: 커스터마이징 가능한 너비 (240-600px, 기본값 420px)
  return `width: ${this.sidebarWidth}px;`;
}
```

#### 모바일 오버레이

**위치**: `web/src/client/app.ts:1764-1779`

```html
${this.shouldShowMobileOverlay
  ? html`
    <div class="fixed inset-0 sm:hidden transition-all"
         style="z-index: 25; transition-duration: 200ms;"
         @click=${this.handleMobileOverlayClick}>
    </div>
  `
  : ''}
```
- 모바일에서 사이드바 확장 시 백드롭 표시
- 탭하여 사이드바 닫기
- `sm:hidden` Tailwind 클래스: 640px+에서 숨김

### 3.5 터미널 크기 조정

#### 동적 크기 조정

**위치**: `web/src/client/components/terminal.ts:34-45`

```typescript
@property({ type: Number }) cols = 80;
@property({ type: Number }) rows = 24;
@property({ type: Number }) fontSize = 14;
@property({ type: Boolean }) fitHorizontally = false;
@property({ type: Number }) maxCols = 0;
```

#### 리사이즈 로직

**위치**: `web/src/client/components/terminal.ts:80-87`

- `ResizeObserver`가 컨테이너 크기 감시
- 디바운스된 리사이즈: `RESIZE_DEBOUNCE: 100ms`
- 문자 너비와 컨테이너 크기를 기반으로 cols/rows 계산

**컨테이너 처리**:

```
ResizeObserver가 컨테이너 크기 변경 감지
    ↓
새 cols/rows 계산: cols = Math.floor(width / charWidth)
    ↓
새 크기로 terminal-resize 이벤트 발생
    ↓
서버가 /api/sessions/{id}/resize를 통해 크기 업데이트 수신
    ↓
PTY가 새 터미널 크기로 리사이즈
```

### 3.6 터치 전용 처리

#### 터치 액션 제약

**위치**: `web/src/client/styles.css:844-862`

```css
html, body {
  touch-action: pan-x pan-y;  /* 패닝 허용, 줌 비허용 */
  -webkit-overflow-scrolling: touch;  /* iOS에서 부드러운 스크롤 */
}

vibe-terminal {
  touch-action: none;  /* 터미널이 모든 터치 처리 */
}

.xterm {
  touch-action: pan-y;  /* 수직 스크롤만, 수평 비허용 */
  overscroll-behavior: none;
}
```

#### 모바일 입력 처리

**위치**: `web/src/client/styles.css:1985-2026`

```css
@media (max-width: 768px) {
  .terminal-container {
    position: relative;
    overflow: hidden;
  }

  .xterm-viewport {
    overflow-y: auto !important;
    -webkit-overflow-scrolling: touch;
  }

  /* 키보드 표시 시 레이아웃 조정 */
  session-view {
    height: 100dvh;  /* 동적 뷰포트 높이 */
    max-height: 100dvh;
  }
}
```

#### 키보드 가시성

`visualViewport` API를 사용하여 키보드 높이 감지:

```javascript
const viewport = window.visualViewport;
const keyboardHeight = window.innerHeight - viewport.height;
```

- 키보드 표시/숨김 시 터미널 스크롤 위치 조정
- 뷰포트 높이 감소 시 세션 뷰 패딩 조정

### 3.7 iOS 전용 조정

#### Safe Areas

**위치**: `web/src/client/styles.css:337-352`

```css
.safe-area-top { padding-top: env(safe-area-inset-top); }
.safe-area-bottom { padding-bottom: env(safe-area-inset-bottom); }
.safe-area-left { padding-left: env(safe-area-inset-left); }
.safe-area-right { padding-right: env(safe-area-inset-right); }
```

#### iOS Safari 수정

**위치**: `web/src/client/styles.css:882-903`

```css
@supports (-webkit-touch-callout: none) {
  /* iOS Safari 전용 */
  .h-screen {
    height: 100vh;
    height: -webkit-fill-available;  /* iOS 우회 방법 */
  }

  .ios-split-view {
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    overflow: hidden;
    -webkit-overflow-scrolling: auto;
  }

  .ios-split-view > * {
    -webkit-overflow-scrolling: touch;
  }
}
```

#### 주소 표시줄 숨김

**위치**: `web/assets/index.html:107-139`

```javascript
window.addEventListener('load', () => {
  setTimeout(() => {
    window.scrollTo(0, 1);  // 주소 표시줄 숨김
    setTimeout(() => window.scrollTo(0, 0), 10);
  }, 10);
});
```

## 🌐 브라우저 환경 적응

### 4.1 동적 뷰포트 높이 관리

#### 모바일 뷰포트 높이 문제

- 모바일 브라우저가 주소 표시줄을 동적으로 숨김/표시
- 키보드/주소 표시줄 표시 시 `window.innerHeight` 변경
- `100vh`가 실제 보이는 높이보다 클 수 있음

#### 해결책

**위치**: `web/assets/index.html:113-129`

```javascript
const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);

function setViewportHeight() {
  const vh = window.innerHeight * 0.01;
  document.documentElement.style.setProperty('--vh', `${vh}px`);
}

setViewportHeight();

if (!isMobile) {
  // 데스크톱: 리사이즈 시 업데이트
  window.addEventListener('resize', setViewportHeight);
  window.addEventListener('orientationchange', () => {
    setTimeout(setViewportHeight, 100);  // 브라우저가 안정될 때까지 지연
  });
}
```

**CSS 사용법**:

```css
html, body {
  height: 100vh;
  height: calc(var(--vh, 1vh) * 100);  /* 커스텀 --vh 값 */
  height: 100dvh;  /* 최신 표준 (동적 뷰포트 높이) */
}
```

### 4.2 윈도우 리사이즈 처리

#### ResizeObserver 패턴

**위치**: `web/src/client/utils/responsive-utils.ts:12-57`

```typescript
this.resizeObserver = new ResizeObserver(() => {
  const newState = this.getMediaQueryState();
  if (this.hasStateChanged(this.currentState, newState)) {
    this.currentState = newState;
    this.notifyCallbacks(newState);  // 앱 재렌더링 트리거
  }
});

this.resizeObserver.observe(document.documentElement);
```

#### 레거시 브라우저용 폴백

```typescript
window.addEventListener('resize', () => {
  // 100ms 타임아웃으로 디바운스
  clearTimeout(timeoutId);
  timeoutId = window.setTimeout(() => {
    const newState = this.getMediaQueryState();
    // 변경 확인...
  }, 100);
});
```

#### 터미널 전용 리사이즈

**위치**: `web/src/client/components/terminal.ts`

- 터미널 컨테이너에 별도 `ResizeObserver`
- `RESIZE_DEBOUNCE: 100ms`로 디바운스
- 새 cols/rows 계산
- 서버로 리사이즈 이벤트 전송

### 4.3 방향 전환 처리

#### 방향 감지

**위치**: `web/src/client/components/terminal.ts:86`, `session-view.ts` 참조

```typescript
// window.orientationchange 이벤트 리스너
// 세로 모드 감지: window.innerHeight > window.innerWidth
// isPortrait가 모바일에서 사이드바 닫기 동작 결정

private get isInSidebarDismissMode(): boolean {
  if (!this.mediaState.isMobile || !this.shouldShowMobileOverlay) return false;

  // 세로: 오버레이로 사이드바 닫기
  // 가로: 사이드바 유지
  const isPortrait = window.innerHeight > window.innerWidth;
  return isPortrait;
}
```

### 4.4 가상 키보드 감지

#### Visual Viewport API

**위치**: `session-view.ts` (참조됨)

```javascript
const viewport = window.visualViewport;
const keyboardHeight = window.innerHeight - viewport.height;

// 키보드 표시 시:
// - 터미널이 위로 스크롤하여 입력 유지
// - 뷰포트 높이 감소
// - 세션 뷰가 패딩 조정
```

**키보드 처리** (`web/src/client/styles.css:2006-2026`):

```css
.session-view-grid[data-keyboard-visible="true"] {
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}

/* 키보드 인셋 조정 */
.session-view-grid[data-keyboard-visible="true"] .terminal-container {
  padding-bottom: env(keyboard-inset-height, 0);
}
```

### 4.5 기기 기능 감지

#### 모바일 감지

**위치**: `web/src/client/utils/mobile-utils.ts:1-56`

```typescript
export function detectMobile(): boolean {
  return (
    /iPhone|iPad|iPod|Android/i.test(navigator.userAgent) ||
    (!!navigator.maxTouchPoints && navigator.maxTouchPoints > 1)
  );
}

export function isIOS(): boolean {
  return /iPad|iPhone|iPod/.test(navigator.userAgent);
}

export function isAndroid(): boolean {
  return /Android/i.test(navigator.userAgent);
}

export function getMobilePlatform(): 'ios' | 'android' | 'other' | 'desktop' {
  // 플랫폼별 처리
}
```

**사용 예**:
- **iOS**: `-webkit-overflow-scrolling: touch` 사용, 고무줄 스크롤 방지
- **Android**: 키보드 가시성을 다르게 처리
- **Desktop**: 사이드바 전환 활성화, 전체 기능 세트

### 4.6 테마 시스템 적응

#### 시스템 설정 감지

**위치**: `web/assets/index.html:78-101`

```javascript
(function() {
  const saved = localStorage.getItem('vibetunnel-theme');
  const theme = saved || 'system';

  if (theme === 'system') {
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    document.documentElement.setAttribute('data-theme', prefersDark ? 'dark' : 'light');
  } else {
    document.documentElement.setAttribute('data-theme', theme);
  }
})();
```

#### CSS 변수

**위치**: `web/src/client/styles.css:167-202`

```css
[data-theme="dark"] {
  --color-bg: rgb(23 23 23);
  --color-text: rgb(228 228 228);
  /* ... */
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    /* [data-theme="dark"]와 동일한 색상 */
  }
}
```

### 4.7 브라우저 기능 지원

#### View Transitions API

**위치**: `web/src/client/app.ts:768-790`, `styles.css:1820-1941`

```typescript
if ('startViewTransition' in document) {
  document.startViewTransition(async () => {
    await performLoad();
    await this.updateComplete;
  });
} else {
  // 폴백: CSS 애니메이션
  document.body.classList.add('initial-session-load');
}
```

#### ResizeObserver 폴백

**위치**: `web/src/client/utils/responsive-utils.ts:20-40`

```typescript
try {
  this.resizeObserver = new ResizeObserver(() => { /* ... */ });
  this.resizeObserver.observe(document.documentElement);
} catch (error) {
  // window resize 이벤트로 폴백
  this.setupFallbackResizeListener();
}
```

#### CSS Containment

**위치**: `web/src/client/styles.css:853-867`

```css
.session-view-grid {
  contain: layout style;  /* 성능 최적화 */
  will-change: contents;
}

.terminal-area {
  contain: strict;
  isolation: isolate;
}
```

### 4.8 당겨서 새로고침 방지

#### 모바일 제스처 처리

**위치**: `web/src/client/styles.css:844-879`

```css
body {
  overscroll-behavior-y: contain;  /* 당겨서 새로고침 방지 */
  touch-action: pan-x pan-y;
  -webkit-overflow-scrolling: touch;
}

vibe-terminal {
  touch-action: none;  /* 터미널이 모든 터치 캡처 */
  overscroll-behavior: none;
}

.session-card {
  overscroll-behavior: none;
}
```

## ⚡ 성능 최적화

### 5.1 렌더링 최적화

- **작업 큐**: 터미널이 `requestAnimationFrame`을 사용하여 업데이트 배치 처리
- **세션 캐싱**: 앱이 선택된 세션을 캐시하여 조회 방지
- **CSS Containment**: 터미널 및 오버레이에 `contain: layout style` 사용
- **Will-change**: 애니메이션 요소에 대한 브라우저 힌트

### 5.2 상태 지속성

- `localStorage`를 사용한 사이드바 상태, 테마, 설정 저장
- 페이지 로드 시 복원하여 세션 간 UX 유지
- 렌더링 전 테마 적용으로 깜박임 방지

### 5.3 세션 뷰에서 애니메이션 비활성화

**위치**: `web/src/client/styles.css`

```css
body.in-session-view .session-flex-responsive > session-card,
body.in-session-view button {
  animation: none !important;
  transition: none !important;
}
```

터미널 반응성을 방해하는 애니메이션 방지.

## 🎨 주요 인터렉션 패턴

### 세션 생성 플로우

```
사용자가 "+" 버튼 클릭 → handleCreateSession()
    ↓
showCreateModal = true 상태 변경
    ↓
LitElement 재렌더링
    ↓
session-create-form 컴포넌트 표시
    ↓
사용자가 제출 → session-created 이벤트
    ↓
앱이 리스트에서 세션 대기, 그 후 handleNavigateToSession()
```

### 터미널 종료 애니메이션

```
사용자가 종료 버튼 클릭
    ↓
killing = true 애니메이션 시작
    ↓
killingFrame이 간격마다 증가
    ↓
회전하는 프레임으로 애니메이션 렌더링
    ↓
완료 시: 세션 제거, session-killed 이벤트
```

### 모바일에서 사이드바 토글

```
사용자가 햄버거 메뉴 클릭
    ↓
sidebarCollapsed = !sidebarCollapsed
    ↓
CSS 변환: translateX(-100%) (숨김) 또는 translateX(0) (표시)
    ↓
모바일 오버레이 표시/숨김
    ↓
localStorage에 상태 저장
```

## 📊 요약

VibeTunnel의 디자인 시스템은 다음의 정교한 통합입니다:

### CSS 아키텍처
- **Tailwind CSS**: 빠른 유틸리티 기반 스타일링
- **CSS 변수**: 동적 테마 및 반응형 레이아웃 상수
- **계층화된 스타일**: 베이스, 컴포넌트, 유틸리티 레이어
- **커스텀 폰트**: Hack Nerd Font Mono 및 Fira Code

### UI 인터렉션
- **LitElement**: 이벤트 기반 아키텍처로 반응형 컴포넌트
- **이벤트 버블링**: 효율적인 이벤트 위임
- **키보드 단축키**: 전역 및 세션별 단축키
- **IME 지원**: 한중일 입력 방법 편집기

### 반응형 디자인
- **3단계 중단점**: 모바일(768px), 태블릿(1024px), 데스크톱(1280px)
- **ResizeObserver**: 효율적인 뷰포트 추적
- **동적 뷰포트 높이**: 모바일 주소 표시줄 및 키보드 대응
- **터치 최적화**: 플랫폼별 터치 처리

### 브라우저 적응
- **기능 감지**: View Transitions, ResizeObserver 폴백
- **iOS 최적화**: Safe areas, `-webkit-fill-available`
- **Android 지원**: 키보드 가시성, 터치 제스처
- **테마 시스템**: 시스템 설정 감지 및 localStorage 지속성

### 성능
- **CSS Containment**: 레이아웃 및 스타일 격리
- **RequestAnimationFrame**: 효율적인 렌더링 큐
- **상태 캐싱**: 불필요한 조회 방지
- **애니메이션 제어**: 세션 뷰에서 선택적으로 비활성화

이 아키텍처는 모든 기기 유형에서 반응형 성능, 모바일 접근성, 원활한 터미널 인터렉션을 우선시합니다.

## 📍 주요 파일 위치 참조

| 컴포넌트 | 파일 위치 |
|---------|----------|
| 메인 스타일시트 | `web/src/client/styles.css` |
| 앱 진입점 | `web/src/client/app.ts` |
| 반응형 유틸리티 | `web/src/client/utils/responsive-utils.ts` |
| 모바일 유틸리티 | `web/src/client/utils/mobile-utils.ts` |
| 터미널 컴포넌트 | `web/src/client/components/terminal.ts` |
| 바이너리 터미널 | `web/src/client/components/vibe-terminal-binary.ts` |
| 입력 관리자 (데스크톱) | `web/src/client/components/session-view/input-manager.ts` |
| 입력 관리자 (모바일) | `web/src/client/components/session-view/mobile-input-manager.ts` |
| 세션 카드 | `web/src/client/components/session-card.ts` |
| HTML 엔트리 | `web/assets/index.html` |
