# 🎙️ OnMeet 발표 대본 (전체 21 슬라이드) — 수정본

---

## 📌 Slide 1 — Home (표지)

안녕하세요. 저희가 만든 **OnMeet**을 소개하겠습니다.

OnMeet은 한 문장으로 말하면, **"AI가 회의를 기록하고 요약하는 B2B 화상회의 플랫폼"**입니다.

실시간 화상회의 중에 참가자들의 음성을 자동으로 인식하고, 회의가 끝나면 AI가 키워드, 결정사항, 액션아이템까지 구조화된 회의록으로 만들어주는 서비스입니다. 단순히 "Zoom 클론"이 아니라, 회의 이후의 문서 작업까지 자동화한다는 점이 핵심 차별점입니다.

기술적으로는 **7개의 마이크로서비스**, **5개 프로그래밍 언어**, 그리고 **VAD → STT → LLM으로 이어지는 3단계 AI 파이프라인**으로 구성되어 있습니다. 크게 4가지 핵심 축이 있는데요 — WebRTC 기반 실시간 화상회의, AI 회의록 자동 생성, SSE와 FCM을 활용한 실시간 알림, 그리고 RSA/AES 기반의 기업급 보안 아키텍처입니다.

그럼 먼저 저희 팀을 소개하겠습니다.

---

## 📌 Slide 2 — Team: 팀 구성 및 역할

OnMeet을 만든 4명의 팀원 소개입니다.

- **최승은** — 팀장 / Full-Stack. 프로젝트 총괄, Video Service 개발, 프론트엔드 전반.
- **박영진** — Backend Lead. MSA 설계, Auth/Gateway/Noti 개발, CI/CD, Go 전환.
- **조예성** — Backend. Notification Service, Flyway 마이그레이션, 인프라 안정화.
- **양진영** — AI Backend. AI Service 전담, STT/LLM 파이프라인 구축.

그럼 아키텍처부터 하나씩 살펴보겠습니다.

---

## 📌 Slide 3 — System Architecture

이 슬라이드는 OnMeet의 **전체 시스템 아키텍처**입니다.

클라이언트 요청의 흐름을 따라가면, **React + Vite** 클라이언트 → **Nginx** 리버스 프록시(SSL 종단, 정적 에셋, 도메인 라우팅) → **Gateway**(Kotlin + WebFlux, JWKS 기반 JWT 검증 및 Cookie 인증)를 거칩니다.

게이트웨이를 통과한 요청은 도메인별로 6개의 마이크로서비스로 분배됩니다.

- **auth-service** (Kotlin) — 인증/인가 전담. MySQL + Redis(Refresh Token, 세션)
- **video-service** (Java) — LiveKit SFU 연동, 회의방 관리, 오디오 녹화
- **ai-service** (Java) — VAD/STT/LLM 파이프라인 전체 처리
- **notification-service** (Java) — SSE + FCM 이중 채널 실시간 알림
- **file-service** (Go) — PostgreSQL + MinIO. Java 대비 응답속도 100배, 메모리 17배 절감을 달성한 서비스입니다.
- **email-service** (Java) — Gmail OAuth2 기반 이메일 발송

하단의 공통 인프라는 **Kafka**(이벤트 버스), **Redis**(세션/캐시), **MinIO**(S3 호환 스토리지), **MySQL/PostgreSQL**(서비스별 DB)입니다.

설계 원칙 하나를 강조하면, **"서비스별 독립 데이터베이스"** 원칙을 철저히 지켰습니다. 서비스 간 통신은 반드시 Kafka 이벤트 또는 API Gateway를 경유하도록 해서, MSA의 핵심인 **느슨한 결합(Loose Coupling)**을 확보했습니다.

---

## 📌 Slide 4 — Key Features

OnMeet의 세 가지 핵심 기능입니다.

**첫 번째, 실시간 화상회의.** WebRTC SFU 아키텍처 기반으로, N:N 회의에서 Mesh 방식의 대역폭 폭발 문제 없이 저지연 중계가 가능합니다. 갤러리/스피커 뷰, 채팅, 화면 공유, 대기실, PIP 모드를 지원하고, **Kafka Egress 오디오 레코딩**이 AI 파이프라인의 입력 소스가 됩니다.

**두 번째, AI 회의록 자동 생성.** 이게 OnMeet의 핵심 차별 기능입니다. Silero VAD로 무음 필터링 → OpenAI STT로 텍스트 변환 → Claude로 구조화 요약이라는 3단계 파이프라인을 거칩니다. 결정사항과 액션아이템까지 자동 추출되고, 사용자는 BlockNote 에디터에서 바로 편집할 수 있습니다. 각 단계의 기술적 세부사항은 다음 슬라이드에서 상세히 다루겠습니다.

**세 번째, 기업급 보안과 알림.** RSA-2048 JWT 서명, AES-256-GCM 개인키 암호화, PBKDF2 60만 회 반복 KDF로 다층 보안을 구현했고, SSE + FCM 이중 채널로 웹과 모바일 모두 실시간 알림을 제공합니다. 보안 아키텍처 역시 뒤의 전용 슬라이드에서 자세히 보여드리겠습니다.

---

## 📌 Slide 5 — AI Pipeline

OnMeet의 심장부, **AI 파이프라인 전체 흐름도**입니다. 6단계로 구성됩니다.

**01 Audio Capture** — LiveKit Track Egress가 각 참여자의 마이크 트랙을 OGG/Opus로 캡처해서 S3에 업로드합니다. 핵심은 혼합 음성이 아닌 **참여자별 개별 트랙**을 분리 저장한다는 것입니다. 혼합 음성은 아무리 좋은 STT를 써도 "누가 말했는지" 구분이 불가능하기 때문입니다.

**02 Kafka Event** — 참여자 퇴장 시 `audio-chunk-ready` 토픽에 이벤트가 발행되고, video-service는 즉시 본업으로 복귀합니다. **Fire & Forget** 패턴으로 비동기 파이프라인이 시작됩니다.

**03 VAD** — ai-service가 **Silero VAD v4 ONNX**를 30ms 슬라이딩 윈도우로 돌려 무음 구간을 탈락시킵니다. 개별 트랙 분리 시 "말하지 않는 참가자의 침묵"까지 STT에 보내면 과금이 N배로 폭발하는데, 이 필터링으로 **STT 비용을 최대 70% 이상 방어**했습니다. ONNX Runtime이 백엔드 로컬에서 돌아가므로 **추가 비용 0원**입니다.

**04 STT** — VAD가 솎아낸 유효 음성만 **gpt-4o-mini-transcribe**에 전송합니다. 초기 whisper-1 대비 한국어 인식률이 압도적으로 향상되었고, 처리 속도와 비용 모두 개선되었습니다.

**05 AI Summary** — 음성 세그먼트와 채팅 텍스트를 시간순으로 병합한 뒤 **Claude 3.5 Sonnet**에 던집니다. 선택 이유는 거대한 컨텍스트 윈도우와, **순수 JSON 포맷을 변형 없이 출력**하는 지시 수행력입니다. 100만 토큰당 약 $3 수준의 비용 효율성도 선택의 이유였습니다.

**06 Output** — 구조화 요약이 저장되고, SSE + FCM으로 회의록 완성을 알립니다.

하단의 Kafka Topics Flow: `audio-chunk-ready → voice-segment → meeting-summary-ready → notification-send`. 평균 응답 지연 **3초 미만**입니다.

---

## 📌 Slide 6 — Tech Stack

사용 기술 전체 목록입니다.

**Backend** — Spring Boot 3.3.5 기반. 서비스 특성에 따라 언어를 선택했습니다. 비동기 I/O가 중요한 auth/gateway는 **Kotlin + WebFlux**, 기존 에코시스템 활용이 중요한 ai/video/notification은 **Java**, 극한의 성능이 필요한 file-service는 **Go**입니다.

**Frontend** — React 18 + TypeScript strict + Vite 7. Zustand(클라이언트 상태), TanStack Query(서버 상태), Tailwind CSS, Cloudflare Pages 배포.

**AI & Media** — OpenAI STT, Claude API, Silero VAD ONNX, LiveKit SFU, WebRTC.

**Infrastructure** — GCP e2-standard-4(4vCPU/16GB), Docker + Compose, GitHub Actions CI/CD, Kafka 7.5.0, Redis 7, MySQL 9.0, PostgreSQL 16, MinIO, Nginx, Cloudflare DNS.

언어 비율은 Java 40%, Kotlin 30%, Go 15%, TypeScript 15%로, **서비스 요구사항에 맞는 최적의 도구를 선택**했다는 점을 강조드립니다.

---

## 📌 Slide 7 — Security Architecture

**다층 보안 아키텍처**, Defense in Depth 전략입니다. 실제 코드는 뒤의 Code 섹션(Slide 10~11)에서 보여드리고, 여기서는 전체 설계를 짚겠습니다.

**Authentication** — RSA-2048 키 페어 생성, 개인키를 AES-256-GCM으로 암호화 저장, PBKDF2 60만 회 반복 KDF, RS256 JWT 서명, JWKS 엔드포인트 공개키 배포, Refresh Token Rotation(Redis 7일 TTL, 1회 사용 후 폐기).

**Gateway Security** — HttpOnly Cookie → Bearer Token 변환, JWKS JWT 검증, X-User-* 헤더 주입(다운스트림 서비스는 JWT 재파싱 불필요), X-Gateway-Secret 내부 인증, Rate Limiting 중앙 관리.

**Data Security** — MIME 스푸핑 방지(바이너리 검사), S3 Key 인젝션 방어, Path Traversal 방어, Semaphore 업로드 제한, Template Traversal 방어.

하단의 **Request Lifecycle**: Browser Cookie → Gateway JWT Validate → Headers Inject → Service Auth Check → X-Gateway-Secret → Response. 이 체인이 모든 요청에 적용됩니다.

---

## 📌 Slide 8 — Frontend Architecture

프론트엔드 아키텍처입니다. 여기서는 **구조와 기술 선택 결과**를 중심으로 설명드리고, 각각의 고민과 의사결정 과정은 제 회고 슬라이드에서 상세히 다루겠습니다.

**Feature-Sliced Design** — app / features(auth·meeting·ai·notification·schedule·team) / shared / pages 구조. 도메인 간 의존성이 명확하고 파일 탐색이 직관적입니다.

**디자인 패턴** — VAC 패턴으로 597줄 → 225줄 축소, Custom Hooks 추출(useMeetingDevices, useMicTest 등), Radix UI 기반 Compound Components 적용.

**Service Fetch Factory** — 4개 MSA와의 통신 로직을 `createServiceFetch()` 팩토리 하나로 통합. Content-Type 자동 설정, X-User-Id 주입, 401 자동 갱신까지 한 줄로 처리됩니다.

**커스텀 SSE 파서** — EventSource의 커스텀 헤더 미지원 한계를 fetch + ReadableStream으로 직접 구현해서 해결. 알림·대기실·STT 세 가지 독립 SSE 관리, 5초 자동 재연결.

**Phase 상태 머신** — Preparing → Joining → Waiting → Connected → Disconnected 5단계 모델링.

**성능 최적화** — Route-level lazy loading, Manual chunk splitting(LiveKit 478KB, TipTap 375KB 등 독립 청크), Route prefetching(사이드바 hover), React.memo/useShallow, 프로덕션 console 자동 제거.

**보안** — CSP 화이트리스트, HttpOnly 쿠키, Promise 싱글턴 토큰 갱신, DOMPurify, 권한별 Route Guard. **PWA** — Precaching, NetworkFirst, 오프라인 폴백, Service Worker 자동 갱신.

---

## 📌 Slide 9 — Infrastructure & DevOps

GCP + Docker + GitHub Actions 기반 CI/CD 파이프라인입니다.

**CI/CD Pipeline**: feature branch → PR/develop → `release/v0.x.0` push → GitHub Actions(변경 서비스 감지) → Jib(Dockerfile 없이 이미지 빌드) → Docker Hub → GCP VM `docker compose up`.

**배포 전략** — Jib 레이어 캐싱으로 CI 시간 단축, 변경 서비스만 선택적 빌드, Semantic Versioning(0.x.y).

**네트워크** — Cloudflare DNS Proxy(DDoS/SSL), `api.onmeet.cloud` → Nginx → Gateway, `rtc.onmeet.cloud` → LiveKit :7880, `/files/` → MinIO :9000.

**인프라** — GCP e2-standard-4(4vCPU/16GB), 서울 리전, Docker Engine 29.3.0.

**Git Workflow** — main / develop / feat/ONMEET-XX / hotfix/v0.x.y 4종 브랜치 전략.

**Version History**: v0.1.0 MVP → v0.2.0 AI → v0.3.0 Go 전환 → v0.4.0 API 정합성 → v0.4.8 현재(Hotfix).

---

## 📌 Slide 10 — Code: 인증 서비스의 암호화 전략

여기서부터 **Code 섹션**입니다. 앞서 보안 아키텍처 슬라이드에서 전체 설계를 보여드렸는데, 여기서는 **실제 구현 코드**를 보시겠습니다.

**왼쪽 KeyManager.kt** — RSA 개인키를 AES-256-GCM으로 암호화하는 코드입니다. PBKDF2 60만 회 반복으로 마스터 키를 유도하고, SecureRandom으로 16바이트 Salt + 12바이트 IV를 생성합니다. 최종 포맷은 `Salt(16) + IV(12) + CipherText`로, 매번 다른 Salt/IV가 생성되므로 동일 평문이라도 암호문이 달라집니다.

**오른쪽 JwtTokenProvider.kt** — RS256 JWT 서명 코드입니다. `keyManager.getPrivateKey()` 호출 시 AES-GCM 복호화가 투명하게 수행되어, 개인키가 메모리에서만 존재합니다.

핵심은 **비밀키를 서비스 간에 공유하지 않는 분산 인증 구조**입니다. 공개키만 JWKS로 노출하고, Gateway의 NimbusDecoder가 이를 가져다 검증합니다.

---

## 📌 Slide 11 — Code: Gateway 보안 관문

앞서 보안 슬라이드에서 소개한 Gateway의 **실제 필터 체인 코드**입니다.

**CookieConverter** — HttpOnly Cookie의 accessToken을 Spring Security BearerToken으로 변환. XSS로 `document.cookie` 접근이 불가능하므로 토큰 탈취를 원천 차단합니다.

**UserHeaderFilter** — JWT claims에서 userId/email/roles를 추출해 X-User-* 헤더로 주입. 다운스트림 서비스는 JWT를 재파싱할 필요 없이 헤더만 신뢰합니다.

**SecureInternalFilter** — X-Gateway-Secret을 자동 주입해서, 외부에서 내부 서비스 포트로 직접 접근하는 우회 공격을 차단합니다.

하단 **Request Flow**: Browser Cookie → CookieConverter → NimbusJwtDecoder(JWKS) → UserHeaderFilter → SecureInternalFilter → 응답.

---

## 📌 Slide 12 — Code: 화상회의 녹화 자동화 파이프라인

AI 파이프라인 슬라이드에서 설명드린 Audio Capture 단계의 **실제 구현 코드**입니다.

**WebhookController** — LiveKit의 `TRACK_PUBLISHED` Webhook을 감지해 참여자별 마이크 트랙 Egress를 **자동 시작**합니다. 수동 조작 없이 입장 즉시 녹화가 시작되고, `PARTICIPANT_LEFT` 시 Kafka에 `audio-chunk-ready`를 발행합니다.

**RecordingService** — LiveKit TrackEgress API로 마이크 트랙을 S3에 OGG 저장. activeEgress Map으로 진행 중인 녹음을 추적합니다.

**LiveKitClient** — LiveKit API가 요구하는 HS256 JWT를 외부 라이브러리 없이 **Raw HMAC-SHA256으로 직접 생성**해서 의존성을 최소화했습니다.

하단 **Recording Flow**: LiveKit Webhook → TrackPublished → startTrackEgress() → S3 OGG → ParticipantLeft → Kafka: audio-chunk-ready. 이 이벤트가 다음 슬라이드의 AI 파이프라인 입력이 됩니다.

---

## 📌 Slide 13 — Code: 음성 → 텍스트 → 회의 요약 AI 파이프라인

AI 파이프라인의 **3대 핵심 모델 구현 코드**입니다.

**SileroVadClient (ONNX)** — ONNX Runtime으로 오디오 배열의 음성 확률을 추론합니다. 임계값 이상이면 유의미한 발화로 판정하고, 연속 침묵이면 유음 덩어리를 잘라냅니다. **PCM 오디오 정규화**로 마이크 볼륨 차이와 관계없이 작은 속삭임까지 누락 없이 분석합니다.

**SttWorkerService (OpenAI STT)** — VAD가 잘라준 세그먼트 중 100ms 미만 노이즈는 스킵하고, 유의미한 세그먼트만 gpt-4o-mini-transcribe에 전송합니다.

**ClaudeSummarizer (LLM)** — "순수 JSON 객체만 출력하라"는 프롬프트와 정확한 스키마를 제시하고, 반환된 JSON을 `objectMapper.readValue()`로 역직렬화해서 **스키마 불일치 시 즉시 예외를 발생**시킵니다. 이 검증을 통과해야만 DB에 적재됩니다.

하단 플로우: Kafka Consumer → S3 다운로드 → VAD 필터링 → STT 변환 → Claude 요약 → file-service 저장 + SSE.

---

## 📌 Slide 14 — Code: 웹 + 모바일 실시간 알림 이중 채널

notification-service의 **SSE + FCM 이중 채널 구현**입니다.

**SSE Multi-tab** — `Map<Long, Map<String, SseEmitter>>` 이중 Map으로 사용자별 다중 탭/디바이스를 독립 관리. 30초 heartbeat로 dead emitter 자동 정리.

**FCM Dispatch** — Auth 서비스 토큰과 로컬 DB 토큰을 Set으로 합쳐 중복 없이 멀티캐스트. 실패 시 3회 지수 백오프 재시도, 무효 토큰 자동 삭제.

**Notification Toggle** — 회의/팀/회의록 3가지 카테고리 개별 ON/OFF. 토글 OFF면 발송 자체를 차단하는 구조.

하단 Kafka Consumer Topics: meeting-start, meeting-end, ai-summary-ready, invitation, temporary-password.

---

## 📌 Slide 15 — Code: 왜 Go를 썼는지 + 보안까지 잡은 파일 서버

file-service **Go 전환의 기술 선택과 결과**입니다.

**보안** — `http.DetectContentType()`으로 MIME 스푸핑 방지, ownerType whitelist + 숫자 ID 검증으로 S3 Key 인젝션 방어.

**성능** — `chan struct{}{}` Semaphore로 동시 업로드 10개 제한, 1MB 기준 캐시/스트리밍 분기.

**결과 수치**:
- 응답 속도: Kotlin ~450ms → **Go ~4ms**
- 메모리: JVM ~512MB → **Go ~30MB**
- Docker 이미지: 300MB → **20MB**
- 동시성: Thread pool → **goroutines**

파일 I/O 바운드 작업에 JVM의 무거운 부트스트랩이 불필요했고, Go의 경량성이 이 유스케이스에 완벽히 맞았습니다.

---

## 📌 Slide 16 — Code: 안전한 이메일 발송 + OAuth2 토큰 관리

email-service의 **보안 + 토큰 효율성** 코드입니다.

**Template 보안** — `ALLOWED_TEMPLATES` Set으로 허용 템플릿만 화이트리스트. Path Traversal 공격을 원천 차단하고, Thymeleaf Context로 변수 안전 바인딩.

**토큰 캐시** — volatile + synchronized **double-checked locking** 구현. 핵심 디테일은 TTL을 실제 만료 60분이 아닌 **55분으로 설정**한 것입니다. 5분 마진이 만료 시점과 갱신 요청이 겹치는 레이스 컨디션을 방지합니다.

---

## 📌 Slide 17 — 회고: 최승은 회고

앞서 프론트엔드 아키텍처와 기술 선택 결과를 보여드렸는데, 이번에는 그 뒤에 있었던 **고민과 시행착오**를 말씀드리겠습니다.

### 🧠 상태 관리 — "어디에 담을 것인가" (1분 30초)

가장 오래 고민한 문제입니다. 화상회의 플랫폼은 한 화면에서 카메라 on/off, 마이크 상태, 참여자 입퇴장, 채팅, AI 실시간 전사가 동시에 벌어집니다.

처음에는 모든 걸 useState로 처리했습니다. 그러다 AI STT 데이터가 초당 수십 번 업데이트되면서 **리렌더 폭풍**을 맞았습니다. React가 매 프레임마다 리렌더를 돌리니까 UI가 완전히 버벅이더라고요.

이 문제를 겪고 나서야, **상태의 성격에 따라 저장소를 분리해야 한다**는 걸 체감했습니다. STT처럼 고빈도 데이터는 useRef 버퍼에 쌓아두고, 200~300ms Throttle로 끊어서 렌더링합니다. "debounce 걸면 되지 않나?" 싶을 수 있는데, debounce는 마지막 입력 후 일정 시간이 지나야 반영되므로 실시간 자막에서는 사용자가 "멈춘 것처럼" 느낍니다. Throttle이 맞습니다.

SSE 스트림 데이터도 고민이었습니다. 스트림이 살아있는 동안은 로컬 버퍼가 진실의 원천이고, 스트림이 종료되면 `invalidateQueries`로 서버 데이터로 전환합니다. 이걸 **"Buffer → Throttle → Snapshot"** 패턴이라 부르기로 했습니다. 처음부터 이 분류가 있었던 건 아니고, 문제를 직접 맞은 다음에야 정리된 결론입니다.

### 📡 SSE 파서 — "표준이 안 되면 직접 만든다" (1분)

자부심을 느끼는 부분입니다. 브라우저 EventSource API가 커스텀 헤더를 지원하지 않아서, fetch + ReadableStream으로 SSE 프로토콜을 직접 파싱하는 훅을 만들었습니다.

가장 어려웠던 건 **청크 경계 문제**였습니다. TCP 패킷 경계와 SSE 메시지 경계가 일치하지 않아서, 메시지가 반만 들어올 수 있습니다. 이걸 모르고 바로 파싱하면 JSON.parse가 터집니다. 버퍼에 쌓아두고 `\n\n` 구분자가 나올 때까지 기다리는 로직을 넣어서 해결했고, 5초 자동 재연결과 AbortController 클린업까지 포함시켰습니다. 이 경험이 나중에 대용량 파일 다운로드의 ReadableStream 진행률 추적에도 그대로 응용되었습니다.

### 🏗 디자인 패턴 — "문제가 있을 때 꺼내 쓰는 도구" (1분)

앞서 결과 수치를 보여드렸는데, 그 이면의 고민을 말씀드리면 — 패턴을 공부하면 "전부 다 적용해야 할 것 같은" 유혹이 옵니다. 하지만 **패턴은 문제가 있을 때 꺼내 쓰는 도구**이지, 미리 심어두는 장식이 아닙니다.

VAC 패턴은 CompanyManagement가 597줄까지 불어나서 하나를 고치려면 파일 전체를 읽어야 했을 때 도입했습니다. Custom Hooks도 MeetingPreparationModal이 435줄이 되어서야 추출했고요. **"3번 반복되기 전에는 추상화하지 않는다"**는 원칙을 지키려고 노력했습니다. 각 페이지의 로딩 스켈레톤 같은 건 비슷해 보여도 각 레이아웃에 맞춰져 있어서 의도적으로 통합하지 않았습니다.

### ⚡ 빌드 & 인증 (30초)

인증에서 가장 까다로웠던 건 **동시 다발적 401**이었습니다. 페이지 로드 시 3개 API가 동시에 401을 반환하면 refresh도 3번 나가서 race condition이 생깁니다. 첫 번째 401의 refresh Promise를 공유하는 **Promise 싱글턴 패턴**으로 해결했습니다. 이 문제는 직접 겪기 전까지는 "왜 이런 걸 고민해야 하지?" 싶은 종류의 문제입니다.

### 🌱 성장 (50초)

25개 파일에 253개 테스트 케이스를 운영하면서, API 레이어 테스트가 가장 가성비가 좋고, 커스텀 훅은 수동 테스트만으로 edge case 검증이 어렵다는 걸 배웠습니다.

이 프로젝트에서 가장 크게 성장한 부분은 **"왜 이렇게 하는가"에 답하는 능력**입니다. 처음에는 "좋다고 해서" 도구를 골랐지만, 프로젝트가 복잡해지면서 도구를 선택하는 것보다 **도구의 경계를 정하는 것**이 더 중요하다는 걸 깨달았습니다. auth/api.ts가 471줄까지 불어난 다음에야 분리의 필요성을 체감했고, 리렌더 폭풍을 맞은 다음에야 useRef 버퍼링의 가치를 이해했습니다. **문제를 직접 겪어야 해결책의 의미를 진짜로 알게 됩니다.** 그게 이 프로젝트의 가장 큰 수확입니다.

---

## 📌 Slide 18 — 회고: 박영진 회고

박영진 Backend Lead의 회고입니다. 앞서 아키텍처와 코드로 결과를 보여드렸는데, 여기서는 **의사결정 과정과 장애 경험**을 중심으로 말씀드립니다.

**MSA 설계의 고민** — 처음부터 MSA였던 건 아닙니다. 모놀리스로 시작했다가 빌드 시간이 점점 길어지고, AI 모듈 변경이 인증 서비스 재배포를 유발하는 문제가 터졌습니다. 7개 서비스로 분리하면서 Kafka 이벤트 버스를 도입했고, 그 결과 서비스별로 최적의 기술(Kotlin/Java/Go)을 독립적으로 선택할 수 있게 된 것이 MSA의 실질적 이점이었습니다.

**인증 설계의 고민** — JWKS를 선택한 핵심 이유는 **비밀키를 서비스 간에 공유하지 않기 위해서**입니다. 대칭키 기반이면 모든 서비스가 비밀키를 가져야 하고, 하나라도 뚫리면 전체가 위험해집니다. 비대칭키 + JWKS로 공개키만 배포하는 구조가 MSA에 맞습니다.

**Go 전환의 고민** — 앞서 응답 속도 100배 개선이라는 결과를 보여드렸는데, 전환 결정이 쉽지는 않았습니다. 팀원 대부분이 Java/Kotlin에 익숙한 상황에서 새 언어를 도입하는 건 리스크입니다. 하지만 file-service의 특성(I/O 바운드, 간단한 비즈니스 로직, 높은 동시성 요구)이 Go의 강점과 정확히 맞아떨어졌기에 결정했습니다.

**장애 대응의 교훈** — 가장 아픈 경험은 **"CORS≠CORS"** 교훈입니다. 프론트에서 CORS 에러가 터지면 보통 CORS 설정을 의심하는데, 실제로는 Nginx DNS 캐시 문제, DB 커넥션 풀 고갈, email 서비스 3중 장애 등 완전히 다른 원인이 **CORS 에러로 포장되어** 나타났습니다. 표면적 증상과 근본 원인이 다를 수 있다는 걸 몸으로 체득했습니다.

**CI/CD** — 510커밋 17회 릴리즈를 무중단으로 배포한 경험. 변경 서비스만 빌드하는 선택적 빌드와 Jib 레이어 캐싱이 결정적이었습니다.

---

## 📌 Slide 19 — 회고: 조예성 회고

조예성 Backend의 회고입니다. 앞서 알림 시스템의 기술 구현을 보여드렸는데, 여기서는 **직면한 문제들과 해결 과정**을 말씀드립니다.

**SSE 멀티탭의 고민** — 가장 당황스러웠던 건 "알림이 안 온다"는 리포트였습니다. 원인을 추적해보니, 사용자가 새 탭을 열면 단일 Map에서 기존 연결이 덮어써져서 **이전 탭의 알림이 유실**되고 있었습니다. "Map 하나면 되지"라는 단순한 가정이 깨진 순간이었고, 이중 Map 구조로 전환하면서 30초 heartbeat로 dead emitter를 정리하는 로직도 추가했습니다.

**FCM 푸시의 고민** — 토큰 관리가 예상보다 복잡했습니다. Auth 서비스의 최신 토큰과 로컬 DB 토큰이 불일치하는 경우가 있어서, Set으로 합쳐 중복을 제거하는 방식을 택했습니다. 발송 실패 시 3회 지수 백오프 재시도를 하고, 무효 토큰은 자동 삭제해서 쓸데없는 재시도를 줄였습니다.

**장애 대응** — 500/503 간헐 에러가 발생했는데, 원인이 CORS SecurityConfig와 RestTemplate 타임아웃(3s/5s)이 겹쳐서 스레드가 대기 상태에 빠지는 것이었습니다. 앞서 박영진이 말한 "CORS≠CORS"와 같은 맥락인데, 저도 에러 메시지만 보고 SecurityConfig만 만지다가 실제로는 타임아웃 조정이 답이었습니다.

**운영 자동화** — 10분 주기 예약 발송, 15일 지난 알림 새벽 3시 자동 정리, Kafka DLT 에러 라우팅으로 무인 운영 시스템을 구축했습니다. "사람이 안 봐도 돌아가는 시스템"을 만드는 것의 가치를 체감한 부분입니다.

---

## 📌 Slide 20 — 회고: 양진영 회고

양진영 AI Backend의 회고입니다. 앞서 AI 파이프라인의 구조와 코드를 보여드렸는데, 여기서는 **3번의 아키텍처 전면 수정**을 거쳐야 했던 고민 과정을 말씀드립니다.

**파이프라인 진화의 고민** — 처음에는 "가장 쉬운 방법"인 혼합 음성 통짜 STT를 시도했습니다. 트래픽도 적고 구현도 단순했지만, 결과물을 보는 순간 **"누가 이 말을 했는지 알 수 없다"**는 치명적 한계에 부딪혔습니다. 화자 분리 없는 회의록은 의미가 없었기에, 참여자별 개별 트랙 분리 구독으로 전면 수정했습니다.

**비용 방어의 고민** — 개별 트랙을 분리하니까 이번에는 **"침묵 과금 폭발"**이라는 새로운 문제가 터졌습니다. 6명 회의에서 2명만 말하고 있으면, 나머지 4명의 빈 오디오까지 전부 STT API로 전송되어 과금이 6배로 늘어났습니다. 이 문제를 해결하기 위해 VAD를 도입했고, 어디에 배치할지도 고민이었습니다. video-service에 넣으면 미디어 처리와 딥러닝 연산이 뒤섞여 메모리 과부하 우려가 있었기 때문에, **MSA 관심사 분리 원칙에 따라 ai-service 입구에 배치**했습니다. 이게 3번의 전면 수정 끝에 도달한 최종 아키텍처입니다.

**모델 선정의 고민** — whisper-1로 시작했지만 한국어 인식률이 아쉬웠습니다. gpt-4o-mini-transcribe로 교체한 건 단순한 업그레이드가 아니라, 테스트 비용을 감수하면서 한국어 품질을 검증한 결과였습니다. Claude도 GPT-4o와 비교 테스트를 거쳤고, JSON 포맷 준수율에서 Claude가 월등했습니다.

**포용성 설계의 고민** — 도서관이나 오픈 스페이스처럼 마이크를 켤 수 없는 참가자도 있습니다. 이들의 채팅이 회의록에서 소외되면 안 됩니다. 음성(STT)과 채팅이라는 완전히 다른 매체를 하나의 대화 흐름으로 합치기 위해 **Redis ZSET**을 선택했습니다. 타임스탬프를 score로 사용해서 삽입 즉시 시간순 정렬이 보장되는 구조입니다. 기술적으로도 의미 있지만, **"모든 참가자의 의견이 요약에 반영된다"**는 제품 철학적 가치도 큽니다.

**무중단 전환의 고민** — S3에서 DB로 저장소를 바꿀 때, 이미 S3에 저장된 기존 회의록이 깨지면 안 됩니다. DB를 먼저 조회하되, 없으면 S3 파싱으로 Fallback하는 이중화 체계를 심어서 **배포 시점의 100% 하위 호환성**을 확보했습니다. 라이브 운영 중에 저장소를 교체하는 건 매우 위험한 작업인데, Fallback 덕분에 무사고로 전환할 수 있었습니다.

---

## 📌 Slide 21 — Thank You / Q&A

마지막 슬라이드입니다.

OnMeet은 **7개 마이크로서비스, 5개 언어, 4개 데이터베이스, 1개 AI 파이프라인**으로 구성된 B2B 화상회의 플랫폼입니다.

저희가 가장 자부하는 세 가지:

첫째, **지갑을 지킨 아키텍처**. 로컬 VAD ONNX와 PCM 정규화로 STT 비용을 70% 이상 방어.

둘째, **다중 LLM 앙상블**. VAD(최단 딜레이) + gpt-4o-mini(오디오 특화) + Claude(NLP 구조화) — 각 단계에 가장 뾰족한 도구를 조립.

셋째, **기술부채 없는 마이그레이션**. S3 → DB 전환을 Fallback으로 무중단 완료.

GitHub 리포지토리, Swagger API 문서, 발표자료 링크는 QR코드로 확인하실 수 있습니다.

발표를 경청해주셔서 감사합니다. 질문 있으시면 말씀해주세요.
