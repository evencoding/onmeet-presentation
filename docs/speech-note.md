# 🎙️ OnMeet 발표 대본 (전체 21 슬라이드)

## 📌 Slide 1 — Home (표지)

안녕하세요. 저희가 만든 **OnMeet**을 소개하겠습니다.

OnMeet은 한 문장으로 말하면, <b>"AI가 회의를 기록하고 요약하는 B2B 화상회의 플랫폼"</b>입니다.

실시간 화상회의 중에 참가자들의 음성을 자동으로 인식하고, 회의가 끝나면 AI가 키워드, 결정사항, 액션아이템까지 구조화된 회의록으로 만들어주는 서비스입니다. 단순히 "Zoom 클론"이 아니라, 회의 이후의 문서 작업까지 자동화한다는 점이 핵심 차별점입니다.

기술적으로는 **7개의 마이크로서비스**, **5개 프로그래밍 언어**, 그리고 **VAD → STT → LLM으로 이어지는 3단계 AI 파이프라인**으로 구성되어 있습니다. 크게 4가지 핵심 축이 있는데요 — WebRTC 기반 실시간 화상회의, AI 회의록 자동 생성, SSE와 FCM을 활용한 실시간 알림, 그리고 RSA/AES 기반의 기업급 보안 아키텍처입니다.

그럼 아키텍처부터 하나씩 살펴보겠습니다.

---

## 📌 Slide 2 — System Architecture

이 슬라이드는 OnMeet의 **전체 시스템 아키텍처**입니다.

클라이언트 요청이 들어오는 흐름부터 설명드리면, **React + Vite** 클라이언트에서 출발한 요청은 먼저 **Nginx 리버스 프록시**를 거칩니다. Nginx는 SSL 종단점 역할과 함께 정적 에셋 서빙, 그리고 도메인별 라우팅을 처리합니다. 그 다음 **Gateway 서비스**로 도달하는데, 이 게이트웨이가 Kotlin과 Spring WebFlux로 구현되어 있고, 모든 요청에 대해 **JWKS 기반 JWT 검증**과 **Cookie 인증**을 수행하는 보안 관문 역할을 합니다.

게이트웨이를 통과한 요청은 도메인별로 6개의 마이크로서비스로 분배됩니다.

- **auth-service** — Kotlin으로 작성. 인증/인가 전담. MySQL과 Redis를 사용합니다. Redis는 Refresh Token 저장과 세션 관리용입니다.
- **video-service** — Java로 작성. 화상회의의 핵심인 LiveKit SFU 서버와 연동하며, 회의방 생성/종료, 참여자 관리, 오디오 녹화를 담당합니다.
- **ai-service** — Java로 작성. VAD, STT, LLM 파이프라인 전체를 처리하는 AI 전용 서비스입니다.
- **notification-service** — Java. SSE와 FCM 이중 채널로 실시간 알림을 제공합니다.
- **file-service** — 이건 **Go**로 작성했습니다. 뒤에서 자세히 설명드리겠지만, Java 대비 응답속도 100배, 메모리 사용량 17배 절감이라는 극적인 성능 개선을 달성한 서비스입니다. PostgreSQL과 MinIO(S3 호환 스토리지)를 사용합니다.
- **email-service** — Java. Gmail OAuth2 기반 이메일 발송 전담 서비스입니다.

그리고 하단의 공통 인프라 레이어로 **Kafka**가 이벤트 버스 역할, **Redis**가 세션/캐시, **MinIO**가 S3 호환 오브젝트 스토리지, **MySQL/PostgreSQL**이 서비스별 데이터 저장소 역할을 수행합니다.

여기서 설계 원칙을 하나 강조하면, **"서비스별 독립 데이터베이스"** 원칙을 철저히 지켰습니다. auth는 auth의 MySQL만, file은 PostgreSQL만 접근합니다. 서비스 간 통신은 반드시 Kafka 이벤트 또는 API Gateway를 경유하도록 해서, MSA의 핵심인 **느슨한 결합(Loose Coupling)**을 확보했습니다.

---

## 📌 Slide 3 — Key Features

OnMeet의 세 가지 핵심 기능을 정리한 슬라이드입니다.

**첫 번째, 실시간 화상회의**입니다. video-service가 Java와 LiveKit SDK로 구현했고, WebRTC SFU 아키텍처를 기반으로 합니다. SFU를 쓴 이유는 N:N 화상회의에서 Mesh 방식은 참여자가 늘어날수록 대역폭이 기하급수적으로 증가하기 때문입니다. SFU는 서버가 중계해주면서도 P2P에 가까운 저지연을 유지할 수 있어요. 갤러리 뷰와 스피커 뷰 전환, 채팅, 화면 공유, 대기실, PIP 모드까지 지원하고, 특히 **Kafka Egress를 통한 오디오 레코딩**이 AI 파이프라인의 입력 소스가 됩니다.

**두 번째, AI 회의록 자동 생성**입니다. ai-service에서 처리하며, 이게 OnMeet의 가장 핵심적인 차별 기능입니다. Silero VAD v4 ONNX로 무음을 필터링하고, OpenAI의 gpt-4o-mini-transcribe로 음성을 텍스트로 변환한 뒤, Claude API가 구조화된 JSON 요약을 생성합니다. 결정사항과 액션아이템까지 자동으로 추출하고, 사용자는 BlockNote 리치 텍스트 에디터에서 회의록을 바로 편집할 수 있습니다. 요약이 완성되면 SSE로 실시간 알림이 갑니다.

**세 번째, 기업급 보안과 알림 시스템**입니다. auth, gateway, notification 세 서비스가 Kotlin으로 구현한 보안 스택인데요. RSA-2048 키 페어로 JWT를 서명하고, 개인키 자체를 AES-256-GCM으로 암호화해서 저장합니다. 키 유도 함수로 PBKDF2를 60만 회 반복하는데, 이건 브루트포스 공격을 사실상 불가능하게 만드는 수치입니다. 알림은 SSE와 FCM 이중 채널로 웹과 모바일 모두 커버하고, Kafka 이벤트 기반으로 비동기 발송합니다.

하단에 보시면 이 기능들을 뒷받침하는 기술 태그들이 나열되어 있습니다 — LiveKit SFU WebRTC, gpt-4o-mini-transcribe, Claude AI 요약, Kafka 이벤트 버스, SSE+FCM 실시간 알림, RSA+AES 보안 스택. 각각이 어떻게 맞물리는지는 다음 슬라이드부터 구체적으로 보여드리겠습니다.

---

## 📌 Slide 4 — AI Pipeline

이 슬라이드가 OnMeet의 심장부, **AI 파이프라인 전체 흐름도**입니다. 6단계로 구성됩니다.

**01 Audio Capture** — 화상회의 중 LiveKit Track Egress가 각 참여자의 마이크 트랙을 OGG/Opus 포맷으로 캡처해서 S3(MinIO)에 업로드합니다. 여기서 핵심은 "혼합 음성이 아닌 참여자별 개별 트랙"을 분리 저장한다는 겁니다. 왜 이렇게 했는지는 백서에서 자세히 다뤘는데, 혼합 음성을 STT에 넣으면 "누가 말했는지" 구분이 불가능합니다. 완벽한 화자 분리를 위해 개별 트랙 분리가 필수적이었습니다.

**02 Kafka Event** — 참여자가 회의를 떠나면 `audio-chunk-ready` 토픽에 이벤트가 발행됩니다. 참여자 정보와 청크 메타데이터가 포함되어 있고, 이 시점부터 비동기 파이프라인이 시작됩니다. video-service는 이벤트를 던지고 즉시 본업(화상회의 라우팅)으로 복귀합니다. **Fire & Forget** 패턴이죠.

**03 VAD (Voice Activity Detection)** — ai-service가 Kafka에서 이벤트를 소비하면, 가장 먼저 **Silero VAD v4 ONNX** 모델을 돌립니다. 30ms 슬라이딩 윈도우로 오디오를 분석해서 무음 구간을 탈락시킵니다. 이게 왜 중요하냐면, 개별 트랙을 분리 저장하면 "2명이 말할 때 나머지 3~4명의 침묵"까지 전부 STT API로 전송되면서 과금이 N배로 폭발합니다. VAD 필터링으로 **STT 클라우드 비용을 최대 70% 이상 방어**했습니다. 게다가 VAD는 백엔드의 내부 ONNX Runtime에서 돌아가므로 **추가 클라우드 비용이 0원**입니다.

**04 STT** — VAD가 솎아낸 유효 음성만 **gpt-4o-mini-transcribe**에 전송합니다. 초기에는 whisper-1을 사용했는데, 한국어 인식률과 타임스탬프 정확성에서 아쉬움이 있었습니다. gpt-4o-mini-transcribe로 교체한 뒤 **한국어 인식률이 압도적으로 향상**되었고, 처리 속도도 더 빠르면서 비용도 절감되는, 말 그대로 가성비 끝판왕 모델입니다. 변환된 텍스트는 `voice-segment` 토픽으로 발행됩니다.

**05 AI Summary** — 모든 음성 세그먼트가 수집되면, 여기에 채팅 텍스트까지 시간순으로 병합한 뒤 **Claude 3.5 Sonnet API**에 던집니다. Claude를 선택한 이유는 두 가지입니다. 첫째, 1~2시간짜리 대본도 소화할 수 있는 거대한 컨텍스트 윈도우. 둘째, 프롬프트 엔지니어링으로 **순수 JSON 포맷을 변형 없이 완벽하게 출력**하는 지시 수행력. DB 스키마와 1:1로 매핑되는 JSON을 뽑아내야 하는 저희 요구사항에 Claude가 타 모델 대비 월등했습니다. 100만 토큰당 약 $3 수준의 비용 효율성도 선택의 이유였고요.

**06 Output** — 생성된 구조화 요약이 file-service에 저장되고, SSE 실시간 알림과 FCM 푸시로 참여자들에게 회의록 완성을 알립니다.

하단의 **Kafka Topics Flow**를 보시면, `audio-chunk-ready → voice-segment → meeting-summary-ready → notification-send`로 이어지는 이벤트 체인이 전체 파이프라인의 뼈대입니다. 평균 응답 지연은 **3초 미만**입니다.

---

## 📌 Slide 5 — Tech Stack

사용 기술 전체 목록입니다. 카테고리별로 간략히 짚겠습니다.

**Backend**는 Spring Boot 3.3.5 기반이고, 언어는 서비스 특성에 따라 선택했습니다. auth와 gateway처럼 비동기 I/O가 중요한 서비스는 **Kotlin + Spring WebFlux**, AI/Video/Notification처럼 기존 에코시스템 활용이 중요한 서비스는 **Java**, 그리고 극한의 성능이 필요한 file-service는 **Go**로 작성했습니다. JPA와 Flyway로 DB 마이그레이션을 관리합니다.

**Frontend**는 **React 18 + TypeScript strict 모드 + Vite 7**입니다. 상태 관리는 Zustand v5, 서버 상태는 TanStack Query v5, 라우팅은 React Router v6, 스타일링은 Tailwind CSS로 구성했고, Cloudflare Pages에 배포합니다.

**AI & Media**는 앞서 설명드린 OpenAI STT, Claude API, Silero VAD ONNX, 그리고 LiveKit SFU와 WebRTC입니다. 이메일 발송에는 Gmail OAuth2와 Thymeleaf 템플릿 엔진을 사용합니다.

**Infrastructure**는 GCP e2-standard-4 인스턴스(4vCPU, 16GB) 위에 Docker + Compose로 전 서비스를 컨테이너화했고, GitHub Actions로 CI/CD를 자동화합니다. 메시지 큐는 Kafka 7.5.0, 캐시는 Redis 7, DB는 MySQL 9.0과 PostgreSQL 16, 오브젝트 스토리지는 MinIO, 리버스 프록시는 Nginx, DNS는 Cloudflare입니다.

하단의 언어 비율을 보시면 Java 40%, Kotlin 30%, Go 15%, TypeScript 15%로, **단일 언어에 종속되지 않고 각 서비스의 요구사항에 맞는 최적의 도구를 선택**했다는 점을 강조드립니다.

---

## 📌 Slide 6 — Security Architecture

OnMeet의 **다층 보안 아키텍처**, Defense in Depth 전략입니다.

**왼쪽, Authentication** — auth-service에서 RSA-2048 키 페어를 생성하는데, 여기서 개인키를 평문으로 저장하면 안 되겠죠. **AES-256-GCM**으로 개인키 자체를 암호화합니다. 키 유도에는 **PBKDF2를 60만 회 반복**해서 브루트포스를 사실상 차단합니다. Salt, IV, CipherText를 하나의 바이트 배열로 조립해서 저장하고요. 이 개인키로 RS256 JWT를 서명하고, 공개키는 JWKS 엔드포인트(`/auth/.well-known/jwks.json`)로 노출합니다. Refresh Token은 Redis에 7일 TTL로 저장하며, **Rotation** 정책을 적용해서 한 번 사용된 리프레시 토큰은 즉시 폐기됩니다.

**가운데, Gateway Security** — 모든 요청의 관문입니다. 브라우저의 HttpOnly Cookie에서 accessToken을 꺼내 Bearer Token으로 변환하고(CookieServerAuthenticationConverter), JWKS를 통해 JWT를 검증합니다(NimbusReactiveJwtDecoder). 검증이 통과되면 JWT claims에서 userId, email, roles를 추출해 X-User-* 헤더로 주입합니다. 다운스트림 서비스들은 JWT 파싱 없이 이 헤더만 신뢰하면 됩니다. 또한 **X-Gateway-Secret** 헤더를 자동 주입해서, 내부 서비스가 "이 요청이 정말 Gateway를 거쳐왔는지" 검증할 수 있습니다. Rate Limiting도 Spring Cloud Gateway에서 중앙 관리합니다.

**오른쪽, Data Security** — file-service의 Go 코드에서 MIME 스푸핑 방지(실제 바이너리 검사), S3 Key 인젝션 방어(ownerType whitelist + 숫자 ID 검증), Path Traversal 방어(../ 체크), Semaphore 기반 동시 업로드 제한(최대 10개), 그리고 email-service의 Template Traversal 방어까지 구현했습니다.

하단의 **Request Lifecycle** 플로우를 보시면, Browser Cookie → Gateway JWT Validate → Headers Inject → Service Auth Check → X-Gateway-Secret → Response로 이어지는 전체 보안 체인이 한눈에 보입니다.

---

## 📌 Slide 7 — Frontend Architecture

프론트엔드 아키텍처입니다. React 18, TypeScript strict 모드, Vite 7, Tailwind CSS 기반이고, 크게 6개 영역으로 나뉩니다.

**Feature-Sliced Design** — 프로젝트 구조를 app(글로벌 설정), features(도메인별: auth, meeting, ai, notification, schedule, team), shared(공용 유틸/훅/컴포넌트), pages(라우트별 페이지)로 나눴습니다. 대규모 프로젝트에서 파일 찾기가 직관적이고, 도메인 간 의존성이 명확해집니다.

**디자인 패턴** 측면에서, **VAC 패턴**(View-Asset-Component)을 적용해서 로직 컨테이너와 순수 UI를 분리했습니다. 이걸로 **597줄짜리 컴포넌트를 225줄로** 축소한 사례가 있습니다. Custom Hooks로 useMeetingDevices, useMicTest, useWaitingRoomSSE 등을 추출했고, Radix UI 기반 Compound Components 패턴도 활용했습니다.

**Service Fetch Factory** — 4개 MSA 서비스와 통신하는 fetch 로직의 중복을 팩토리 패턴으로 통합했습니다. `createServiceFetch("/video/v1")` 한 줄로 Content-Type 자동 설정, X-User-Id 주입, 401 자동 갱신까지 처리됩니다.

**커스텀 SSE 파서** — 브라우저의 EventSource API는 커스텀 헤더를 지원하지 않아서, **fetch + ReadableStream으로 직접 구현**했습니다. 실시간 알림은 React Query 캐시 무효화와 연동하고, 대기실 승인/거절은 Phase 상태 전이로, 실시간 STT는 useRef 버퍼 + flush 최적화로 처리합니다. 연결 끊김 시 **5초 자동 재연결** 로직도 포함입니다.

**Phase 상태 머신** — 회의 참여 프로세스를 Preparing → Joining → Waiting → Connected → Disconnected 다섯 단계로 모델링했습니다. 각 Phase에서 카메라/마이크 열거, 실시간 프리뷰, 호스트 퇴장 감지 등이 상태에 따라 자동 처리됩니다.

**성능 최적화** — Route-level lazy loading으로 10개 이상의 페이지를 동적 로딩하고, LiveKit·TipTap·Firebase·Recharts를 Manual chunk splitting으로 독립 청크화했습니다. 사이드바 hover 시 다음 페이지를 Route prefetching으로 미리 로드하고, React.memo와 useShallow로 회의 컴포넌트의 리렌더를 최소화했으며, 프로덕션 빌드에서 esbuild.drop으로 console/debugger를 자동 제거합니다.

**보안** 측면에서 CSP를 default-src 'self' 화이트리스트로 적용하고, HttpOnly 쿠키 인증으로 XSS 토큰 탈취를 원천 차단합니다. 401 응답 시 Promise 싱글턴으로 토큰 자동 갱신하며, DOMPurify로 XSS를 방지하고, ProtectedRoute/GuestRoute/ManagerRoute로 접근 권한을 분리했습니다. PWA도 지원해서 정적 에셋 Precaching, API NetworkFirst 캐싱, 오프라인 폴백, Service Worker 자동 갱신까지 구현했습니다.

---

## 📌 Slide 8 — Infrastructure & DevOps

GCP + Docker + GitHub Actions 기반 CI/CD 파이프라인입니다.

상단의 **CI/CD Pipeline** 흐름을 보시면, 개발자가 feature branch에서 작업 → PR을 통해 develop 브랜치에 머지 → `release/v0.x.0` 브랜치를 push하면 CI/CD가 트리거됩니다. GitHub Actions가 **변경된 서비스만 감지**해서 선택적으로 빌드하고, **Jib**으로 Dockerfile 없이 Docker 이미지를 빌드합니다. 이미지는 Docker Hub에 `:v0.x.0`과 `:latest` 태그로 push되고, GCP VM에서 `docker compose up`으로 배포됩니다.

**배포 전략** — Jib을 쓰면 Dockerfile 작성 없이도 최적화된 레이어 캐싱이 가능하고, 변경된 서비스만 선택적으로 빌드하니까 CI 시간이 대폭 단축됩니다. Semantic Versioning(0.x.y)을 적용해서 버전 관리를 체계화했습니다.

**네트워크** — Cloudflare DNS Proxy를 통해 DDoS 방어와 SSL을 처리하고, `api.onmeet.cloud`는 Nginx를 거쳐 Gateway로, `rtc.onmeet.cloud`는 LiveKit 포트 7880으로, `/files/` 경로는 MinIO 포트 9000으로 라우팅됩니다.

**인프라** — GCP e2-standard-4 인스턴스(4vCPU, 16GB RAM)를 서울 리전(Asia-northeast3)에 배치했고, Docker Engine 29.3.0과 Docker Compose v5.1.1을 사용합니다.

**Git Workflow** — main(프로덕션 안정), develop(통합 개발), feat/ONMEET-XX(기능 개발), hotfix/v0.x.y(긴급 수정) 네 가지 브랜치 전략을 운용합니다.

하단의 **Version History**를 보시면, v0.1.0 MVP → v0.2.0 AI 기능 → v0.3.0 파일 관리(Go 전환) → v0.4.0 API 통합 정합성 → v0.4.8 현재(Hotfix)까지, 점진적으로 기능을 확장하면서 안정성을 확보해온 과정이 보입니다.

---

## 📌 Slide 9 — Code: 인증 서비스의 암호화 전략

여기서부터 **Code 섹션**으로, 실제 구현 코드를 보여드립니다. 첫 번째는 auth-service의 암호화 전략입니다.

**왼쪽 KeyManager.kt** — RSA 키를 AES-256-GCM으로 암호화해서 저장하는 코드입니다. PBKDF2를 60만 번 반복하는 KDF로 마스터 키를 유도하고, 16바이트 Salt와 12바이트 IV를 SecureRandom으로 생성합니다. GCM 인증 태그 길이는 128비트. 최종 저장 포맷은 `Salt(16) + IV(12) + CipherText`로, 하나의 바이트 배열에 담깁니다. 이 구조 덕분에 복호화 시 Salt/IV 분리가 깔끔하고, 매번 다른 Salt/IV가 생성되므로 동일 평문이라도 암호문이 달라집니다.

**오른쪽 JwtTokenProvider.kt** — RS256으로 JWT를 서명하는 코드입니다. email을 subject로, userId와 roles를 claims에 담고, accessToken은 30분 TTL로 발행합니다. `keyManager.getPrivateKey()`가 호출되면 위의 AES-GCM 복호화가 투명하게 수행되어 개인키가 메모리에서만 존재합니다.

핵심 포인트를 정리하면, PBKDF2 60만 반복으로 브루트포스를 방어하고, Salt+IV+CipherText 구조로 안전하게 저장하며, RS256 서명으로 JWT를 발행하고, JWKS 엔드포인트로 공개키를 배포해서 Gateway의 NimbusDecoder가 검증합니다. **비밀키를 서비스 간에 공유하지 않는 분산 인증 구조**라는 점이 핵심입니다.

---

## 📌 Slide 10 — Code: Gateway 보안 관문

모든 요청이 반드시 거치는 Gateway의 보안 필터 체인입니다.

**CookieConverter** — 브라우저에서 HttpOnly Cookie에 담긴 accessToken을 Spring Security의 BearerToken 객체로 변환합니다. 왜 Cookie를 쓰느냐면, XSS 공격으로 JavaScript에서 토큰을 탈취하는 걸 원천 차단하기 위해서입니다. Cookie는 HttpOnly 속성 덕분에 `document.cookie`로 접근이 불가능합니다.

**UserHeaderFilter** — JWT claims에서 userId, email, roles를 추출해서 `X-User-Id`, `X-User-Email`, `X-User-Roles` 헤더로 요청에 주입합니다. 다운스트림 서비스들은 JWT를 다시 파싱할 필요 없이 헤더만 읽으면 됩니다. 인증 로직이 Gateway에 집중되고, 각 서비스는 비즈니스 로직에만 집중할 수 있습니다.

**SecureInternalFilter** — Gateway가 X-Gateway-Secret 헤더를 자동 주입합니다. 내부 서비스는 이 헤더의 존재 여부로 "이 요청이 진짜 Gateway를 통과한 건지"를 검증합니다. 외부에서 직접 내부 서비스 포트로 접근하는 우회 공격을 차단합니다.

하단의 **Request Flow**를 보시면, Browser Cookie → CookieConverter → BearerToken → NimbusJwtDecoder (JWKS 검증) → UserHeaderFilter → SecureInternalFilter → 성공 응답. 이 체인이 모든 요청마다 실행됩니다.

---

## 📌 Slide 11 — Code: 화상회의 녹화 자동화 파이프라인

video-service가 LiveKit과 연동해서 **참여자별 오디오를 자동 녹화**하는 파이프라인입니다.

**WebhookController** — LiveKit 서버에서 Webhook이 오면, `TRACK_PUBLISHED` 이벤트를 감지해서 참여자별 마이크 트랙의 Egress를 자동 시작합니다. 참여자가 회의에서 퇴장(`PARTICIPANT_LEFT`)하면 Kafka에 `audio-chunk-ready` 이벤트를 발행합니다. 핵심은 **참여자가 입장할 때마다 자동으로 개별 녹화가 시작**된다는 겁니다. 수동 조작이 필요 없습니다.

**RecordingService** — LiveKit의 TrackEgress API를 호출해서 마이크 트랙을 S3에 OGG 포맷으로 저장합니다. activeEgress Map으로 현재 진행 중인 녹음을 추적 관리하고, 참여자 퇴장 시 해당 Egress를 종료합니다.

**LiveKitClient** — LiveKit API는 HS256 JWT를 요구하는데, 외부 라이브러리 없이 **Raw HMAC-SHA256으로 직접 생성**했습니다. base64Url 인코딩, HMAC 서명까지 순수 Java 코드로 구현해서 의존성을 최소화했습니다.

하단의 **Recording Flow**: LiveKit Webhook → TrackPublished → startTrackEgress() → S3 OGG 저장 → ParticipantLeft → Kafka: audio-chunk-ready로 이벤트가 흘러갑니다. 이 이벤트가 다음 슬라이드의 AI 파이프라인 입력이 됩니다.

---

## 📌 Slide 12 — Code: 음성 → 텍스트 → 회의 요약 AI 파이프라인

이제 AI 파이프라인의 **실제 코드 구현체** 3가지입니다.

**왼쪽 SileroVadClient (ONNX)** — 과금 방어의 첫 번째 관문입니다. ONNX Runtime으로 Silero VAD v4 모델의 딥러닝 추론을 수행합니다. 오디오 배열을 OnnxTensor로 변환하고 state 벡터와 함께 모델에 입력하면, 각 윈도우의 음성 확률(probability)이 출력됩니다. 임계값 이상이면 유의미한 발화 구간으로 판정하고, 일정 시간 이상 연속 침묵하면 이전 유음 덩어리를 잘라냅니다. 추가로 **PCM 오디오 정규화**를 적용해서 마이크 볼륨이 달라도 작은 속삭임까지 누락 없이 분석합니다. 이 모든 게 **백엔드 메모리에서 로컬로 동작하므로 클라우드 비용 0원**입니다.

**가운데 SttWorkerService (OpenAI STT)** — VAD가 잘라준 유효 음성 세그먼트들을 순회하면서 gpt-4o-mini-transcribe에 전송합니다. 100ms 미만의 초단 노이즈는 스킵하고, 의미 있는 길이의 세그먼트만 STT API를 호출합니다. whisper-1에서 교체한 뒤 한국어 인식률이 비약적으로 향상되었고, 화자 식별과 타임스탬프 정확성도 우수합니다.

**오른쪽 ClaudeSummarizer (LLM)** — 프롬프트 엔지니어링의 핵심입니다. "순수 JSON 객체만 출력하라"는 강력한 지시와 함께, 정확한 스키마(description, keywords, decisions, actionItems)를 제시합니다. Claude가 반환한 JSON을 `objectMapper.readValue(text, SummaryResult.class)`로 역직렬화해서, **스키마와 맞지 않으면 즉시 파싱 예외가 발생**하도록 설계했습니다. 이 검증을 통과해야만 DB에 구조화된 데이터로 적재될 자격이 주어집니다.

하단 플로우: Kafka Consumer → S3 오디오 다운로드 → Silero VAD 필터링 → OpenAI STT 변환 → Claude 요약 생성 → file-service 저장 + SSE 알림.

---

## 📌 Slide 13 — Code: 웹 + 모바일 실시간 알림 이중 채널

notification-service의 **SSE + FCM 이중 채널 알림 시스템**입니다.

**SSE Subscribe (Multi-tab)** — 가장 까다로운 부분이 **멀티탭 지원**이었습니다. 초기에는 userId → SseEmitter 단일 Map으로 관리했는데, 사용자가 여러 탭을 열면 새 탭의 연결이 이전 탭을 덮어써서 알림이 유실됐습니다. 이걸 해결하기 위해 `Map<Long, Map<String, SseEmitter>>` **이중 Map 구조**로 전환했습니다. userId → Map<emitterId, SseEmitter>로, emitterId는 `userId + "_" + UUID`로 생성합니다. 여러 탭/디바이스의 동시 연결을 독립적으로 관리하고, 30초 heartbeat로 dead emitter를 자동 정리합니다.

**FCM Dispatch** — 모바일 기기를 위한 Firebase Cloud Messaging 알림입니다. Auth 서비스에서 관리하는 최신 토큰과 로컬 DB의 토큰을 Set으로 합쳐 중복 없이 멀티캐스트로 발송합니다. 실패 시 **최대 3회 지수 백오프 재시도**를 하고, 무효 토큰은 자동 삭제합니다. WEB, AOS, iOS 모든 플랫폼을 커버합니다.

**Notification Toggle** — 사용자별로 회의/팀/회의록 3가지 카테고리를 개별 ON/OFF할 수 있습니다. 발송 로직에서 **토글이 OFF면 발송 자체를 차단**하는 구조로, 불필요한 알림이 전혀 가지 않습니다.

하단의 **Kafka Consumer Topics**를 보시면, meeting-start, meeting-end, ai-summary-ready, invitation, temporary-password 5가지 이벤트를 소비합니다.

---

## 📌 Slide 14 — Code: 왜 Go를 썼는지 + 보안까지 잡은 파일 서버

file-service를 **Go로 전환한 이유와 그 결과**입니다.

**왼쪽 보안 코드** — MIME 스푸핑 방지를 위해 `http.DetectContentType()`으로 파일의 실제 바이너리를 검사합니다. 확장자를 `.jpg`로 바꿔서 업로드해도, 실제 바이너리가 이미지가 아니면 차단됩니다. S3 Key 인젝션 방어는 ownerType을 whitelist로 제한하고, ownerId를 숫자 정규식으로 검증해서, 악의적인 경로 주입을 원천 차단합니다.

**오른쪽 성능 코드** — `chan struct{}{}`로 Semaphore를 구현해서 동시 업로드를 최대 10개 goroutine으로 제한합니다. Go의 채널을 세마포어로 활용한 관용적 패턴입니다. 스마트 캐싱도 적용해서 1MB 미만 파일은 메모리 캐시, 1MB 이상은 스트리밍으로 처리합니다.

하단의 **Why Go?** 수치가 인상적입니다:
- 응답 속도: Kotlin ~450ms → **Go ~4ms** (100배 이상 개선)
- 메모리: JVM ~512MB → **Go ~30MB** (17배 절감)
- 동시성: Thread pool → **goroutines** (경량 동시성)

파일 업로드/다운로드는 I/O 바운드 작업이 대부분이라 JVM의 무거운 부트스트랩이 불필요했고, Go의 경량성과 네이티브 바이너리 컴파일이 이 유스케이스에 완벽히 맞았습니다. Docker 이미지 크기도 300MB에서 20MB로 줄었습니다.

---

## 📌 Slide 15 — Code: 안전한 이메일 발송 + OAuth2 토큰 관리

email-service의 **보안 + 토큰 효율성** 코드입니다.

**왼쪽 Template 보안** — `ALLOWED_TEMPLATES`라는 Set에 허용된 템플릿 이름만 미리 등록해둡니다: company-invitation, guest-invitation, temporary-password. 사용자 입력에서 templateName이 오면 이 Set에 포함되어 있는지 먼저 체크하고, 없으면 즉시 거부합니다. 이렇게 하면 `../../etc/passwd` 같은 Path Traversal 공격이 원천 차단됩니다. Thymeleaf Context로 변수를 안전하게 바인딩해서 템플릿 인젝션도 방어합니다.

**오른쪽 토큰 캐시 전략** — Gmail OAuth2 토큰을 매번 Google에 요청하면 레이턴시가 발생하고 rate limit에도 걸릴 수 있습니다. 그래서 **volatile + synchronized로 double-checked locking**을 구현했습니다. cachedToken이 유효하면 바로 반환(캐시 히트)하고, 만료됐거나 없으면 synchronized 블록 안에서 Google OAuth2 토큰 엔드포인트를 호출해 갱신합니다. 핵심 디테일은 **TTL을 실제 만료 시간(60분)이 아닌 55분으로 설정**한 것입니다. 이 5분의 마진이 레이스 컨디션을 방지합니다. 만약 딱 60분에 맞추면, 토큰이 만료되는 순간과 갱신 요청이 겹쳐서 인증 실패가 발생할 수 있거든요.

---

## 📌 Slide 16 — Team: 팀 구성 및 역할

OnMeet을 만든 4명의 팀원 소개입니다.

- **최승은** — 팀장이자 Full-Stack. 프로젝트 총괄, Video Service 개발(화상회의/채팅/대기실), 프론트엔드 전반 개발을 담당했습니다.
- **박영진** — Backend Lead. MSA 전체 설계, Auth/Gateway/Notification 서비스 개발, CI/CD 파이프라인 구축, Go 전환 최적화를 리드했습니다.
- **조예성** — Backend. Notification Service 개발, Flyway 마이그레이션, 인프라 안정화, 모니터링 설정을 담당했습니다.
- **양진영** — AI Backend. AI Service 전담으로 STT 음성 인식, LLM 회의록 생성, AI 파이프라인 전체 구축을 담당했습니다.

4명의 개발자가 7개의 마이크로서비스를 5개 언어로 만들었습니다.

---

## 📌 Slide 17 — Team: 최승은 회고

## 🧠 상태 관리 전략 (1분 30초)

제가 프론트엔드를 전담하면서 가장 오래 고민한 부분은 **상태의 성격에 따라 저장소를 분리**하는 것이었습니다. STT처럼 고빈도로 갱신되는 데이터는 **useRef 버퍼**에 쌓아두고, 200~300ms 간격의 Throttle로 끊어서 렌더링합니다. 이걸 저는 **"Buffer → Throttle → Snapshot"** 패턴이라고 부르는데요 — SSE 스트림이 살아있는 동안은 로컬 버퍼가 진실의 원천이고, 스트림이 종료되면 `invalidateQueries`를 호출해서 서버 데이터로 전환합니다.

최종적으로 상태를 네 가지로 분류했습니다. 서버 원본은 **React Query**, UI 로컬 상태는 **useState**, 여러 컴포넌트가 공유하는 글로벌 상태는 **Zustand**, 렌더링과 무관한 고빈도 데이터는 **useRef**. 처음부터 이 분류가 있었던 건 아니고, 리렌더 폭풍을 직접 맞고 나서야 체감한 결론입니다.

---

## 📡 SSE 파서 직접 구현 (1분)

이 프로젝트에서 가장 자부심을 느끼는 부분입니다. 브라우저 표준 EventSource API는 **커스텀 헤더를 지원하지 않습니다**. 저희는 X-User-Id 헤더로 사용자를 식별하기 때문에 EventSource를 쓸 수 없었고, **fetch + ReadableStream으로 SSE 프로토콜을 직접 파싱**하는 훅을 구현했습니다.

가장 까다로웠던 건 **청크 경계 문제**입니다. TCP 패킷 경계와 SSE 메시지 경계가 일치하지 않아서, 하나의 청크에 메시지가 반만 들어올 수 있습니다. 버퍼에 쌓아두고 `\n\n` 구분자가 나올 때까지 기다리는 로직을 넣어야 했습니다. 연결 끊김 시 5초 자동 재연결, AbortController 기반 클린업까지 구현해서 알림·대기실·STT 세 가지 SSE를 독립적으로 관리합니다.

---

## 🏗 디자인 패턴 (1분)

패턴은 문제가 있을 때 꺼내 쓰는 도구라고 생각합니다. 실제로 적용한 것 위주로 말씀드리면,

**VAC 패턴**이 가장 효과가 컸습니다. CompanyManagement 컴포넌트가 **597줄**이었는데, 로직 컨테이너와 순수 UI를 분리하니까 **225줄**로 줄었습니다. 62% 감소입니다. 핵심은 UI를 고칠 때 로직을 읽을 필요가 없어졌다는 것이고, 이건 유지보수 속도에 직접적으로 영향을 줍니다.

**Custom Hooks**도 마찬가지입니다. 435줄짜리 MeetingPreparationModal에서 디바이스 관리(`useMeetingDevices`)와 마이크 테스트(`useMicTest`)를 훅으로 추출했습니다. 훅은 UI를 모릅니다. 상태와 핸들러만 반환하고, 렌더링은 컴포넌트가 결정합니다.

**Service Fetch Factory**도 빼놓을 수 없는데, 4개 MSA 서비스와의 통신 로직이 거의 동일한 패턴으로 반복되고 있었습니다. Content-Type 설정, X-User-Id 주입, 응답 파싱, 에러 처리 — 이걸 `createServiceFetch()` 팩토리 하나로 통합해서 약 120줄의 중복을 제거했습니다.

---

## ⚡ 빌드 & 인증 (40초)

빌드 최적화는 세 가지를 적용했습니다. **Manual Chunk Splitting**으로 LiveKit 478KB, TipTap 375KB 같은 대형 라이브러리를 독립 청크화해서 캐시 효율을 높였고, 사이드바 hover 시 **Route Prefetching**으로 다음 페이지를 미리 로드해서 즉각적 전환을 구현했습니다. 프로덕션에서는 esbuild.drop으로 console과 debugger를 자동 제거합니다.

인증 쪽에서 까다로웠던 건 **동시 다발적 401 처리**입니다. 페이지 로드 시 3개 API가 동시에 401을 반환하면 refresh 요청도 3번 나가서 race condition이 발생합니다. **Promise 싱글턴 패턴**으로, 첫 번째 401의 refresh Promise를 공유하게 만들어서 해결했습니다.

---

## 🌱 성장 & 교훈 (50초)

마지막으로, 25개 파일에 **253개 테스트 케이스**를 운영하고 있습니다. 이 과정에서 느낀 건, API 레이어 테스트가 가장 가성비가 좋고, 커스텀 훅은 수동 테스트만으로는 edge case 검증이 어렵다는 것입니다.

이 프로젝트에서 가장 크게 성장한 부분은 **"왜 이렇게 하는가"에 답하는 능력**입니다. 처음에는 "Zustand가 좋다고 해서" 쓰고, "React Query가 좋다고 해서" 썼습니다. 하지만 프로젝트가 복잡해지면서, 도구를 선택하는 것보다 **도구의 경계를 정하는 것**이 더 중요하다는 걸 깨달았습니다. "이 상태는 React Query에 담을까 Zustand에 담을까?" 같은 질문에 근거를 가지고 답할 수 있게 된 것 — 그게 이 프로젝트의 가장 큰 수확입니다.

감사합니다.

---

## 📌 Slide 18 — Team: 박영진 회고

Backend Lead 박영진의 회고입니다.

**MSA 설계** — 모놀리스의 빌드 병목을 해결하기 위해 7개 서비스로 도메인을 분리하고, Kafka 이벤트 버스로 서비스 간 통신을 비동기화했습니다. 각 서비스에 최적의 기술(Kotlin/Java/Go)을 선택할 수 있게 된 것이 MSA의 실질적 이점이었습니다.

**인증 & Gateway** — JWKS 공개키 배포로 비밀키 공유 없는 분산 인증을 구현하고, HttpOnly 쿠키 인증, CORS 중앙 관리를 Gateway에서 일원화했습니다.

**Go 전환** — Java 512MB/200ms였던 file-service를 Go 50MB/2ms로 개선했고, Docker 이미지도 300MB에서 20MB로 줄였습니다. API 응답 100배 개선이라는 극적인 결과를 얻었습니다.

**장애 대응** — Nginx DNS 캐시로 인한 502 에러, DB 커넥션 풀 고갈이 전체 서비스로 전파된 장애, email 서비스의 3중 장애 등을 경험하며 **"CORS≠CORS"** (표면적 증상과 실제 원인이 다를 수 있다)라는 교훈을 체득했습니다.

**CI/CD & 인프라** — 변경된 서비스만 빌드하는 선택적 빌드, Jib 레이어 캐싱 최적화, 17회 릴리즈 경험, 510커밋 무중단 배포를 달성했습니다.

---

## 📌 Slide 19 — Team: 조예성 회고

Backend 조예성의 회고입니다.

**SSE 멀티탭** — 단일 Map으로 연결이 끊기는 문제를 이중 Map 구조로 해결하고, 30초 heartbeat로 dead emitter를 자동 정리해서 멀티탭 동시 알림을 완성했습니다.

**FCM 푸시** — 3회 지수 백오프 재시도, 무효 토큰 자동 삭제, WEB/AOS/iOS 멀티 플랫폼 지원으로 SSE+FCM 이중 채널을 구축했습니다.

**알림 설정** — 5종 카테고리 토글, DND(방해금지) 모드, 발송 전 다층 체크로 세밀한 알림 제어를 구현했습니다.

**장애 대응** — 500/503 간헐 에러의 원인이 CORS SecurityConfig와 RestTemplate의 3s/5s 타임아웃 설정에 있었고, 스레드 대기 문제와 CORS를 동시에 해결했습니다.

**운영 자동화** — 10분 주기 예약 발송, 15일 지난 알림 새벽 3시 자동 정리, Kafka DLT(Dead Letter Topic) 에러 라우팅으로 무인 운영 시스템을 구축했습니다.

---

## 📌 Slide 20 — Team: 양진영 회고

AI Backend 양진영의 회고입니다.

**파이프라인 진화** — Mix STT의 화자 분리 불가 문제를 개별 트랙 분리 구독으로 해결하고, VAD를 ai-service에 배치하는 3 Phase 아키텍처를 완성했습니다.

**비용 방어** — N명 침묵 과금 폭발 문제를 Silero ONNX 로컬(비용 0원)과 PCM 정규화 정밀 필터로 해결해서 STT 과금을 70% 이상 방어했습니다.

**모델 선정** — whisper-1에서 gpt-4o-mini-transcribe로 교체해 한국어 인식률을 극대화하고, Claude의 Strict JSON 출력으로 Multi-LLM 앙상블 체계를 완성했습니다.

**포용성 설계** — 채팅 전용 참가자(마이크를 켤 수 없는 상황)의 텍스트도 음성 대본과 **Redis ZSET으로 시간순 병합**해서, 소외 없는 회의록 생성을 실현했습니다. 이건 기술적으로 인상적일 뿐 아니라, 제품 철학적으로도 의미 있는 설계였습니다.

**무중단 전환** — S3에서 DB 정규화로의 전환 시 Fallback 이중화를 심어서 기술부채 없는 마이그레이션을 달성했습니다.

---

## 📌 Slide 21 — Thank You / Q&A

마지막 슬라이드입니다.

정리하면, OnMeet은 **7개 마이크로서비스, 5개 프로그래밍 언어, 4개 데이터베이스, 1개 AI 파이프라인**으로 구성된 B2B 화상회의 플랫폼입니다.

저희가 가장 자부하는 세 가지를 말씀드리면:

첫째, **지갑을 지킨 아키텍처**. 침묵으로 나가는 무지성 클라우드 과금을 로컬 VAD ONNX와 PCM 오디오 정규화로 완벽히 방어했습니다.

둘째, **다중 LLM 조율 앙상블 체계**. 로컬 VAD로 최단 딜레이, gpt-4o-mini로 오디오 청취 특화, Claude로 NLP 구조화 특화 — 상황에 맞는 가장 뾰족하고 저렴한 무기 3가지를 조립해서 가성비 최상의 파이프라인을 만들었습니다.

셋째, **기술부채 없는 마이그레이션**. S3에서 DB로의 전환을 Fallback 패턴으로 무중단 완료해서, 기존 사용자 경험을 온전히 지켰습니다.

GitHub 리포지토리와 Swagger API 문서, 그리고 이 발표자료 링크는 QR코드로 확인하실 수 있습니다.
