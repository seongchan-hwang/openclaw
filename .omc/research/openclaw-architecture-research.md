# OpenClaw 아키텍처 리서치
> 시니어 플랫폼 개발자 관점 — 설계 결정(trade-off) 중심
>
> 기준 브랜치: `main` (2026-03-23)

---

## 목차

1. [개요 & 포지셔닝](#1-개요--포지셔닝)
2. [핵심 Primitive 설계](#2-핵심-primitive-설계)
3. [Gateway — 중앙 제어 평면 내부](#3-gateway--중앙-제어-평면-내부)
4. [Agent 실행 모델](#4-agent-실행-모델)
5. [Routing 엔진 내부](#5-routing-엔진-내부)
6. [Tool & Function Calling 설계](#6-tool--function-calling-설계)
7. [Model Abstraction Layer](#7-model-abstraction-layer)
8. [Context Engine & Memory 아키텍처](#8-context-engine--memory-아키텍처)
9. [Plugin 시스템 설계 원칙](#9-plugin-시스템-설계-원칙)
10. [보안 & 신뢰 모델](#10-보안--신뢰-모델)
11. [Slack 연동 — 아키텍처 관점](#11-slack-연동--아키텍처-관점)
12. [AI Company 레퍼런스 아키텍처](#12-ai-company-레퍼런스-아키텍처)

---

## 1. 개요 & 포지셔닝

### OpenClaw가 해결하는 문제

OpenClaw는 **"N개 채널 → M개 AI 에이전트"를 운영 수준으로 연결하는 개인·팀 AI 게이트웨이**다. 핵심 문제 공간은 세 가지다.

1. **채널 파편화**: iMessage, WhatsApp, Slack, Discord, Telegram, Signal, Matrix 등 서로 다른 전송 프로토콜과 이벤트 모델을 통일된 인터페이스 뒤에 숨긴다.
2. **에이전트 생명주기**: LLM 호출 자체가 아니라 세션 연속성, 큐잉, 병렬성 제어, 오류 복구를 포함하는 *실행 모델*을 다룬다.
3. **멀티테넌트 라우팅**: 채널·계정·Guild·역할·대화 상대에 따라 어떤 에이전트가 응답할지를 설정 기반으로 결정하는 *라우팅 엔진*을 내장한다.

이 세 문제는 "LLM API 호출 코드"를 직접 작성하면 반드시 재발명하게 되는 인프라층이다. OpenClaw는 그 층을 제품화한다.

### Bedrock · LangChain · AutoGen과 설계 철학 비교

| 관점 | AWS Bedrock | LangChain | AutoGen | OpenClaw |
|------|------------|-----------|---------|---------|
| **추상화 대상** | 모델 API 균일화 | 체인/도구 합성 | 다중 에이전트 협업 | 채널-에이전트 연결 + 세션 관리 |
| **런타임 형태** | 관리형 클라우드 | 라이브러리 | 프레임워크 | 셀프호스팅 게이트웨이 프로세스 |
| **상태 소유** | 없음 (stateless API) | 개발자 책임 | 프레임워크 내부 | Gateway 프로세스가 세션 상태 소유 |
| **채널 통합** | 없음 | 없음 | 없음 | 핵심 Primitive (Channel) |
| **실시간 라우팅** | 없음 | 없음 | 없음 | 7-tier AgentBinding 매칭 엔진 |
| **확장 모델** | SDK/API | Python 패키지 | Python 클래스 상속 | Plugin SDK (격리된 npm 패키지) |

핵심 설계 철학 차이: Bedrock/LangChain/AutoGen은 **"어떻게 LLM을 호출하는가"**가 중심이다. OpenClaw는 **"누가, 어떤 채널에서, 어떤 메시지를 보내면, 어떤 에이전트의 어떤 세션에 도달하는가"**가 중심이다.

### 레포 구조 맵

```
openclaw/
├── src/                    # 핵심 Gateway 런타임
│   ├── gateway/            # WebSocket RPC 서버, 프로토콜, 인증
│   ├── routing/            # AgentBinding 매칭, 세션키 생성
│   ├── agents/             # Agent 실행 (embedded LLM, ACP 클라이언트)
│   ├── acp/                # Agent Control Protocol (외부 프로세스 연결)
│   ├── channels/           # 채널 공통 인터페이스, RunStateMachine
│   ├── auto-reply/         # 큐, 정책, drain 로직
│   ├── memory/             # 하이브리드 메모리 (벡터+BM25+MMR)
│   ├── context-engine/     # 컨텍스트 윈도우 관리
│   ├── plugins/            # 플러그인 로더, SDK alias, 레지스트리
│   ├── infra/              # DB, 스토리지, 아웃바운드 서비스
│   ├── slack/              # Slack 빌트인 채널 (코어)
│   ├── discord/            # Discord 빌트인 채널
│   ├── telegram/           # Telegram 빌트인 채널
│   └── ...                 # 기타 채널, CLI, 미디어 파이프라인
├── extensions/             # 플러그인 패키지 (격리된 workspace)
│   ├── slack/              # Slack 채널 플러그인
│   ├── msteams/            # MS Teams 채널 플러그인
│   ├── matrix/             # Matrix 채널 플러그인
│   ├── zalo/               # Zalo 채널 플러그인
│   └── ...
├── apps/                   # iOS / macOS / Android 앱
├── docs/                   # Mintlify 문서 (docs.openclaw.ai)
└── scripts/                # 빌드, 패키징, i18n, 배포 스크립트
```

---

## 2. 핵심 Primitive 설계

### 왜 5개로 나눴는가

OpenClaw의 5개 핵심 Primitive는 **변경 빈도와 소유권(ownership)**이 서로 다른 5가지 관심사를 분리한 결과다.

| Primitive | 핵심 책임 | 변경 동인 |
|-----------|----------|----------|
| **Gateway** | 연결 관리, 세션 상태, RPC 라우팅 | 프로토콜 버전, 클라이언트 수 |
| **Agent** | LLM 호출, 도구 실행, 응답 생성 | AI 모델 발전, 도구 추가 |
| **Channel** | 외부 메시징 플랫폼 연결 | 플랫폼 API 변경 |
| **Provider** | LLM API 어댑터 | 모델 제공사 API 변경 |
| **Plugin** | 번들 바깥 확장 | 개별 팀의 요구사항 |

이 분리가 없으면 "Slack API 변경 → Agent 코드 수정", "OpenAI 모델 추가 → Channel 코드 터치" 같은 사고가 반복된다. 각 Primitive는 *다른 Primitive의 내부*를 알 필요가 없도록 계약(interface)을 통해서만 연결된다.

### 각 Primitive의 책임 경계

**Gateway**
- WebSocket 연결 수락 및 인증
- 클라이언트별 세션 상태 소유 (`stateVersion`, snapshot)
- 70+ RPC 메서드 라우팅
- Config 변경 감지 및 핫리로드 트리거
- 책임 *외*: LLM 호출, 채널 메시지 파싱

**Agent**
- 대화 컨텍스트 조립 (시스템 프롬프트 + 히스토리 + 메모리 주입)
- LLM 스트리밍 호출 및 응답 파싱
- 도구 실행 오케스트레이션 (before/after 훅 포함)
- Failover 및 재시도 결정
- 책임 *외*: 메시지를 어떤 채널로 보낼지

**Channel**
- 플랫폼별 인바운드 이벤트 → 표준 Message 객체 변환
- 아웃바운드 표준 Reply → 플랫폼별 API 호출 변환
- 계정(accountId) 인증 상태 관리
- 책임 *외*: 어떤 Agent에게 라우팅할지

**Provider**
- 특정 LLM API의 인증, 요청 구성, 응답 파싱
- 스트리밍 델타 → 표준 청크 변환
- 토큰 카운팅 (모델별 상이한 tokenizer 래핑)
- 책임 *외*: 재시도 정책, 세션 컨텍스트

**Plugin**
- 기존 Primitive 조합의 확장 (새 Channel, 새 도구, 새 메모리 백엔드)
- 격리된 npm 패키지로 배포됨
- `openclaw/plugin-sdk/*`만을 공개 계약으로 사용
- 책임 *외*: Gateway 내부 상태에 직접 접근

### Primitive 간 의존 방향

```
Plugin
  └─ 의존 → Channel / Provider / Agent (plugin-sdk 계약을 통해)
               └─ 의존 → Gateway (런타임 서비스 주입을 통해)
                            └─ 의존 → infra (DB, 스토리지, 아웃바운드)
```

핵심 규칙: **의존은 항상 "좁은 인터페이스 방향"으로만 흐른다.** Plugin이 `src/routing/resolve-route.ts`를 직접 임포트하면 이 규칙이 깨진다. `extensions/*`에서 `src/**` 직접 임포트를 금지하는 빌드 린트 규칙은 이 원칙의 강제 집행 장치다.

---

## 3. Gateway — 중앙 제어 평면 내부

### WebSocket RPC 설계 — 왜 REST가 아닌가

Gateway는 REST API 대신 WebSocket 위의 커스텀 RPC 프레임을 사용한다. 핵심 이유는 **양방향 스트리밍**이다.

- LLM 응답은 스트림으로 도착한다. REST SSE로도 가능하지만, *동일 연결*로 제어 메시지(cancel, heartbeat, config reload 알림)도 보내야 한다.
- iOS 앱, macOS 메뉴바, CLI, 웹 UI가 동일 Gateway에 *동시* 연결한다. 각 클라이언트는 에이전트 상태 변경 이벤트를 실시간으로 받아야 한다.
- 연결 단위 상태(`stateVersion`)를 유지하면, 재연결 시 delta sync만으로 전체 상태를 복원할 수 있다.

프레임 스키마(`src/gateway/protocol/schema/frames.ts`):

```typescript
// 클라이언트 → Gateway 요청
RequestFrame: { type: "req", id: string, method: string, params?: unknown }

// Gateway → 클라이언트 응답
ResponseFrame: { type: "res", id: string, ok: boolean, payload?: unknown, error?: unknown }

// Gateway → 클라이언트 푸시 이벤트
EventFrame: { type: "event", event: string, payload?: unknown, seq?: number, stateVersion?: number }
```

`discriminator: "type"` 필드로 단일 WebSocket 스트림에서 세 종류의 프레임을 구분한다. `seq`와 `stateVersion`은 이벤트 순서 보장과 재연결 delta sync에 사용된다.

### 연결 협상 (hello handshake)

클라이언트는 연결 직후 `ConnectParams`를 전송한다. 이 구조체는 프로토콜 버전 범위(`minProtocol`/`maxProtocol`), 클라이언트 메타데이터, 인증 자격증명, 선언된 capability를 포함한다. Gateway는 `hello-ok` (버전 합의 성공) 또는 `hello-error`로 응답한다.

이 설계는 서버-클라이언트 간 **프로토콜 버전 스키마 협상**을 연결 시점에 완료하여, 이후 프레임 처리에서 버전 분기 로직을 제거한다.

### 70+ RPC 메서드 분류 체계

Gateway RPC 메서드는 기능 도메인으로 분류된다:

| 도메인 | 메서드 예시 | Gateway 직접 처리 여부 |
|--------|-----------|---------------------|
| **세션** | `session.get`, `session.list`, `session.delete` | ✅ Gateway 소유 |
| **에이전트** | `agent.send`, `agent.cancel`, `agent.status` | Agent로 위임 |
| **채널** | `channel.list`, `channel.status`, `channel.connect` | Channel 플러그인으로 위임 |
| **설정** | `config.get`, `config.set`, `config.reload` | ✅ Gateway 소유 |
| **인증** | `auth.login`, `auth.logout`, `auth.status` | ✅ Gateway 소유 |
| **메모리** | `memory.search`, `memory.add`, `memory.delete` | Memory 레이어로 위임 |
| **미디어** | `media.upload`, `media.process` | 미디어 파이프라인으로 위임 |

**Gateway가 직접 처리하는 것**: 클라이언트 연결 상태, 세션 메타데이터, Config 스냅샷, 인증 결정, 이벤트 팬아웃(fan-out)

**Agent에 위임하는 것**: LLM 호출, 도구 실행, 응답 스트리밍, 취소 신호 전파

이 경계의 핵심 기준: **"클라이언트 연결이 없어도 존재해야 하는 상태"는 Gateway가 소유한다.**

### 세션 상태를 Gateway가 소유하는 이유

대화 히스토리와 세션 메타데이터를 Agent에 두지 않고 Gateway가 소유하는 이유는 **다중 클라이언트 일관성** 때문이다.

- iOS 앱과 macOS 앱이 동시에 열려 있을 때 둘 다 같은 대화 히스토리를 보아야 한다.
- Agent 프로세스가 재시작되어도 대화 상태는 유지되어야 한다.
- `stateVersion` 증분을 통해 클라이언트는 자신이 놓친 이벤트의 범위를 알 수 있다.

### Config 핫리로드 메커니즘

Config 파일 변경을 감지하면 Gateway는 다음 순서로 핫리로드를 수행한다:

1. 새 Config를 파싱하고 유효성 검증
2. 라우팅 캐시 무효화 (WeakMap 기반, 구 Config 객체가 GC되면 자동 해제)
3. `config.changed` 이벤트를 모든 연결된 클라이언트에 팬아웃
4. Agent에 재초기화 신호 전달

WeakMap을 캐시 키로 사용하는 이유: Config 객체의 참조가 사라지면 캐시 항목도 자동으로 GC된다. 명시적 캐시 무효화 코드가 필요 없다.

---

## 4. Agent 실행 모델

### Agent 상태 머신

Agent의 상태는 `src/channels/run-state-machine.ts`의 `createRunStateMachine()`으로 관리된다. 상태 머신의 핵심 변수는 두 개다:

- `activeRuns: number` — 현재 진행 중인 LLM 호출 수 (병렬 실행 지원)
- `busy: boolean` — `activeRuns > 0`과 동치; UI 상태 표시용

```
Idle (activeRuns=0, busy=false)
  └─ 메시지 도착 → Running (activeRuns=1, busy=true)
       └─ 병렬 도구 실행 → Running (activeRuns=N, busy=true)
       └─ 응답 완료 → Idle (모든 runs 완료 시)
```

상태 머신은 60초마다 heartbeat를 외부로 발행한다 (`DEFAULT_RUN_ACTIVITY_HEARTBEAT_MS = 60_000`). 이는 Gateway가 "에이전트가 살아있지만 아직 응답 중"임을 클라이언트에게 알리는 생존 신호다.

AbortSignal을 받으면 `lifecycleActive = false`로 전환하고 heartbeat 인터벌을 정리한다. 이는 프로세스 종료 시 클린업이 누락되는 것을 방지한다.

### 큐 설계 — 백프레셔 처리

에이전트가 실행 중일 때 새 메시지가 도착하면 세 가지 중 하나를 선택한다 (`src/auto-reply/reply/queue-policy.ts`):

```typescript
type ActiveRunQueueAction = "run-now" | "enqueue-followup" | "drop"

function resolveActiveRunQueueAction(params: {
  isActive: boolean;       // 현재 실행 중?
  isHeartbeat: boolean;    // heartbeat 메시지?
  shouldFollowup: boolean; // 큐에 추가할 만한 중요한 메시지?
  queueMode: "steer" | ...;
}): ActiveRunQueueAction
```

결정 로직:
- 실행 중이 아니면: `run-now`
- heartbeat이면: `drop` (중복 실행 방지)
- `shouldFollowup` 또는 `queueMode === "steer"`이면: `enqueue-followup`
- 그 외: `run-now` (현재 실행과 병렬로 즉시 실행)

`steer` 모드는 사용자가 에이전트 실행 중에 추가 지시를 보낼 때 사용한다. 현재 LLM 호출을 취소하지 않고 "방향 조정" 메시지를 대기열에 추가한다.

### Embedded vs ACP — 프로세스 격리 트레이드오프

| | Embedded | ACP |
|--|----------|-----|
| **실행 모델** | Gateway 프로세스 내에서 직접 실행 | 별도 프로세스로 격리 |
| **시작 비용** | 없음 | 프로세스 시작 오버헤드 |
| **장애 격리** | 에이전트 크래시 → Gateway 크래시 위험 | 에이전트 크래시 → ACP 재시작으로 복구 |
| **사용 시나리오** | Claude Code, 빌트인 Anthropic 에이전트 | 외부 AI CLI (사용자 커스텀 에이전트) |
| **세션 연속성** | Gateway 메모리 내 직접 | ACP 바인딩 해시로 영속적 매핑 |

**ACP 이중 세션 ID**: ACP는 두 종류의 세션 식별자를 사용한다.
- **인메모리 세션 ID**: `randomUUID()` — 단일 ACP 서버 런타임 내에서만 의미있는 임시 식별자. 최대 5,000개, 24시간 idle TTL, LRU 제거.
- **바인딩 해시**: `sha256("${channel}:${accountId}:${conversationId}").hex().slice(0,16)` — ACP 서버 재시작 후에도 동일한 대화를 같은 세션으로 식별하는 영속적 식별자.

이 이중 구조의 이유: ACP 서버 재시작 시 인메모리 세션은 사라지지만, 바인딩 해시는 `channel:accountId:conversationId` 조합이 변하지 않는 한 동일하게 재계산된다. Gateway는 바인딩 해시를 외부 키로 사용하여 재시작 후에도 대화 히스토리와 세션을 올바르게 연결한다.

### 오류 전파 경로와 복구 전략

```
LLM API 호출
  └─ 오류 발생 → FailoverError 생성 (reason 분류)
       ├─ billing (402): 과금 문제 → 다른 auth 프로파일로 전환
       ├─ rate_limit (429): 레이트 리밋 → 쿨다운 후 재시도 또는 다른 프로파일
       ├─ overloaded (503): 서버 과부하 → 동일 또는 다른 프로파일 재시도
       ├─ auth (401): 인증 만료 → 프로파일 갱신 후 재시도
       ├─ auth_permanent (403): 영구 인증 실패 → 해당 프로파일 비활성화
       ├─ model_not_found: 모델 존재하지 않음 → 폴백 모델로 전환
       └─ session_expired: 세션 만료 → 새 세션으로 재시작
```

`FailoverError.profileId` 필드는 어떤 auth 프로파일에서 오류가 발생했는지 추적하여 특정 프로파일만 선택적으로 쿨다운시킬 수 있게 한다.

---

## 5. Routing 엔진 내부

### AgentBinding 매칭 알고리즘 — 7-tier 우선순위 결정 로직

`src/routing/resolve-route.ts:723-803`에 구현된 7단계 폭포식 매칭:

```
Tier 1: binding.peer          — 특정 대화 상대(DM 파트너, 채널 ID) 매칭
Tier 2: binding.peer.parent   — 스레드의 부모 대화 매칭
Tier 3: binding.guild+roles   — 서버 ID + 역할(role) 조합 매칭
Tier 4: binding.guild         — 서버 ID만 매칭
Tier 5: binding.team          — 팀 ID 매칭
Tier 6: binding.account       — 계정(accountId) 패턴 매칭
Tier 7: binding.channel       — 채널 와일드카드 (*) 매칭
         ↓ 모두 실패
default agent                 — `resolveDefaultAgentId(cfg)` 로 폴백
```

각 Tier는 **비활성화될 수 있다** (`enabled: boolean`). 예를 들어 `guildId`가 없으면 Tier 3·4는 skip된다. 이는 불필요한 인덱스 조회를 방지한다.

**EvaluatedBindingsIndex**: 7개 Tier를 효율적으로 처리하기 위해 바인딩 목록을 사전 인덱싱한다:
- `byPeer`: peer ID → 바인딩 배열
- `byGuildWithRoles`: guildId → 역할 제약이 있는 바인딩
- `byGuild`: guildId → 역할 제약 없는 바인딩
- `byTeam`: teamId → 바인딩
- `byAccount`: accountId 패턴이 있는 바인딩 배열
- `byChannel`: 와일드카드 바인딩 배열

**WeakMap 캐싱**: 인덱스 계산은 비용이 크므로 Config 객체를 키로 하는 WeakMap에 캐싱한다 (최대 4,000개 항목). Config 객체가 GC되면 캐시도 자동 해제된다.

### Session Key 생성 원칙 — 무엇이 대화를 격리하는가

Session Key는 `agent:{agentId}:{rest}` 형식이다. `{rest}` 부분이 대화 격리의 단위를 결정한다.

**DM 4가지 scope** (`src/routing/session-key.ts:buildAgentPeerSessionKey()`):

```
dmScope = "main"
  → agent:{agentId}:main
  → 모든 DM이 하나의 공유 세션 (기본값)

dmScope = "per-peer"
  → agent:{agentId}:direct:{peerId}
  → 대화 상대별 독립 세션

dmScope = "per-channel-peer"
  → agent:{agentId}:{channel}:direct:{peerId}
  → 채널+대화상대 조합별 독립 세션

dmScope = "per-account-channel-peer"
  → agent:{agentId}:{channel}:{accountId}:direct:{peerId}
  → 계정+채널+대화상대 조합별 완전 격리
```

**그룹 채팅 세션**:
```
agent:{agentId}:{channel}:group:{peerId}
agent:{agentId}:{channel}:channel:{peerId}
```

**스레드 세션**:
```
{baseSessionKey}:thread:{normalizedThreadId}
```
스레드는 부모 세션의 `parentSessionKey`도 보유하여 컨텍스트 상속이 가능하다.

**ACP 바인딩 세션**:
```
agent:{agentId}:acp:binding:{channel}:{accountId}:{hash16}
```
hash16은 `sha256(channel:accountId:conversationId).hex().slice(0,16)`

**정체성 링크 (identityLinks)**: 동일 사용자가 여러 채널에서 다른 ID를 가질 때 (`{telegram: ["@alice"], slack: ["U12345"]}`), `resolveLinkedPeerId()`가 세션 키 생성 전에 정규화된 ID로 통합한다. 이로써 채널을 넘나드는 연속 대화가 가능해진다.

### Fallback 체인

```
binding.peer 매칭 실패
  → binding.peer.parent 매칭 시도 (스레드 부모 상속)
  → binding.guild+roles 시도
  → binding.guild 시도
  → binding.team 시도
  → binding.account 시도
  → binding.channel (* 와일드카드) 시도
  → default agent (config의 defaultAgentId)
```

폴백에서 중요한 설계 결정: **"매칭 없음"은 에러가 아니다.** 항상 기본 에이전트로 폴백하여 메시지를 버리지 않는다.

---

## 6. Tool & Function Calling 설계

### Tool Registry 구조 — 빌트인 vs 플러그인 vs 사용자 정의

```
Tool Registry
├── Built-in Tools       — bash, file read/write, web search 등 (src/agents/bash-tools.ts 등)
├── Plugin Tools         — 플러그인이 openclaw/plugin-sdk를 통해 등록
└── User-defined Tools   — 에이전트 설정(config)에서 직접 정의
```

각 도구는 TypeBox 스키마로 입력을 정의하며, Google Antigravity 호환성을 위해 `Type.Union`, `anyOf`, `oneOf`를 피하고 `stringEnum`/`optionalStringEnum`을 사용한다.

### Tool 실행 파이프라인 — LLM 응답에서 결과 주입까지

```
1. LLM이 tool_use 블록 포함 응답 반환
2. tool_use.name으로 Registry 조회
3. before-tool-call 훅 실행
   ├─ 루프 감지 (동일 도구 반복 호출 확인)
   └─ 플러그인 훅 (승인 요청, 로깅, 차단)
4. 도구 함수 실행 (sync 또는 async)
5. after-tool-call 훅 실행
6. tool_result 블록을 다음 LLM 요청의 messages에 주입
7. LLM 재호출 (도구 결과를 포함한 새 요청)
8. 최종 텍스트 응답 반환 또는 다시 tool_use → 2번으로
```

이 루프는 LLM이 도구 없는 최종 응답을 반환하거나, 최대 반복 횟수에 도달할 때까지 계속된다.

### Parallel Tool Calling — 순서 의존성 해결

Anthropic API는 단일 응답에서 여러 `tool_use` 블록을 반환할 수 있다. OpenClaw는 이를 처리할 때:

1. 독립적인 도구는 병렬로 실행 (`Promise.all`)
2. 모든 결과를 수집한 뒤 단일 요청으로 LLM에 반환
3. 순서 의존성이 있는 도구(한 도구의 출력이 다른 도구의 입력) 처리:
   - LLM이 순서를 암묵적으로 결정 (첫 번째 응답에서 독립 도구만, 두 번째 응답에서 의존 도구)
   - OpenClaw는 LLM의 순서 결정을 그대로 따름

### before-tool-call 훅 인터셉트 포인트

`src/agents/pi-tools.before-tool-call.ts`의 `wrapToolWithBeforeToolCallHook()`:

- **루프 감지**: 동일한 인자로 동일 도구가 반복 호출되는 패턴 감지 및 중단
- **플러그인 훅**: 채널 플러그인이 도구 실행 전에 개입 가능 (로깅, 사용자 승인 요청, 보안 정책 적용)
- **AbortSignal 전달**: 취소 요청이 들어오면 진행 중인 도구 실행을 중단

이 훅 구조가 없으면 "에이전트가 bash를 무한 루프로 실행하는" 상황을 방지할 수 없다.

---

## 7. Model Abstraction Layer

### 통합 LLM 인터페이스 계약 — Provider가 구현해야 하는 것

Provider는 세 가지 핵심 계약을 구현한다:

1. **스트리밍 호출**: 메시지 배열 + 도구 스키마 → AsyncIterable<StreamDelta>
2. **토큰 카운팅**: 메시지 배열 → `{ inputTokens, outputTokens, cacheReadTokens, cacheWriteTokens }`
3. **모델 메타데이터**: 컨텍스트 윈도우 크기, 최대 출력 토큰, 비용 정보

Provider 구현체 예시:
- Anthropic (직접 API)
- AWS Bedrock (Anthropic 모델)
- Google Vertex AI (Anthropic on Vertex)
- Chutes (OpenAI 호환 엔드포인트)
- BytePlus (Claude API 호환)

### Streaming 추상화 — 각 Provider 차이를 어떻게 정규화하는가

각 LLM API는 스트리밍 이벤트 구조가 다르다:
- Anthropic: `content_block_delta`, `message_delta`, `message_stop`
- OpenAI 호환: `choices[0].delta.content`, `finish_reason`
- Vertex AI: 동일하나 인증이 다름

OpenClaw는 모든 Provider를 표준 `StreamDelta` 타입으로 정규화한다:

```typescript
type StreamDelta =
  | { type: "text"; text: string }
  | { type: "tool_use"; id: string; name: string; input: unknown }
  | { type: "done"; stopReason: string; usage: TokenUsage }
```

Provider 어댑터는 원본 API 이벤트를 이 형식으로 변환한다. Agent 코드는 어떤 Provider인지 알 필요 없이 `StreamDelta` 스트림만 소비한다.

**Anthropic Vertex Stream**: `src/agents/anthropic-vertex-stream.ts`는 Vertex AI의 Anthropic 호환 스트리밍을 처리하는 별도 어댑터다. Vertex AI는 표준 Anthropic API와 인증 방식, 엔드포인트 URL, 일부 응답 필드가 다르기 때문에 분리된 구현체가 필요하다.

### 토큰 카운팅 · 비용 추적 설계

Provider 레벨에서 토큰 정보는 4가지로 분류된다:
- `inputTokens`: 입력된 텍스트/이미지 토큰
- `outputTokens`: 생성된 텍스트 토큰
- `cacheReadTokens`: 프롬프트 캐시에서 읽은 토큰 (비용 절감)
- `cacheWriteTokens`: 프롬프트 캐시에 쓴 토큰

이 분류가 필요한 이유: Anthropic의 프롬프트 캐싱은 캐시 읽기와 캐시 쓰기의 단가가 다르다. 캐시 히트율이 높은 워크로드에서 비용 추적의 정확도가 달라진다.

### 모델 폴백 체인

```
모델 X 호출 실패 (FailoverError)
  └─ reason 확인
       ├─ model_not_found → 설정된 fallbackModels 목록의 다음 모델로
       ├─ rate_limit / billing / overloaded
       │    → 다음 auth 프로파일로 전환 (프로파일 로테이션)
       │    → 프로파일 소진 시 fallbackModels로
       └─ auth_permanent → 해당 프로파일 비활성화 후 다음 프로파일
```

프로파일 로테이션(`src/agents/auth-profiles.ts`)은 "동일 모델을 여러 API 키로 사용하는" 패턴을 지원한다. 레이트 리밋에 걸렸을 때 다른 키로 즉시 전환함으로써 가용성을 높인다.

---

## 8. Context Engine & Memory 아키텍처

### Context Engine의 역할 — 컨텍스트 윈도우 관리 전략

Context Engine은 LLM 호출 직전에 실행되어 다음을 결정한다:

1. 시스템 프롬프트 조립 (에이전트 설정 + 채널 컨텍스트 + 날짜/시간)
2. 대화 히스토리에서 얼마나 많은 메시지를 포함할지
3. 메모리 검색 결과를 어디에 주입할지
4. 전체 토큰 수가 모델 컨텍스트 윈도우를 초과하면 어떻게 트리밍할지

`src/context-engine/registry.ts`는 `Symbol.for("openclaw.contextEngineRegistryState")`를 프로세스 전역 싱글턴으로 사용한다. 플러그인이 자체 Context Engine을 등록하면 기본 엔진을 대체할 수 있는 배타적 슬롯(exclusive slot) 구조다.

### 대화 히스토리 압축·트리밍 알고리즘

컨텍스트 윈도우 초과 시 전략:

1. **토큰 예산 계산**: `maxTokens - systemPromptTokens - memoryTokens - toolSchemaTokens = availableForHistory`
2. **최근 메시지 우선**: 가장 최근 메시지부터 역순으로 포함
3. **청크 단위 트리밍**: 메시지 하나를 잘라 넣는 것이 아니라, 완전한 메시지 단위로 포함/제외 결정
4. **도구 결과 압축**: 긴 tool_result는 요약 또는 잘라내기

### Memory Provider 계층 구조

```
Memory Provider 계층
├── In-process (임시)
│   └─ 세션 내 Key-Value 저장소 (프로세스 재시작 시 소멸)
└── Persistent
    ├─ SQLite + sqlite-vec (기본값)
    │   ├─ 벡터 컬럼: 코사인 유사도 검색
    │   └─ FTS5 테이블: BM25 전문 검색
    └─ LanceDB / 외부 벡터 DB (플러그인 교체 가능)
```

### 하이브리드 메모리 검색 — 벡터 + BM25 + MMR + 시간 감쇠

`src/memory/hybrid.ts`의 `mergeHybridResults()`:

```
최종 점수 = (vectorWeight × vectorScore) + (textWeight × textScore) + temporalDecay
```

**BM25 점수 정규화** (`bm25RankToScore()`):
```typescript
if (rank < 0) return -rank / (1 + (-rank))  // 음수 rank → 0~1 정규화
else return 1 / (1 + rank)                  // 양수 rank → 0~1 정규화
```

SQLite FTS5의 BM25 rank는 음수로 반환된다 (낮을수록 관련성 높음). 이를 양수 점수로 변환하는 래퍼다.

**FTS 쿼리 빌드** (`buildFtsQuery()`):
```typescript
// "hello world" → '"hello" AND "world"'
// Unicode word boundary(\p{L}\p{N}_)로 토큰화
// 각 토큰을 쌍따옴표로 감싸 exact match
// AND로 연결 (모든 토큰 포함)
```

**MMR (Maximal Marginal Relevance)** re-ranking (`src/memory/mmr.ts`):
```
score = λ × relevance - (1-λ) × max_similarity_to_already_selected
lambda = 0.7 (기본값)
```

MMR의 목적: 검색 결과에서 유사한 메모리가 반복 등장하는 것을 방지. 관련성과 다양성을 동시에 최적화한다. λ=0.7은 "관련성 70%, 다양성 30%"를 의미한다.

**시간 감쇠**: 오래된 메모리일수록 점수가 낮아진다. 최근 대화와 관련성 높은 기억이 오래된 기억보다 우선된다.

### 메모리 주입 타이밍 — 프롬프트 조립 파이프라인

```
1. 쿼리 추출: 현재 메시지에서 메모리 검색 쿼리 추출
2. 하이브리드 검색: 벡터 + BM25 검색 병렬 실행
3. 결과 병합: mergeHybridResults() → MMR re-ranking
4. 주입 위치 결정:
   - 시스템 프롬프트 하단 (Agent의 장기 기억)
   - 또는 최근 메시지 직전 (대화 관련 컨텍스트)
5. 토큰 예산 내에서 상위 N개 선택
6. 형식화하여 프롬프트에 삽입
```

### Session 범위 vs 에이전트 장기 기억 분리

| | Session-scoped Memory | Agent Long-term Memory |
|--|----------------------|----------------------|
| **생명주기** | 세션 종료 시 소멸 | 영속 (DB 저장) |
| **범위** | 현재 대화 | 에이전트 전체 |
| **사용 예** | "방금 전에 언급한 파일" | "이 사용자의 선호도" |
| **저장소** | In-process KV | SQLite (sqlite-vec) |

이 분리의 이유: 모든 대화 내용을 장기 기억으로 저장하면 메모리 검색 결과가 오염된다. 임시 세션 컨텍스트와 학습된 장기 기억을 분리함으로써 검색 정확도를 유지한다.

---

## 9. Plugin 시스템 설계 원칙

### 왜 `extensions/*`를 `src/`와 물리적으로 분리했는가

물리적 분리는 세 가지 불변 조건을 강제한다:

1. **빌드 의존성 격리**: `extensions/slack`이 빌드될 때 `src/routing/resolve-route.ts`가 변경되어도 재빌드되지 않는다.
2. **런타임 패키지 경계**: 플러그인은 독립 npm 패키지로 설치된다. `npm install --omit=dev`로 프로덕션 의존성만 설치하는 격리된 환경.
3. **SDK 계약 강제**: `src/`에 있으면 직접 임포트의 유혹이 생긴다. `extensions/`에 있으면 `openclaw/plugin-sdk/*`를 거치지 않고서는 코어 내부에 접근할 수 없다.

### 임포트 경계 규칙의 설계 의도

```
허용:  extensions/slack/src/*.ts → openclaw/plugin-sdk/slack
허용:  extensions/slack/src/*.ts → ./local-barrel.ts
금지:  extensions/slack/src/*.ts → ../../../src/routing/resolve-route.ts
금지:  extensions/slack/src/*.ts → openclaw/plugin-sdk/slack (자기 자신 임포트)
```

자기 자신을 `openclaw/plugin-sdk/slack`으로 임포트하지 않는 이유: 순환 의존성 방지 및 번들러 최적화. 플러그인 내부에서는 로컬 배럴(`runtime-api.ts`, `api.ts`)을 통해 접근하고, 외부 계약으로서의 `openclaw/plugin-sdk/slack`은 다른 패키지가 사용한다.

### Plugin 로딩 메커니즘 — jiti alias와 동적 임포트

**jiti alias 기반 로딩** (`src/plugins/sdk-alias.ts`):

```
개발 환경 (isDistRuntime=false):
  openclaw/plugin-sdk/foo → {root}/src/plugin-sdk/foo.ts

프로덕션 환경 (isDistRuntime=true):
  openclaw/plugin-sdk/foo → {root}/dist/plugin-sdk/foo.js
```

`resolvePluginSdkAliasCandidateOrder()`: 현재 모듈 경로에 `/dist/`가 포함되어 있으면 프로덕션, 아니면 개발 환경으로 판단. 6단계 상위 디렉토리를 순회하며 SDK 루트를 탐색한다.

이 설계의 핵심: 플러그인은 `openclaw/plugin-sdk/foo`를 임포트하면 되고, 실제 파일이 `src/`에 있는지 `dist/`에 있는지 알 필요가 없다. 빌드 환경 판단과 경로 해석은 alias 시스템이 담당한다.

**ESM 번들 경계 문제**: Node.js ESM 환경에서 동일 모듈이 서로 다른 번들(Gateway 코어 vs 플러그인)에 포함되면 싱글턴 인스턴스가 두 개로 분리된다. `Symbol.for()` + `globalThis` 패턴이 이를 해결한다.

```typescript
// src/shared/global-singleton.ts
export function resolveGlobalSingleton<T>(key: symbol, create: () => T): T {
  const globalStore = globalThis as Record<PropertyKey, unknown>;
  if (Object.prototype.hasOwnProperty.call(globalStore, key)) {
    return globalStore[key] as T;
  }
  const created = create();
  globalStore[key] = created;
  return created;
}
```

`Symbol.for("openclaw.pluginRegistryState")`는 프로세스 내 모든 번들에서 동일한 심볼로 해석된다. `globalThis`는 Node.js의 단일 전역 스코프를 참조하므로, 어느 번들에서 접근하든 동일한 레지스트리 인스턴스를 얻는다.

### `openclaw.plugin.json` 매니페스트 스키마

플러그인 매니페스트의 핵심 필드:

```json
{
  "id": "slack",                           // 플러그인 식별자 (extensions/<id>와 일치)
  "openclaw": {
    "install": {
      "npmSpec": "@openclaw/slack"         // npm 설치 스펙 (id와 일치)
    },
    "channel": {
      "id": "slack"                        // channel.id = plugin.id (불변 조건)
    }
  }
}
```

`openclaw.channel.id === plugin.id === extensions/<id>`의 3-방향 일치 불변 조건은 레포 인변리언트 테스트로 강제된다. 이 조건이 깨지면 플러그인 로더가 채널 ID로 플러그인을 찾을 수 없다.

### SDK 퍼블릭 서피스가 버전 안정성을 유지하는 방법

`plugin-sdk:api:check` (drift check):
- 공개 SDK 서피스(`openclaw/plugin-sdk/*`)의 타입 시그니처를 기준 스냅샷과 비교
- 변경이 감지되면 CI가 실패
- 의도적인 변경은 `pnpm plugin-sdk:api:gen`으로 스냅샷을 갱신

이 설계의 목적: 코어 내부 리팩토링이 실수로 플러그인 계약을 깨는 것을 방지한다.

---

## 10. 보안 & 신뢰 모델

### 인증 아키텍처 — Gateway 인증 모드 선택 기준

Gateway는 7가지 인증 모드를 지원한다 (`src/gateway/auth.ts`):

| 모드 | 사용 시나리오 | 신뢰 근거 |
|------|------------|---------|
| `none` | localhost 개발, 단일 사용자 신뢰 환경 | 네트워크 격리 |
| `token` | API 키 기반 클라이언트 | 공유 비밀 |
| `password` | 웹 UI 로그인 | 패스워드 해시 |
| `trusted-proxy` | Nginx/Caddy 역프록시 뒤 | IP 신뢰 + 헤더 |
| `tailscale` | Tailscale 네트워크 내 | 네트워크 레벨 인증 |
| `device-token` | 등록된 디바이스 | 디바이스 공개키 + 서명 |
| `bootstrap-token` | 초기 디바이스 등록 | 일회용 토큰 |

타이밍 공격 방지: `safeEqualSecret()`은 비교 전 양쪽 값을 SHA-256 해싱한 후 `timingSafeEqual()`로 비교한다. 직접 문자열 비교는 길이에 따라 실행 시간이 달라져 타이밍 공격에 취약하다.

**디바이스 등록 흐름** (`ConnectParams.device`):
```
device: {
  id: string        // 디바이스 고유 ID
  publicKey: string // 공개키 (서버가 검증)
  signature: string // 개인키로 서명한 nonce
  signedAt: number  // 서명 시간 (재전송 공격 방지)
  nonce: string     // 일회성 값
}
```

### 채널 레벨 격리 — 크로스 채널 데이터 유출 방지 설계

각 채널 계정은 독립적인 `accountId`를 가진다. 세션 키에 `accountId`가 포함되어 (`per-account-channel-peer` 모드) Slack 계정 A의 대화가 Slack 계정 B의 세션에 접근할 수 없다.

`senderIsOwner` 플래그: 발신자가 Gateway 운영자(소유자)인지 여부는 OAuth 스코프 `operator.admin`의 존재로 결정된다. 이 플래그가 없는 사용자는 에이전트 설정 변경, 다른 사용자의 세션 조회 등의 특권 작업이 거부된다.

### Tool 실행 샌드박싱 경계

Bash 도구(`src/agents/bash-tools.ts`)는 세 가지 실행 컨텍스트를 지원한다:
- **Node.js 직접 실행** (`bash-tools.exec-host-node.ts`): 프로세스 격리 없음
- **Gateway 호스트 실행** (`bash-tools.exec-host-gateway.ts`): Gateway 프로세스 컨텍스트
- **PTY 실행** (`bash-tools.exec.pty.ts`): 터미널 에뮬레이션, 인터랙티브 명령 지원

`bash-tools.exec-approval-request.ts`: ACP 에이전트가 bash 명령을 실행하기 전 사용자 승인을 요청한다. 이것이 "샌드박싱"의 실제 구현이다 — 기술적 격리가 아닌 *사람 루프(human-in-the-loop)* 승인 게이트.

### Credential 스토리지 설계

자격증명은 `~/.openclaw/credentials/`에 파일로 저장된다. 채널별, 계정별로 분리된 파일 구조로 격리한다. OAuth 토큰 갱신은 채널 플러그인이 담당하며, Gateway는 현재 유효한 자격증명의 상태만 추적한다.

---

## 11. Slack 연동 — 아키텍처 관점

### Channel Plugin 계약 — inbound/outbound 인터페이스

`extensions/slack/src/channel.ts`는 `createChatChannelPlugin<ResolvedSlackAccount, SlackProbe>()` 팩토리를 사용한다. 이 팩토리가 정의하는 계약:

**인바운드 인터페이스**:
```
Slack 이벤트 → 파싱 → Message 표준 객체
  └─ message_type: "direct" | "group" | "channel" | "thread"
  └─ peerId: Slack 사용자 ID / 채널 ID
  └─ guildId: Slack 워크스페이스 ID
  └─ accountId: 봇 토큰 계정 ID
```

**아웃바운드 인터페이스**:
```
Reply 표준 객체 → Slack 메시지 포맷 변환 → Slack API 호출
  └─ 텍스트 → Block Kit 또는 plain text
  └─ 파일 첨부 → files.upload API
  └─ 스레드 답변 → thread_ts 파라미터
```

**self-import 금지 패턴**: `extensions/slack/src/runtime-api.ts` 배럴은 슬랙 플러그인 내부에서 SDK 기능에 접근하는 창구다. `openclaw/plugin-sdk/slack`을 직접 임포트하면 자기 자신을 임포트하는 순환 구조가 되므로, 항상 로컬 배럴을 거친다.

### Slack OAuth 플로우가 Plugin 시스템에 통합되는 방식

```
1. `openclaw channels connect slack` 명령
2. Gateway → Slack OAuth 인증 URL 생성
3. 사용자 브라우저에서 Slack OAuth 승인
4. Slack → 설정된 redirect_uri로 code 전달
5. Gateway → code를 access_token으로 교환
6. access_token → `~/.openclaw/credentials/slack/{accountId}` 저장
7. 채널 플러그인 재초기화 (새 토큰으로)
8. Slack RTM/Events API 웹소켓 연결 수립
```

플러그인 시스템과의 통합 포인트: OAuth 콜백은 Gateway HTTP 엔드포인트가 받고, 토큰 저장과 플러그인 재초기화는 Gateway가 Slack 플러그인의 `connect()` 메서드를 호출하는 방식으로 처리된다.

### 멀티 워크스페이스 라우팅 설계

단일 OpenClaw 인스턴스가 여러 Slack 워크스페이스에 동시 연결되는 경우:

```
워크스페이스 A (accountId=ws-a)
  └─ 메시지 → resolve-route(channel="slack", accountId="ws-a", guildId="T111", ...)
        → AgentBinding Tier 4 (guild) 또는 Tier 6 (account) 매칭

워크스페이스 B (accountId=ws-b)
  └─ 메시지 → resolve-route(channel="slack", accountId="ws-b", guildId="T222", ...)
        → 다른 AgentBinding 매칭 가능
```

`accountId`가 라우팅 결정에 포함되므로 같은 Slack 채널 이름이더라도 다른 워크스페이스면 다른 에이전트로 라우팅할 수 있다. 세션 키도 `accountId`를 포함하여 워크스페이스 간 대화 히스토리가 섞이지 않는다.

---

## 12. AI Company 레퍼런스 아키텍처

### 역할별 Agent 분리 토폴로지

OpenClaw는 단일 에이전트 패턴뿐 아니라 **역할 기반 다중 에이전트 토폴로지**를 지원한다. AgentBinding의 7-tier 매칭을 활용하여 동일 채널에서 역할에 따라 다른 에이전트를 호출한다.

예시: AI 회사 내부 Slack 봇 구성

```yaml
# config.yaml 예시
agents:
  eng-assistant:
    model: claude-sonnet-4
    systemPrompt: "엔지니어링 팀 지원 에이전트..."
    tools: [bash, file-read, github]

  hr-assistant:
    model: claude-haiku-4
    systemPrompt: "HR 질문 응답 에이전트..."
    tools: []

  executive-briefing:
    model: claude-opus-4
    systemPrompt: "임원 보고용 심층 분석..."
    tools: [web-search, memory]

bindings:
  - agentId: eng-assistant
    channel: slack
    guildId: "T_WORKSPACE"
    roles: ["engineer", "tech-lead"]

  - agentId: hr-assistant
    channel: slack
    guildId: "T_WORKSPACE"
    roles: ["hr"]

  - agentId: executive-briefing
    channel: slack
    guildId: "T_WORKSPACE"
    roles: ["executive"]

  - agentId: eng-assistant   # 기본 폴백
    channel: slack
    accountId: "*"
```

이 구성에서 `@engineer` 역할을 가진 사람이 메시지를 보내면 Tier 3 (guild+roles)에서 `eng-assistant`가 매칭된다. `@hr` 역할이면 `hr-assistant`, 매칭 없으면 기본 `eng-assistant`로 폴백된다.

### Cron 기반 자율 에이전트 패턴

OpenClaw는 외부 메시지 없이도 스케줄에 따라 에이전트를 실행하는 Cron 패턴을 지원한다.

```yaml
crons:
  - agentId: daily-briefing
    schedule: "0 9 * * 1-5"  # 평일 오전 9시
    message: "오늘의 이슈와 PR 요약을 작성해"
    sessionKey: "agent:daily-briefing:cron-session"
```

Cron 세션은 `isCronSessionKey()` 함수로 식별된다 (`src/routing/session-key.ts`). 일반 대화 세션과 구별하여 메모리 관리 및 컨텍스트 트리밍 정책을 달리 적용할 수 있다.

**자율 에이전트 활용 예**:
- 매일 아침 GitHub 이슈 요약 → Slack 채널 게시
- 매시간 모니터링 지표 확인 → 이상 감지 시 알림
- 주간 회고 보고서 자동 작성

### Webhook 외부 트리거 통합

GitHub Actions, PagerDuty, 커스텀 워크플로우에서 OpenClaw Agent를 직접 트리거할 수 있다.

```bash
# GitHub Actions에서 PR 리뷰 요청
curl -X POST https://gateway.company.com/webhook/agent \
  -H "Authorization: Bearer $OPENCLAW_TOKEN" \
  -d '{
    "agentId": "code-reviewer",
    "message": "PR #1234 리뷰해줘",
    "sessionKey": "pr-1234"
  }'
```

Gateway는 이 요청을 내부적으로 `agent.send` RPC와 동일하게 처리한다. `sessionKey`를 고정하면 여러 webhook 호출이 같은 대화 컨텍스트를 공유한다 (예: PR에 대한 여러 리뷰 코멘트가 같은 세션).

**ACP를 통한 Claude Code 연동**: 회사 개발 환경에서 Claude Code CLI를 ACP 에이전트로 Gateway에 연결하면, Slack DM으로 코드 작업을 지시하고 결과를 Slack으로 받는 패턴이 가능하다.

```
개발자 Slack DM → Gateway → ACP binding (claudecode:account:conversation) → Claude Code CLI
                                                                              ↓
개발자 Slack DM ← Gateway ← ACP 응답 스트림 ←──────────────────────────── 실행 결과
```

---

## 부록: 핵심 설계 결정 요약

| 결정 | 선택 | 대안 | 이유 |
|------|------|------|------|
| RPC 전송 | WebSocket | REST/SSE | 양방향 스트리밍 + 단일 연결 이벤트 |
| 라우팅 캐시 | WeakMap | TTL 캐시 | Config 객체 GC로 자동 무효화 |
| 플러그인 격리 | npm 패키지 | 단일 레포 파일 | 독립 배포, 의존성 격리 |
| 메모리 검색 | 벡터 + BM25 + MMR | 벡터만 | 키워드 검색 보완 + 다양성 |
| ACP 세션 ID | randomUUID + 바인딩 해시 | 단일 ID | 재시작 후 연속성 + 메모리 효율 |
| 전역 싱글턴 | Symbol.for + globalThis | 싱글톤 모듈 | ESM 번들 경계 교차 |
| 타이밍 공격 방지 | SHA-256 + timingSafeEqual | 직접 비교 | 일정 시간 비교 보장 |
| 프로파일 로테이션 | 다중 API 키 | 단일 키 재시도 | 레이트 리밋 고가용성 |

---

*이 문서는 `main` 브랜치 코드베이스를 직접 분석하여 작성되었습니다. 인용된 모든 코드 경로는 실제 파일 위치입니다.*
