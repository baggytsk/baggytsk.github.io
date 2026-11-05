# SSE (Server-Sent Events) — 폴링·WebSocket과 뭐가 다르고 왜 SSE였나

> 공부한 날: 11/5 · 출처: [rlaope/estudy — 웹소켓, HTTP와의 차이](https://github.com/rlaope/estudy/blob/master/brains/Back-End/Internet/websocket.md) · [MDN — Server-Sent Events 사용하기](https://developer.mozilla.org/ko/docs/Web/API/Server-sent_events/Using_server-sent_events)
> 관련: [jpa/persistence-context.md](../jpa/persistence-context.md) (같은 `SseService.subscribe()`의 재확인 버그) · (예정) `messaging/rabbitmq.md`

## 한눈에

**SSE는 HTTP 응답 하나를 닫지 않고 열어 둔 채, 서버가 클라이언트에게 이벤트를 여러 번 보내는 단방향 통신이다.** 클라이언트가 서버로 보낼 게 없고 서버가 "상태가 바뀌었다"만 알리면 되는 화면이면 SSE로 충분하다.

- 폴링 = 클라이언트가 계속 묻는다 / SSE = 서버만 보낸다 / WebSocket = 둘 다 보낸다
- Spring `SseEmitter`는 Servlet 비동기라 **요청 스레드는 바로 반납되고 연결만 남는다**
- 브라우저 `EventSource`는 **헤더를 못 넣는다** → MINE은 JWT 대신 30초짜리 일회용 토큰을 URL에 넣었고, 그 대가로 토큰 발급 경로가 하나 더 생겼다

## 꼬리 질문

면접에서 나올 순서대로 (11/5 [우사기3] 면접관 검토)

- [ ] 요청 수 91% 감소는 무엇을 잰 건가? 당연히 주는 것 아닌가? → [내 프로젝트에서](#내-프로젝트에서)
- [ ] WebSocket이 이미 있는데 왜 SSE였나? → [내 프로젝트에서](#내-프로젝트에서)
- [ ] 연결 50개를 열어 두면 스레드 50개를 잡나? → [조건과 예외](#조건과-예외)
- [ ] 토큰을 URL에 넣어도 되나? 재연결하면 토큰은? → [조건과 예외](#조건과-예외)
- [ ] 구독하기 전에 분석이 끝나 버리면? → [조건과 예외](#조건과-예외)
- [ ] 서버가 여러 대면 이벤트가 연결을 가진 서버까지 어떻게 가나? → [조건과 예외](#조건과-예외)
- [ ] 폴링·SSE·WebSocket은 누가 보내고, 연결은 얼마나 유지되나? → [인접 개념](#인접-개념)

---

## 비유

| 방식 | 비유 |
|---|---|
| 폴링 | 배달 앱을 3초마다 새로고침하며 "왔어?" 묻기 |
| SSE | 라디오. 주파수를 한 번 맞추면 방송국만 말한다 |
| WebSocket | 전화. 연결 후 양쪽 다 말한다 |

## 정의

- **SSE**: 서버가 `Content-Type: text/event-stream` 응답을 끝내지 않고, 이벤트가 생길 때마다 아래 형식의 글자를 이어 쓴다
- **EventSource**: 브라우저가 SSE를 받는 API. `new EventSource(url)`로 연결을 연다
- **SseEmitter**: Spring MVC에서 SSE 응답을 보내는 객체. 컨트롤러가 이걸 반환하면
  - 요청 스레드는 풀로 돌아가고(Servlet 비동기) **연결과 emitter 객체만 남는다**
  - 반환 **전에** 보낸 이벤트는 Spring이 버퍼에 담아 뒀다가 연결이 준비되면 내보낸다 (MINE `emitImmediately`가 동작하는 이유)

이벤트 스트림 형식 (빈 줄 = 이벤트 하나 끝):

```
event: status-update
data: {"status":"PROCESSING","step":"AUDIO_ANALYSIS"}

: keep-alive

event: status-update
data: {"status":"COMPLETED"}
```

| 필드 | 뜻 | MINE |
|---|---|---|
| `event:` | 이벤트 이름 | 전부 `status-update` |
| `data:` | 내용 | `{status, step}` |
| `: ` (콜론으로 시작) | 주석. 브라우저가 무시 | 30초 heartbeat `: keep-alive` |
| `id:` / `retry:` | 재연결 때 이어받을 위치 / 재연결 대기 시간 | 안 씀 |

## 왜 쓰나

폴링은 결과가 없어도 계속 요청을 보내고, 결과가 나와도 다음 요청 때까지 최대 간격(3초)만큼 늦게 안다. SSE는 연결 하나로, **서버가 결과를 아는 순간** 브라우저에 보낸다.

- ⚠️ "서버가 아는 순간"이다. MINE은 이 시점에 서버→AI 구간이 여전히 폴링(3초 × 최대 60회)이라, 서버도 AI 결과를 다음 폴링 때 안다. SSE가 없앤 건 **브라우저↔서버 구간의 최대 3초**뿐이다

```java
// 클라이언트: 연결 열기 (헤더 못 넣음)
const es = new EventSource(`/cards/${cardId}/attempts/${attemptId}/stream?token=${token}`);
es.addEventListener("status-update", (e) => { /* JSON.parse(e.data) */ });

// 서버: 응답 통로를 만들어 반환 (AttemptSseController.subscribe → SseService.subscribe)
SseEmitter emitter = new SseEmitter(185_000L);
emitter.send(SseEmitter.event().name("status-update").data(Map.of("status", "COMPLETED")));
emitter.complete();
```

## 동작 흐름

MINE 솔로 모드 (녹음 → AI 피드백):

1. 녹음 업로드 완료
2. `POST /stream-token` (JWT 인증) → Redis에 `sse:token:{UUID}` 저장, TTL 30초
3. `GET /stream?token=...` → Redis `getAndDelete`로 토큰 확인과 삭제를 한 번에 → attemptId 일치 확인
4. `SseEmitter(185초)`를 만들어 `ConcurrentHashMap<attemptId, emitter>`에 등록
5. 분석 중이면 첫 이벤트로 현재 단계(`AUDIO_ANALYSIS` / `FEEDBACK_GENERATION`) 전송
6. 중간 단계는 `push()` (연결 유지), 끝나면 `emit()` → 최종 상태 전송 후 `complete()`

연결이 끊기는 경우:

| 누가/언제 | 클라이언트 | 서버 (MINE) |
|---|---|---|
| 서버가 `complete()` | 브라우저는 다시 연결을 시도 → 토큰이 이미 없어 401 → `error` 이벤트 → `CLOSED`, 더는 재시도 안 함. 깔끔하게 하려면 완료 이벤트를 받고 `es.close()` [확인 필요: FE 코드] | `onCompletion` → 맵에서 제거 |
| 185초 지남 | 위와 같음 | `onTimeout` → 제거 |
| 네트워크 끊김 | 재연결 1번 → 401 → `CLOSED`. 결과를 계속 받으려면 FE가 `onerror`에서 토큰을 새로 받아 다시 연결해야 함 | 바로 모름. heartbeat 쓰기가 **실패하면** 맵에서 뺌. 첫 쓰기는 TCP 버퍼에 들어가 성공할 수 있어서 몇 주기 늦게 알 수도 있음 [추측]. 이때 `complete()`는 안 부르고 맵에서만 뺌 |

## 조건과 예외

- ⚠️ **스레드·연결 점유**: SSE 연결 50개 ≠ 요청 스레드 50개. `SseEmitter`를 반환하면 Tomcat 요청 스레드는 반납된다. 남는 건 TCP 연결과 emitter 객체(메모리)라서, 상한은 스레드 풀이 아니라 Tomcat `max-connections`와 메모리에 걸린다. OSIV는 prod·release에서 `false`라 연결을 여는 동안 DB 커넥션을 잡지 않는다 (`application-prod.yml:19`)
  - 예전에 "스레드 수가 비슷한 건 Virtual Thread 효과"라고 해석한 적이 있는데 **틀렸다**. `spring.threads.virtual` 미설정이라 Tomcat은 플랫폼 스레드다. 이유는 Servlet 비동기 (MINE 00-facts 9절)
- ⚠️ **EventSource는 커스텀 헤더를 못 넣는다** (`withCredentials`로 쿠키만). `Authorization: Bearer JWT`를 못 쓰니 토큰을 URL에 넣어야 하는데, URL은 서버 접근 로그·브라우저 기록·프록시에 남는다 → 진짜 JWT 대신 **30초 + 한 번 쓰면 사라지는 토큰**을 넣는다. 새어도 이미 쓴 토큰이거나 곧 만료된다
  - 대가: JWT 필터를 그대로 못 써서 발급 API·Redis 저장·`getAndDelete`가 따로 생겼다
- ⚠️ **일회용 토큰 + 자동 재연결**: 브라우저가 같은 URL로 다시 연결하면 토큰은 이미 지워져서 실패한다. EventSource는 200 + `text/event-stream`이 아닌 응답을 받으면 재시도를 멈춘다. 즉 MINE에선 **자동 재연결이 사실상 동작하지 않는다**
- ⚠️ **구독 전에 분석이 끝나면** 보낼 대상이 없어서 이벤트를 영영 못 받는다 → `subscribe()`가 등록 전에 상태를 보고, 이미 끝났으면 바로 보내고 닫는다 (`emitImmediately`). 등록 직후 한 번 더 확인하는데, 이 재확인이 1차 캐시 때문에 DB를 다시 안 읽었다 → [jpa/persistence-context.md](../jpa/persistence-context.md)
- ⚠️ **send를 부르는 스레드가 셋**: 요청 스레드(첫 이벤트), `sse-heartbeat` 스케줄러, Redis 리스너(`SoloRedisSubscriber`). `ConcurrentHashMap`은 맵만 안전하게 해 주고, 같은 emitter에 동시에 send해도 글자가 섞이지 않는 건 Spring이 send 안에서 동기화하기 때문이다 [추측: 버전별 구현 확인 필요]. **순서는 보장 안 된다** → 첫 이벤트(옛 단계 `AUDIO_ANALYSIS`)가 등록 직후 들어온 `FEEDBACK_GENERATION`보다 늦게 갈 수 있다
- **emitter 맵은 서버 메모리**: 서버가 여러 대면 연결을 가진 서버에만 emitter가 있다. MINE은 결과를 Redis Pub/Sub으로 뿌리고, 각 서버의 `SoloRedisSubscriber`가 자기 맵에 있으면 `push`/`emit`한다 (Subscriber는 팀원 mamc325 구현 `fd66389`, 나는 emitter 쪽)
- **프록시**: nginx를 거치면 기본 버퍼링 때문에 이벤트가 늦게 갈 수 있다 → `proxy_buffering off` 또는 응답 헤더 `X-Accel-Buffering: no`. 유휴 연결은 `proxy_read_timeout`(기본 60초)에 끊기니 heartbeat 30초는 그 안쪽이다 [확인 필요: MINE 레포에서 nginx 설정 못 찾음]
- 타임아웃 185초 = 서버→AI 폴링 최대 대기 180초(3초 × 60회) + 여유 5초 (`SseEmitterRegistry.java:26` 주석). 컨트롤러 Swagger 문서엔 EXPIRED가 "80초 초과"로 적혀 있어 코드끼리 안 맞는다 [확인 필요]
- HTTP/1.1에선 브라우저가 한 도메인에 연결을 6개 정도만 연다 → SSE 탭을 여러 개 열면 막힐 수 있다. HTTP/2면 완화 [MDN]

## 인접 개념

| | 폴링 | SSE | WebSocket |
|---|---|---|---|
| 누가 보내나 | 클라이언트가 묻고 서버가 답 | 서버 → 클라이언트만 | 양쪽 다 (전이중) |
| 연결 | 요청마다 새로 | 응답 하나를 열어 둠 | 연결 하나를 열어 둠 |
| 프로토콜 | HTTP | HTTP 그대로 | HTTP로 시작해 Upgrade 후 WebSocket (`ws://`, `wss://`) |
| 데이터 | 자유 | 텍스트(UTF-8)만 | 텍스트·바이너리 |
| 끊기면 | 다음 요청이 곧 재시도 | 브라우저가 자동 재연결 (MINE은 일회용 토큰이라 1번 실패 후 멈춤) | 직접 구현 |
| 인증 헤더 | 넣을 수 있음 | **못 넣음** (EventSource) | 브라우저 API는 못 넣음 → 보통 첫 메시지나 URL로 [추측: MINE PvP 방식은 미확인] |
| 맞는 곳 | 드물게 바뀌는 값, 단순함 우선 | 알림·진행 상황·피드 | 채팅·게임·공동 편집 |

용어 메모
- **TCP**: 데이터가 빠지거나 순서가 꼬이지 않게 보내 주는 연결 (4계층). HTTP·SSE·WebSocket 모두 TCP 위에서 돈다
- **전이중**: 양쪽이 동시에 보낼 수 있음 (전화). 반이중은 번갈아 (무전기)
- **핸드셰이크**: 본론 전 "연결할래?" "좋아". WebSocket은 HTTP로 `Upgrade: websocket`을 보내고 서버가 `101 Switching Protocols`로 답하면 전환
- WebSocket에서 두 사용자는 직접 연결되지 않는다. A↔서버, B↔서버이고 서버가 중계한다

## 장단점

| 장점 | 단점 |
|---|---|
| HTTP 그대로라 Spring MVC·프록시 구성을 거의 그대로 씀 | 클라이언트 → 서버는 못 보냄 (별도 HTTP 요청 필요) |
| 자동 재연결이 브라우저에 내장 (단, 일회용 토큰이면 FE가 토큰을 다시 받아야 함) | EventSource에 헤더를 못 넣어 **인증 우회용 토큰 발급 경로가 추가됨** |
| 폴링보다 요청 수가 크게 줆 | 연결을 오래 잡고 있음 → 연결 수·메모리, 다중 서버면 이벤트 전달 경로, 프록시 버퍼링 설정 필요 |

트레이드오프 한 줄: **양방향이 필요 없으면 SSE가 WebSocket보다 프로토콜은 단순하지만, 인증·재연결·다중 서버·프록시는 따로 챙겨야 한다.**

## 내 프로젝트에서

### MINE — 솔로 모드 AI 피드백 (폴링 → SSE)

- 주어: SSE 전환은 본인 (#244, 03-06, `global/sse/` 커밋 전부 본인). AI 도움을 받아 빠르게 짰고, 원리는 이 TIL로 다시 확인 [본인 11/5]. 다중 서버 전달(`SoloRedisSubscriber`)은 팀원 구현
- 시간 순서 [MINE 00-facts, wiki#90·#244]
  - 01-14 PL 미팅: "FE-BE 폴링을 SSE로 전환" 방향을 설계 단계에서 이미 정함 (본인 작성)
  - MVP1 출시(02월)는 폴링으로 (3초 간격 최대 10회)
  - 02-11 PvP에 WebSocket이 들어옴(팀원) → 본인이 "있는 WebSocket을 쓰자"고 제안했다가, 기술 스택 문서의 **양방향/단방향 기준**으로 다시 판단해 SSE로 결정
  - 03-05 이슈 #244 → 03-06 구현 → 03-09 경쟁 조건 수정
- 왜 SSE: AI 결과를 기다리는 동안 **클라이언트가 서버에 보낼 게 없다**. 서버가 "분석 중 → 피드백 생성 중 → 완료"만 알리면 된다
- 측정 (k6, 03-09 폴링 / 03-10 SSE, 인스턴스 1대, 50 VU, k6 스크립트는 #254 공동)
  - Notion 측정 기록 값: `/attempts` TPS 피크 ~14 → ~1.2 req/s, p95 165 → 21.1 ms
  - **한계 3개**: ① 업로드 호출이 없어 시도가 PENDING에 머묾 → AI 분석 없이 **대기 요청 부하만** 잼 ② Before/After가 다른 인스턴스 ③ Grafana 캡처로는 14→1.2가 결과 조회 API 값인지 확인 불가 (MINE 00-facts 680·693행)
  - 그래서 말할 수 있는 건 **"폴링 최대 10회가 연결 1회로 바뀌어 요청 수가 준 것을 확인"**까지. p95는 "결과 조회 API 응답 시간"이지 사용자가 결과를 받기까지의 지연이 아니다. "결과 수신 지연 ~0ms"는 계산값이라 쓰지 않는다
  - 다시 잰다면: 업로드 포함 시나리오로 **결과 도착까지 걸린 시간**, 동시 연결 N개일 때 메모리·연결 수
- 코드: `AttemptSseController.java:35~37` (두 단계 흐름) · `SseTokenService.java:61` (getAndDelete) · `SseEmitterRegistry.java:27` (185초), `:54` (`register`, 정리 콜백), `:113` (heartbeat) · `SoloRedisSubscriber` (다중 서버 전달, 팀원)

### 알게 된 결함 (11/5 [우사기3] 검토로 찾음, 코드 확인)

| 결함 | 위치 | 고칠 방향 |
|---|---|---|
| 같은 attemptId로 다시 구독하면 `put()`이 옛 emitter를 닫지 않고 덮어씀 | `SseEmitterRegistry.register()` | put 전에 기존 emitter `complete()` |
| `push()` 실패 시 `remove(attemptId)`만 해서 재구독한 새 emitter를 지울 수 있음 (정리 콜백은 `remove(attemptId, emitter)`로 막아 둠) | `SseEmitterRegistry.java:86` | `remove(attemptId, emitter)`로 통일 |
| heartbeat 실패 시 맵에서만 빼고 `complete()` 안 함 | `SseEmitterRegistry.java:113~121` | 실패한 emitter `completeWithError` |

- 고치지 않음 (11/5 baggy 결정: 서비스 종료된 지난 프로젝트). 면접에서 말할 땐 주어를 사실대로: "AI 코드 리뷰로 찾고, 코드에서 확인했다"

면접 문장 (baggy가 씀):
- (아직 없음)

## 실제 면접에서 받은 질문

- (아직 없음)

## 공부 기록

| 날짜 | 질문 | 바뀐 곳 |
|---|---|---|
| 11/5 | SSE는 폴링·WebSocket과 뭐가 다르고 왜 SSE였나 | 처음 작성 |
| 11/5 | [우사기3] 검토 (반대 검토자·면접관·현업 선배) | 꼬리 질문 순서, 스레드·Servlet 비동기, 재연결 실제 동작, send 동시성, 프록시, 측정 한계, 서버→AI 지연 범위, 주어, 알게 된 결함 |

## 남은 것

- [ ] 면접 문장 2~3줄 (baggy)
- [ ] FE가 완료 이벤트 뒤 `es.close()`를 하나, `onerror`에서 토큰을 새로 받나 [확인 필요]
- [ ] nginx 버퍼링·read timeout 설정 [확인 필요]
- [ ] "10명 × 2방향 × 20회 = 400 HTTP" 계산을 전환 전에 했나 [확인 필요]
- [x] 알게 된 결함 3개: **고치지 않음** (11/5 baggy 결정, 서비스 종료된 지난 프로젝트). 면접에선 "찾았고, 고친다면 이렇게"까지
- [ ] (나중에) MINE PvP WebSocket은 인증을 어떻게 했나
