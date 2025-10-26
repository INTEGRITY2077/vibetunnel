# VibeTunnel 터미널 파이프라인 및 시그널 플로우

> VibeTunnel이 터미널에 진입하여 동작하는 전체 파이프라인과 시그널 처리 메커니즘에 대한 상세 문서

## 📊 전체 아키텍처 개요

```
┌──────────────────────────────────────────────────────────────┐
│                    터미널 세션 (PTY)                          │
│  node-pty를 통한 pseudo-terminal                              │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│              Asciinema Writer (녹화)                          │
│  ~/.vibetunnel/control/[session-id]/stdout                   │
│  - 모든 터미널 출력을 asciinema 포맷으로 기록                 │
│  - 리사이즈 이벤트도 함께 기록                                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│         Terminal Manager (파일 감시 + 에뮬레이션)             │
│  - stdout 파일을 실시간 감시                                  │
│  - Headless xterm.js로 터미널 상태 에뮬레이션                 │
│  - 10,000줄 스크롤백 버퍼 유지                                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│            Buffer Snapshot (버퍼 추출)                        │
│  - 현재 화면에 보이는 영역만 추출 (cols × rows)               │
│  - 셀 속성 포함: 색상, 굵게, 밑줄 등                          │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│         Binary Encoder (바이너리 압축)                        │
│  - 최적화된 바이너리 프로토콜로 인코딩                        │
│  - ~70% 크기 감소 효과                                        │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│         WebSocket 전송 (0xbf magic byte)                      │
│  /ws/buffers 엔드포인트로 전송                                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│            클라이언트 (웹 브라우저)                            │
│  - Binary Decoder로 디코딩                                    │
│  - xterm.js UI로 렌더링                                       │
└──────────────────────────────────────────────────────────────┘
```

## 🔄 세션 생성 플로우

**위치**: `web/src/server/pty/pty-manager.ts:293-500`

### 1. 세션 초기화

```typescript
createSession(command: string[], options: SessionCreateOptions)
```

**단계**:

1. **UUID 생성**: 각 세션에 고유 ID 부여

2. **디렉토리 구조 생성**:
   ```
   ~/.vibetunnel/control/[session-id]/
   ├── session.json        # 메타데이터
   ├── stdout              # 출력 스트림 (TerminalManager가 감시)
   └── stdin               # 입력 파이프
   ```

3. **명령어 해석**: `ProcessUtils.resolveCommand()`로 alias, shell builtin 등 처리

4. **PTY 프로세스 생성**:
   ```typescript
   const ptyProcess = pty.spawn(finalCommand, finalArgs, {
     name: 'xterm-256color',
     cols: options.cols || 80,
     rows: options.rows || 24,
     cwd: workingDir,
     env: {
       ...process.env,
       TERM: 'xterm-256color',
       VIBETUNNEL_SESSION_ID: sessionId
     }
   });
   ```

5. **Asciinema 녹화 시작**:
   - 모든 터미널 출력을 stdout 파일에 기록
   - 출력(type 'o')과 리사이즈 이벤트(type 'r') 모두 기록
   - 세션 재생 및 버퍼 재구성 지원

6. **종료 처리**:
   - 종료 코드와 시그널 추적
   - `sessionExited` 이벤트 발생
   - 세션 상태를 'exited'로 업데이트

## 🎯 시그널 처리 메커니즘

**위치**: `web/src/server/pty/pty-manager.ts:194-290`

### SIGWINCH (터미널 리사이즈) 처리

VibeTunnel은 **두 가지 소스**에서 리사이즈를 감지합니다:

#### 소스 1: 호스트 터미널 리사이즈

```typescript
private setupTerminalResizeDetection() {
  // 방법 1: Node.js TTY 리사이즈 이벤트
  process.stdout.on('resize', () => {
    const cols = process.stdout.columns;
    const rows = process.stdout.rows;
    this.handleHostTerminalResize(cols, rows);
  });

  // 방법 2: Unix SIGWINCH 시그널 (백업)
  process.on('SIGWINCH', () => {
    this.handleHostTerminalResize(
      process.stdout.columns,
      process.stdout.rows
    );
  });
}
```

**동작 원리**:

1. 호스트 터미널 크기 변경 감지
2. **"마지막 리사이즈 우선" 로직** 적용:
   - 터미널 리사이즈가 우선권을 가짐
   - 단, 브라우저 리사이즈 후 1초 유예 기간 존재
   - 리사이즈 피드백 루프 방지
3. 모든 활성 PTY 프로세스 업데이트: `ptyProcess.resize(cols, rows)`
4. Asciinema 파일에 리사이즈 이벤트 기록: `asciinemaWriter.writeResize(cols, rows)`

#### 소스 2: 브라우저 리사이즈

- WebSocket으로 수신: `{ cmd: 'resize', cols, rows }`
- `resizeSession()` 메서드에서 처리
- 리사이즈 소스와 타임스탬프를 추적하여 우선순위 구현

#### 빠른 리사이즈 루프 감지

```typescript
if (timeSinceLastResize < 100) {
  // 잠재적 피드백 루프 감지
  logger.warn(`Rapid resize detected`);
}
```

### 주요 시그널 처리 목록

| 시그널 | 소스 | 핸들러 | 액션 |
|--------|------|--------|------|
| **SIGWINCH** | 호스트 터미널 | `handleStdoutResize()` | 모든 활성 PTY 프로세스 리사이즈 |
| **SIGCHLD** | PTY 종료 | `ptyProcess.onExit()` | 세션 상태 업데이트, 이벤트 발생 |
| **Ctrl+C** | 브라우저 입력 | 특수 키 매핑 | PTY에 `\x03` 전송 |
| **Ctrl+D** | 브라우저 입력 | 특수 키 매핑 | PTY에 `\x04` (EOF) 전송 |

## 🔧 터미널 버퍼 관리 및 플로우 컨트롤

**위치**: `web/src/server/services/terminal-manager.ts` (전체 파일)

### Headless 터미널 인스턴스

```typescript
const terminal = new XtermTerminal({
  cols: 80,
  rows: 24,
  scrollback: 10000,      // 10,000줄 히스토리
  convertEol: true,       // 다양한 줄바꿈 처리
});
```

### 스트림 파일 감시 및 처리

#### 파일 감시 설정 (`watchStreamFile()`)

- `~/.vibetunnel/control/[session-id]/stdout` 파일 감시
- 추가된 데이터를 증분적으로 읽음 (오프셋 추적)
- Asciinema JSON 포맷을 줄 단위로 파싱

#### Asciinema 이벤트 처리 (`handleStreamLine()`)

```typescript
// 헤더: { version: 2, width: 80, height: 24 }
// 출력: [timestamp, "o", data]  → terminal.write(data)
// 리사이즈: [timestamp, "r", "80x24"]  → terminal.resize(80, 24)
// 입력: [timestamp, "i", data]  → 무시 (재생용으로만 기록)
```

### 플로우 컨트롤 시스템

메모리 고갈 방지를 위한 정교한 **백프레셔(backpressure)** 메커니즘:

```typescript
const FLOW_CONTROL_CONFIG = {
  highWatermark: 0.8,        // 80% 채워지면 일시정지
  lowWatermark: 0.5,         // 50%로 떨어지면 재개
  checkInterval: 100,        // 100ms마다 체크
  maxPendingLines: 10000,    // 최대 큐 크기
  maxPauseTime: 5 * 60 * 1000 // 5분 타임아웃
};
```

**동작 방식**:

1. **High Watermark (80%)**: 버퍼가 80% 차면 파일 감시 일시정지
2. **Low Watermark (50%)**: 50%로 떨어지면 감시 재개
3. **대기 큐**: 일시정지 중 최대 10,000줄까지 큐에 저장
4. **타임아웃**: 5분 이상 일시정지 시 대기 데이터 삭제
5. **백프레셔**: 파일 감시를 일시정지하여 메모리 고갈 방지

### 레이트 제한 쓰기

```typescript
private queueTerminalWrite(sessionId, data) {
  queue.push(data);
  // 배치 간 10ms 딜레이로 처리 (lines 1040-1085)
  // 터미널 에뮬레이터 과부하 방지
}
```

### 버퍼 스냅샷 추출

`getBufferSnapshot(sessionId)` 메서드가 **화면에 보이는 터미널 영역** (cols × rows)을 추출:

- 커서 위치 획득 (뷰포트 기준)
- 셀 속성 추출:
  - 문자 데이터
  - 색상 (전경/배경)
  - 텍스트 속성 (굵게, 기울임, 밑줄, 흐림, 반전, 취소선)
- 빈 행/셀을 제거하여 전송 효율성 향상

## 📦 바이너리 프로토콜 (WebSocket 전송)

**위치**: `web/src/server/services/terminal-manager.ts:722-957`

### 인코딩 포맷

`encodeSnapshot()` 메서드가 고도로 최적화된 바이너리 메시지 생성:

#### 헤더 (32바이트)

```
오프셋  크기  필드
0       2     Magic: "VT" (0x5654)
2       1     Version: 0x01
3       1     Flags: 0x00
4       4     Cols (uint32 LE)
8       4     Rows (uint32 LE)
12      4     ViewportY (int32 LE, signed)
16      4     CursorX (int32 LE)
20      4     CursorY (int32 LE)
24      4     Reserved
28      4     Reserved
```

#### 셀 데이터

- **빈 행**: `0xfe 0x01` (2바이트)
- **행 마커**: `0xfd [길이:uint16LE]` (3바이트)
- **단순 공백**: `0x00` (1바이트)
- **ASCII 문자**: 타입 바이트 + 문자 코드 (2+ 바이트)
- **유니코드 문자**: 타입 바이트 + 길이 + UTF-8 (가변)
- **색상/속성 포함**: 타입 바이트 + 문자 + 속성 + 색상 (2-8 바이트)

#### 타입 바이트 플래그

```
Bit 7: 확장 데이터 존재 (속성/색상)
Bit 6: 유니코드 여부 (vs ASCII)
Bit 5: 전경색 존재
Bit 4: 배경색 존재
Bit 3: RGB 전경색 (vs 팔레트)
Bit 2: RGB 배경색 (vs 팔레트)
Bits 1-0: 문자 타입 (00=공백, 01=ASCII, 10=유니코드)
```

**압축 효과**: 평균 **~70% 크기 감소**

### 실시간 알림

- 50ms 디바운싱: `scheduleBufferChangeNotification()`
- 버퍼 변경 시 모든 구독 리스너에게 알림
- 리스너가 스냅샷을 인코딩하여 WebSocket으로 전송

## 🌐 Buffer Aggregator 및 WebSocket 배포

**위치**: `web/src/server/services/buffer-aggregator.ts`

### 바이너리 프로토콜 래퍼

```typescript
// Magic byte 0xbf가 바이너리 메시지를 나타냄
[0xbf] + [sessionIdLen:uint32LE] + [sessionId:utf8] + [encodedBuffer]
```

### 구독 모델

1. **클라이언트 구독**: `{ type: 'subscribe', sessionId }`
2. **Aggregator가 구독**: 해당 세션의 로컬 TerminalManager에 구독
3. **스냅샷 캡처 및 인코딩**: 버퍼 변경 시마다
4. **프로토콜로 래핑 후 클라이언트에 전송**

### 원격 세션 지원 (HQ 모드)

- Headquarters 모드에서 원격 서버로 프록시 가능
- 원격 인스턴스에 WebSocket 연결 유지
- 구독 요청 전달: `remoteWs.send({ type: 'subscribe', sessionId })`
- 수신한 버퍼를 구독한 클라이언트에게 중계

## ⌨️ 입력 처리 및 특수 키

**파일**:
- 입력 수신: `web/src/server/routes/websocket-input.ts`
- 입력 처리: `web/src/server/pty/pty-manager.ts`

### WebSocket 입력 프로토콜

**일반 텍스트**: UTF-8 문자열로 전송

**특수 키**: null 바이트로 감싸서 전송

```
키: "\x00enter\x00"     → Enter 키
키: "\x00escape\x00"    → Escape 키
키: "\x00tab\x00"       → Tab 키
텍스트: "Enter"         → 문자 그대로 "Enter" 문자열
```

### 서버 측 입력 처리 (`sendInput()`)

1. **특수 키 감지** (null 바이트로 감싸진 경우)
2. **이스케이프 시퀀스로 변환**:
   ```typescript
   'enter' → '\r\n'
   'escape' → '\x1b'
   'tab' → '\t'
   'ctrl_c' → '\x03'
   'ctrl_d' → '\x04'
   'backspace' → '\x7f'
   // 등등...
   ```

3. **PTY에 쓰기**: `ptyProcess.write(dataToSend)`
4. **출력 캡처**: Asciinema 레코더가 캡처하여 TerminalManager로 피드백

### 쓰기 큐 관리

- 쓰기를 배치 처리 (한 번에 10개)
- 배치 간 10ms 딜레이
- 터미널 에뮬레이터 플러딩 방지

## 💾 세션 라이프사이클 및 지속성

**위치**: `web/src/server/pty/session-manager.ts`

### 세션 디렉토리 구조

```
~/.vibetunnel/control/[session-id]/
├── session.json          # 메타데이터 (임시 파일을 통한 원자적 쓰기)
├── stdout                # Asciinema 스트림 파일
└── stdin                 # 입력 파이프/파일 (FIFO 또는 일반 파일)
```

### 세션 메타데이터

```typescript
{
  id: string,
  name: string,
  command: string[],
  status: 'starting' | 'running' | 'exited',
  pid?: number,
  exitCode?: number,
  startedAt: ISO8601 timestamp,
  lastModified: ISO8601 timestamp,
  workingDir: string,
  initialCols?: number,
  initialRows?: number,
  version: string,
  gitRepoPath?: string,
  attachedViaVT: boolean
}
```

### 고유 이름 처리

- 이름 충돌 시 자동으로 접미사 추가: `"Terminal (2)"`, `"Terminal (3)"` 등
- 모든 활성 세션과 비교하여 검사

### 버전 추적

- `.vibetunnel/control/.version`에 마지막 알려진 버전 저장
- 업그레이드 시 정리에 사용

## 🖥️ 클라이언트 측 렌더링 및 입력

**파일**:
- 바이너리 터미널: `web/src/client/components/vibe-terminal-binary.ts`
- 연결 관리: `web/src/client/components/session-view/connection-manager.ts`
- 입력 처리: `web/src/client/components/session-view/input-manager.ts`

### 바이너리 버퍼 수신

1. **WebSocket 연결**: `/ws/buffers`로 연결
2. **구독 메시지**: `{ type: 'subscribe', sessionId }`
3. **바이너리 메시지 수신**: magic byte `0xbf`로 시작
4. **파싱**: 세션 ID 길이와 세션 ID 추출
5. **디코딩**: 터미널 버퍼 스냅샷 디코딩
6. **렌더링**: 클라이언트 측 xterm.js UI로 렌더링

### 입력 전송

1. **키보드 캡처**: 숨겨진 input 엘리먼트를 통해
2. **특수 키 감지** vs 일반 텍스트
3. **WebSocket으로 전송**: `/ws/input?sessionId=...`
4. **Fire-and-forget** (확인 응답 없음)

### 스트림 연결

- **SSE (Server-Sent Events)** 연결 유지 (세션 라이프사이클 이벤트용)
- 자동 재연결 (5초 창 내에서 최대 3회 재연결)
- 스트림 실패 시 스냅샷 로딩으로 폴백

## 🔁 완전한 데이터 플로우 다이어그램

### 출력 플로우 (터미널 → 브라우저)

```
터미널 (PTY)
    ↓
node-pty [프로세스 spawn]
    ↓
Asciinema Writer [stdout 파일에 기록]
    ↓
Terminal Manager [파일 감시자가 stdout 감시]
    ↓
xterm.js Headless [asciinema 파싱, 터미널 에뮬레이션]
    ↓
Buffer Snapshot [보이는 영역 + 속성 추출]
    ↓
Binary Encoder [압축 및 최적화]
    ↓
Buffer Aggregator [프로토콜로 래핑 + 배포]
    ↓
WebSocket [0xbf magic byte + sessionId + 인코딩된 버퍼]
    ↓
Client WebSocket [/ws/buffers]
    ↓
Binary Decoder [프로토콜 파싱 및 버퍼 추출]
    ↓
xterm.js UI [브라우저에서 렌더링]
```

### 입력 플로우 (브라우저 → 터미널)

```
키보드 (브라우저)
    ↓
Input Manager [특수 키 vs 텍스트 감지]
    ↓
WebSocket [/ws/input로 raw 텍스트 또는 null로 감싼 키 전송]
    ↓
WebSocket Input Handler [수신 및 특수 키 감지]
    ↓
Special Key Converter [이스케이프 시퀀스로 매핑]
    ↓
PTY Process.write() [셸로 전송]
    ↓
출력이 asciinema에 캡처됨
    ↓
[위의 출력 플로우로 다시 순환]
```

## ⚡ 성능 최적화 기법

1. **플로우 컨트롤**: 백프레셔로 메모리 고갈 방지 (80%/50% 워터마크)
2. **바이너리 인코딩**: 고도로 압축된 터미널 업데이트 (~70% 크기 감소)
3. **디바운싱**: 50ms 버퍼 업데이트 디바운스로 과도한 알림 방지
4. **레이트 리미팅**: 10ms 배치 딜레이로 터미널 에뮬레이터 과부하 방지
5. **증분 파일 읽기**: stdout 파일에 추가된 새 데이터만 읽음
6. **지연 구독**: 클라이언트가 구독할 때만 파일 감시 시작
7. **셀 트리밍**: 스냅샷에서 후행 공백과 빈 줄 제거
8. **Headless xterm.js**: DOM 렌더링 없는 경량 에뮬레이션

## 📍 주요 파일 위치 참조

| 컴포넌트 | 파일 위치 |
|---------|----------|
| PTY 생성 | `web/src/server/pty/pty-manager.ts:293-500` |
| 시그널 처리 | `web/src/server/pty/pty-manager.ts:194-290` |
| 터미널 버퍼 | `web/src/server/services/terminal-manager.ts` (전체) |
| 바이너리 인코딩 | `web/src/server/services/terminal-manager.ts:722-957` |
| 버퍼 배포 | `web/src/server/services/buffer-aggregator.ts` (전체) |
| 입력 처리 | `web/src/server/routes/websocket-input.ts` (전체) |
| 세션 지속성 | `web/src/server/pty/session-manager.ts` (전체) |
| 클라이언트 렌더링 | `web/src/client/components/vibe-terminal-binary.ts` (전체) |
| 연결 관리 | `web/src/client/components/session-view/connection-manager.ts` (전체) |
| 클라이언트 입력 | `web/src/client/components/session-view/input-manager.ts` (전체) |

## 🎓 핵심 개념 요약

### 1. 이중 터미널 에뮬레이션
- **서버 측**: Headless xterm.js가 실제 터미널 상태를 에뮬레이션
- **클라이언트 측**: UI xterm.js가 스냅샷을 렌더링
- 이 분리로 인해 클라이언트가 연결/재연결해도 상태 유지 가능

### 2. Asciinema 기반 지속성
- 모든 터미널 활동이 asciinema 포맷으로 기록됨
- 세션 재생, 버퍼 재구성, 감사(audit) 가능
- 표준 포맷으로 외부 도구와 호환성 보장

### 3. 바이너리 프로토콜의 효율성
- 텍스트 기반 프로토콜 대비 ~70% 대역폭 절약
- 셀 단위 압축으로 중복 제거
- 빈 영역을 효율적으로 인코딩

### 4. 백프레셔 및 플로우 컨트롤
- 메모리 고갈 방지 필수적
- 빠른 출력을 생성하는 프로그램(예: `cat large_file`)도 안전하게 처리
- 우아한 성능 저하 (graceful degradation)

### 5. 양방향 리사이즈 동기화
- 호스트 터미널과 브라우저 모두에서 리사이즈 가능
- 피드백 루프 방지 로직
- 모든 연결된 클라이언트에 일관된 크기 유지

---

이 파이프라인은 네트워크 연결을 통해 **최소 지연 시간**과 **최소 대역폭 사용**으로 안정적이고 효율적인 터미널 스트리밍을 보장합니다.
