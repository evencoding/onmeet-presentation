# OnMeet 발표 대본 (최종 — 읽기 전용)

---

## Slide 1 — Home

안녕하세요. 저희가 만든 OnMeet을 소개하겠습니다.

OnMeet은 한 문장으로 말하면, "AI가 회의를 기록하고 요약하는 B2B 화상회의 플랫폼"입니다.
실시간 화상회의 중에 참가자들의 음성을 자동으로 인식하고, 회의가 끝나면 AI가 키워드, 결정사항, 액션아이템까지 구조화된 회의록으로 만들어주는 서비스입니다.
단순히 Zoom 클론이 아니라, 회의 이후의 문서 작업까지 자동화한다는 점이 핵심 차별점입니다.

기술적으로는 7개의 마이크로서비스, 5개 프로그래밍 언어, 그리고 VAD에서 STT, LLM으로 이어지는 3단계 AI 파이프라인으로 구성되어 있습니다.
크게 4가지 핵심 축이 있는데요. WebRTC 기반 실시간 화상회의, AI 회의록 자동 생성, SSE와 FCM을 활용한 실시간 알림, 그리고 RSA와 AES 기반의 기업급 보안 아키텍처입니다.

그럼 아키텍처부터 하나씩 살펴보겠습니다.

---

## Slide 2 — System Architecture

이 슬라이드는 OnMeet의 전체 시스템 아키텍처입니다.

클라이언트 요청의 흐름을 따라가면, React와 Vite로 만든 클라이언트에서 출발한 요청은 먼저 Nginx 리버스 프록시를 거칩니다.
Nginx는 SSL 종단점 역할과 함께 정적 에셋 서빙, 도메인별 라우팅을 처리합니다.
그 다음 Gateway 서비스로 도달하는데, 이 게이트웨이가 Kotlin과 Spring WebFlux로 구현되어 있고, 모든 요청에 대해 JWKS 기반 JWT 검증과 Cookie 인증을 수행하는 보안 관문 역할을 합니다.

게이트웨이를 통과한 요청은 도메인별로 6개의 마이크로서비스로 분배됩니다.
auth-service는 Kotlin으로 작성했고, 인증과 인가를 전담합니다.
MySQL과 Redis를 사용하는데, Redis는 Refresh Token 저장과 세션 관리용입니다.
video-service는 Java로 작성했고, 화상회의의 핵심인 LiveKit SFU 서버와 연동하며 회의방 생성과 종료, 참여자 관리, 오디오 녹화를 담당합니다.
ai-service도 Java로 작성했고, VAD, STT, LLM 파이프라인 전체를 처리하는 AI 전용 서비스입니다.
notification-service도 Java이고, SSE와 FCM 이중 채널로 실시간 알림을 제공합니다.
file-service는 Go로 작성했습니다.
뒤에서 자세히 설명드리겠지만, Java 대비 응답속도 100배, 메모리 사용량 17배 절감이라는 극적인 성능 개선을 달성한 서비스입니다.
PostgreSQL과 MinIO를 사용합니다.
마지막으로 email-service는 Java이고, Gmail OAuth2 기반 이메일 발송을 전담합니다.

하단의 공통 인프라 레이어를 보시면, Kafka가 이벤트 버스 역할, Redis가 세션과 캐시, MinIO가 S3 호환 오브젝트 스토리지, MySQL과 PostgreSQL이 서비스별 데이터 저장소 역할을 수행합니다.

여기서 설계 원칙을 하나 강조하면, 서비스별 독립 데이터베이스 원칙을 철저히 지켰습니다.
auth는 auth의 MySQL만, file은 PostgreSQL만 접근합니다.
서비스 간 통신은 반드시 Kafka 이벤트 또는 API Gateway를 경유하도록 해서, MSA의 핵심인 느슨한 결합을 확보했습니다.

---

## Slide 3 — Key Features

OnMeet의 세 가지 핵심 기능을 정리한 슬라이드입니다.

첫 번째는 실시간 화상회의입니다.
video-service가 Java와 LiveKit SDK로 구현했고, WebRTC SFU 아키텍처를 기반으로 합니다.
SFU를 쓴 이유는 N대N 화상회의에서 Mesh 방식은 참여자가 늘어날수록 대역폭이 기하급수적으로 증가하기 때문입니다.
SFU는 서버가 중계해주면서도 P2P에 가까운 저지연을 유지할 수 있습니다.
갤러리 뷰와 스피커 뷰 전환, 채팅, 화면 공유, 대기실, PIP 모드까지 지원하고, 특히 Kafka Egress를 통한 오디오 레코딩이 AI 파이프라인의 입력 소스가 됩니다.

두 번째는 AI 회의록 자동 생성입니다.
ai-service에서 처리하며, 이게 OnMeet의 가장 핵심적인 차별 기능입니다.
Silero VAD로 무음을 필터링하고, OpenAI STT로 텍스트를 변환한 뒤, Claude가 구조화된 JSON 요약을 생성합니다.
결정사항과 액션아이템까지 자동으로 추출되고, 사용자는 BlockNote 에디터에서 바로 편집할 수 있습니다. 각 단계의 기술적 세부사항은 다음 슬라이드에서 상세히 다루겠습니다.

세 번째는 기업급 보안과 알림 시스템입니다.
RSA-2048 키 페어로 JWT를 서명하고, 개인키 자체를 AES-256-GCM으로 암호화해서 저장합니다.
키 유도 함수로 PBKDF2를 60만 회 반복하는데, 이건 브루트포스 공격을 사실상 불가능하게 만드는 수치입니다.
알림은 SSE와 FCM 이중 채널로 웹과 모바일 모두 커버하고, Kafka 이벤트 기반으로 비동기 발송합니다.
보안 아키텍처 역시 뒤의 전용 슬라이드에서 코드와 함께 자세히 보여드리겠습니다.

---

## Slide 4 — AI Pipeline

이 슬라이드가 OnMeet의 심장부, AI 파이프라인 전체 흐름도입니다.
6단계로 구성됩니다.

첫 번째 단계 Audio Capture에서는, 화상회의 중 LiveKit Track Egress가 각 참여자의 마이크 트랙을 OGG Opus 포맷으로 캡처해서 S3에 업로드합니다.
여기서 핵심은 혼합 음성이 아닌 참여자별 개별 트랙을 분리 저장한다는 겁니다.
혼합 음성을 STT에 넣으면 누가 말했는지 구분이 불가능하기 때문에, 완벽한 화자 분리를 위해 개별 트랙 분리가 필수적이었습니다.

두 번째 단계 Kafka Event에서는, 참여자가 회의를 떠나면 audio-chunk-ready 토픽에 이벤트가 발행됩니다.
참여자 정보와 청크 메타데이터가 포함되어 있고, 이 시점부터 비동기 파이프라인이 시작됩니다.
video-service는 이벤트를 던지고 즉시 본업인 화상회의 라우팅으로 복귀합니다.
Fire and Forget 패턴이죠.

세 번째 단계 VAD에서는, ai-service가 Kafka에서 이벤트를 소비하면 가장 먼저 Silero VAD v4 ONNX 모델을 돌립니다.
30밀리초 슬라이딩 윈도우로 오디오를 분석해서 무음 구간을 탈락시킵니다.
이게 왜 중요하냐면, 개별 트랙을 분리 저장하면 2명이 말할 때 나머지 3~4명의 침묵까지 전부 STT API로 전송되면서 과금이 N배로 폭발합니다.
VAD 필터링으로 STT 클라우드 비용을 최대 70% 이상 방어했습니다.
게다가 VAD는 백엔드의 내부 ONNX Runtime에서 돌아가므로 추가 클라우드 비용이 0원입니다.

네 번째 단계 STT에서는, VAD가 솎아낸 유효 음성만 gpt-4o-mini-transcribe에 전송합니다.
초기에는 whisper-1을 사용했는데, 한국어 인식률과 타임스탬프 정확성에서 아쉬움이 있었습니다.
gpt-4o-mini-transcribe로 교체한 뒤 한국어 인식률이 압도적으로 향상되었고, 처리 속도도 더 빠르면서 비용도 절감되는 가성비 끝판왕 모델입니다.

다섯 번째 단계 AI Summary에서는, 모든 음성 세그먼트가 수집되면 여기에 채팅 텍스트까지 시간순으로 병합한 뒤 Claude 3.5 Sonnet API에 던집니다.
Claude를 선택한 이유는 두 가지입니다.
첫째, 1~2시간짜리 대본도 소화할 수 있는 거대한 컨텍스트 윈도우. 둘째, 프롬프트 엔지니어링으로 순수 JSON 포맷을 변형 없이 완벽하게 출력하는 지시 수행력입니다.
DB 스키마와 1대1로 매핑되는 JSON을 뽑아내야 하는 저희 요구사항에 Claude가 타 모델 대비 월등했습니다.
100만 토큰당 약 3달러 수준의 비용 효율성도 선택의 이유였습니다.

여섯 번째 단계 Output에서는, 생성된 구조화 요약이 file-service에 저장되고, SSE 실시간 알림과 FCM 푸시로 참여자들에게 회의록 완성을 알립니다.

하단의 Kafka Topics Flow를 보시면, audio-chunk-ready에서 voice-segment, meeting-summary-ready, notification-send로 이어지는 이벤트 체인이 전체 파이프라인의 뼈대입니다.
평균 응답 지연은 3초 미만입니다.

---

## Slide 5 — Tech Stack

사용 기술 전체 목록입니다.

Backend는 Spring Boot 3.3.5 기반이고, 언어는 서비스 특성에 따라 선택했습니다.
auth와 gateway처럼 비동기 I/O가 중요한 서비스는 Kotlin과 Spring WebFlux, AI와 Video, Notification처럼 기존 에코시스템 활용이 중요한 서비스는 Java,
그리고 극한의 성능이 필요한 file-service는 Go로 작성했습니다.
JPA와 Flyway로 DB 마이그레이션을 관리합니다.

Frontend는 React 18에 TypeScript strict 모드, Vite 7입니다.
상태 관리는 Zustand v5, 서버 상태는 TanStack Query v5, 라우팅은 React Router v6, 스타일링은 Tailwind CSS로 구성했고, Cloudflare Pages에 배포합니다.

AI와 Media 영역은 앞서 설명드린 OpenAI STT, Claude API, Silero VAD ONNX, 그리고 LiveKit SFU와 WebRTC입니다.
이메일 발송에는 Gmail OAuth2와 Thymeleaf 템플릿 엔진을 사용합니다.

Infrastructure는 GCP e2-standard-4 인스턴스, 4 vCPU에 16기가 메모리 위에 Docker와 Compose로 전 서비스를 컨테이너화했고,
GitHub Actions로 CI/CD를 자동화합니다. 메시지 큐는 Kafka 7.5.0, 캐시는 Redis 7,
DB는 MySQL 9.0과 PostgreSQL 16, 오브젝트 스토리지는 MinIO, 리버스 프록시는 Nginx, DNS는 Cloudflare입니다.

하단의 언어 비율을 보시면 Java 40%, Kotlin 30%, Go 15%, TypeScript 15%로,
단일 언어에 종속되지 않고 각 서비스의 요구사항에 맞는 최적의 도구를 선택했다는 점을 강조드립니다.

---

## Slide 6 — Security Architecture

OnMeet의 다층 보안 아키텍처, Defense in Depth 전략입니다.
실제 코드는 뒤의 Code 섹션에서 보여드리고, 여기서는 전체 설계를 짚겠습니다.

왼쪽의 Authentication부터 보시면, auth-service에서 RSA-2048 키 페어를 생성하고 개인키를 AES-256-GCM으로 암호화해서 저장합니다.
키 유도에는 PBKDF2를 60만 회 반복하고, RS256으로 JWT를 서명한 뒤 공개키는 JWKS 엔드포인트로 노출합니다.
Refresh Token은 Redis에 7일 TTL로 저장하며 Rotation 정책을 적용해서 한 번 사용된 리프레시 토큰은 즉시 폐기됩니다.

가운데 Gateway Security는 모든 요청의 관문입니다.
HttpOnly Cookie에서 accessToken을 꺼내 Bearer Token으로 변환하고, JWKS를 통해 JWT를 검증합니다.
검증 통과 후 JWT claims에서 userId, email, roles를 추출해 X-User 헤더로 주입하므로 다운스트림 서비스들은 JWT를 재파싱할 필요가 없습니다.
X-Gateway-Secret 헤더도 자동 주입해서 내부 서비스 인증을 처리하고, Rate Limiting도 중앙 관리합니다.

오른쪽 Data Security에서는, file-service의 Go 코드에서 MIME 스푸핑 방지를 위한 바이너리 검사, S3 Key 인젝션 방어, Path Traversal 방어, 
Semaphore 기반 동시 업로드 제한, 그리고 email-service의 Template Traversal 방어까지 구현했습니다.

하단의 Request Lifecycle 플로우를 보시면, Browser Cookie에서 시작해서 Gateway JWT 검증, Headers 주입,
Service Auth Check, X-Gateway-Secret 확인을 거쳐 응답이 나가는 전체 보안 체인이 한눈에 보입니다.

---

## Slide 7 — Frontend Architecture

프론트엔드 아키텍처입니다.
여기서는 구조와 기술 선택 결과를 중심으로 설명드리고, 각각의 고민과 의사결정 과정은 제 회고 슬라이드에서 상세히 다루겠습니다.

프로젝트 구조는 Feature-Sliced Design을 적용해서, app에 글로벌 설정, features에 도메인별 모듈, shared에 공용 유틸과 훅과 컴포넌트, pages에 라우트별 페이지를 두었습니다.
도메인 간 의존성이 명확하고 파일 탐색이 직관적인 구조입니다.

디자인 패턴 측면에서는 VAC 패턴을 적용해 597줄짜리 컴포넌트를 225줄로 축소했고,
Custom Hooks로 useMeetingDevices, useMicTest 등을 추출했으며, Radix UI 기반 Compound Components 패턴도 활용했습니다.
Service Fetch Factory로는 4개 MSA와의 통신 로직 중복을 createServiceFetch 팩토리 하나로 통합해서,
Content-Type 자동 설정, X-User-Id 주입, 401 자동 갱신까지 한 줄로 처리됩니다.

커스텀 SSE 파서도 구현했는데, 브라우저의 EventSource API가 커스텀 헤더를 지원하지 않아서 fetch와 ReadableStream으로 직접 만들었습니다.
알림, 대기실, STT 세 가지 독립 SSE를 관리하고 5초 자동 재연결도 포함됩니다.
회의 참여 프로세스는 Preparing, Joining, Waiting, Connected, Disconnected 다섯 단계의 Phase 상태 머신으로 모델링했습니다.

성능 최적화로는 Route-level lazy loading으로 10개 이상의 페이지를 동적 로딩하고,
LiveKit이나 TipTap 같은 대형 라이브러리를 Manual chunk splitting으로 독립 청크화했습니다.
사이드바 hover 시 다음 페이지를 미리 로드하고, React.memo와 useShallow로 리렌더를 최소화했으며, 프로덕션 빌드에서 console과 debugger를 자동 제거합니다.

보안은 CSP 화이트리스트, HttpOnly 쿠키 인증, Promise 싱글턴 토큰 갱신, DOMPurify XSS 방지, 권한별 Route Guard를 적용했고,
PWA도 지원해서 정적 에셋 Precaching, API NetworkFirst 캐싱, 오프라인 폴백, Service Worker 자동 갱신까지 구현했습니다.

---

## Slide 8 — Infrastructure & DevOps

GCP와 Docker, GitHub Actions 기반 CI/CD 파이프라인입니다.

상단의 CI/CD Pipeline 흐름을 보시면, 개발자가 feature branch에서 작업하고, PR을 통해 develop 브랜치에 머지한 뒤, release 브랜치를 push하면 CI/CD가 트리거됩니다.
GitHub Actions가 변경된 서비스만 감지해서 선택적으로 빌드하고, Jib으로 Dockerfile 없이 Docker 이미지를 빌드합니다.
이미지는 Docker Hub에 push되고, GCP VM에서 docker compose up으로 배포됩니다.

배포 전략은 Jib의 레이어 캐싱으로 CI 시간을 단축하고, 변경된 서비스만 선택적으로 빌드하며, Semantic Versioning을 적용해서 버전 관리를 체계화했습니다.

네트워크는 Cloudflare DNS Proxy로 DDoS 방어와 SSL을 처리하고, api.onmeet.cloud는 Nginx를 거쳐 Gateway로,
rtc.onmeet.cloud는 LiveKit 포트로, files 경로는 MinIO로 라우팅됩니다.
인프라는 GCP e2-standard-4 인스턴스를 서울 리전에 배치했습니다.

Git Workflow는 main, develop, feature, hotfix 네 가지 브랜치 전략을 운용하고,
하단의 Version History를 보시면 v0.1.0 MVP에서 v0.4.20 현재까지 점진적으로 기능을 확장하면서 안정성을 확보해온 과정이 보입니다.

---

## Slide 9 — Code: 인증 서비스의 암호화 전략

여기서부터 Code 섹션입니다.
앞서 보안 아키텍처 슬라이드에서 전체 설계를 보여드렸는데, 여기서는 실제 구현 코드를 보시겠습니다.

왼쪽의 KeyManager.kt는 RSA 개인키를 AES-256-GCM으로 암호화해서 저장하는 코드입니다.
PBKDF2를 60만 번 반복하는 KDF로 마스터 키를 유도하고, SecureRandom으로 16바이트 Salt와 12바이트 IV를 생성합니다.
GCM 인증 태그 길이는 128비트이고, 최종 저장 포맷은 Salt 16바이트, IV 12바이트, CipherText를 하나의 바이트 배열로 담는 구조입니다.
매번 다른 Salt와 IV가 생성되므로 동일 평문이라도 암호문이 달라집니다.

오른쪽의 JwtTokenProvider.kt는 RS256으로 JWT를 서명하는 코드입니다.
email을 subject로, userId와 roles를 claims에 담고, accessToken은 30분 TTL로 발행합니다.
keyManager.getPrivateKey가 호출되면 위의 AES-GCM 복호화가 투명하게 수행되어, 개인키가 메모리에서만 존재합니다.

핵심은 비밀키를 서비스 간에 공유하지 않는 분산 인증 구조라는 점입니다.
공개키만 JWKS로 노출하고, Gateway의 NimbusDecoder가 이를 가져다 검증합니다.

---

## Slide 10 — Code: Gateway 보안 관문

앞서 보안 슬라이드에서 소개한 Gateway의 실제 필터 체인 코드입니다.

CookieConverter는 브라우저의 HttpOnly Cookie에 담긴 accessToken을 Spring Security의 BearerToken 객체로 변환합니다.
XSS 공격으로 document.cookie에 접근하는 것 자체가 불가능하므로 토큰 탈취를 원천 차단합니다.

UserHeaderFilter는 JWT claims에서 userId, email, roles를 추출해서 X-User-Id, X-User-Email, X-User-Roles 헤더로 요청에 주입합니다.
다운스트림 서비스들은 JWT를 다시 파싱할 필요 없이 헤더만 읽으면 됩니다.
인증 로직이 Gateway에 집중되고, 각 서비스는 비즈니스 로직에만 집중할 수 있습니다.

SecureInternalFilter는 Gateway가 X-Gateway-Secret 헤더를 자동 주입해서, 내부 서비스가 이 요청이 진짜 Gateway를 통과한 건지 검증합니다.
외부에서 직접 내부 서비스 포트로 접근하는 우회 공격을 차단하는 역할입니다.

하단의 Request Flow를 보시면, Browser Cookie에서 CookieConverter, BearerToken, NimbusJwtDecoder의 JWKS 검증,
UserHeaderFilter, SecureInternalFilter를 거쳐 성공 응답으로 이어지는 체인이 모든 요청마다 실행됩니다.

---

## Slide 11 — Code: 화상회의 녹화 자동화 파이프라인

AI 파이프라인 슬라이드에서 설명드린 Audio Capture 단계의 실제 구현 코드입니다.

WebhookController는 LiveKit 서버에서 오는 Webhook 중 TRACK_PUBLISHED 이벤트를 감지해서 참여자별 마이크 트랙의 Egress를 자동 시작합니다.
참여자가 입장할 때마다 수동 조작 없이 자동으로 개별 녹화가 시작되고, PARTICIPANT_LEFT가 발생하면 Kafka에 audio-chunk-ready 이벤트를 발행합니다.

RecordingService는 LiveKit의 TrackEgress API를 호출해서 마이크 트랙을 S3에 OGG 포맷으로 저장합니다.
activeEgress Map으로 현재 진행 중인 녹음을 추적 관리하고, 참여자 퇴장 시 해당 Egress를 종료합니다.

LiveKitClient는 LiveKit API가 요구하는 HS256 JWT를 외부 라이브러리 없이 Raw HMAC-SHA256으로 직접 생성했습니다.
base64Url 인코딩과 HMAC 서명까지 순수 Java 코드로 구현해서 의존성을 최소화했습니다.

하단의 Recording Flow를 보시면, LiveKit Webhook에서 TrackPublished, startTrackEgress, S3 OGG 저장, ParticipantLeft를 거쳐 Kafka audio-chunk-ready로 이벤트가 흘러갑니다.
이 이벤트가 다음 슬라이드의 AI 파이프라인 입력이 됩니다.

---

## Slide 12 — Code: 음성에서 텍스트, 회의 요약까지의 AI 파이프라인

AI 파이프라인의 3대 핵심 모델 구현 코드입니다.

왼쪽의 SileroVadClient는 과금 방어의 첫 번째 관문입니다.
ONNX Runtime으로 Silero VAD v4 모델의 딥러닝 추론을 수행합니다.
오디오 배열을 OnnxTensor로 변환하고 state 벡터와 함께 모델에 입력하면, 각 윈도우의 음성 확률이 출력됩니다.
임계값 이상이면 유의미한 발화 구간으로 판정하고, 일정 시간 이상 연속 침묵하면 이전 유음 덩어리를 잘라냅니다.
PCM 오디오 정규화도 적용해서 마이크 볼륨이 달라도 작은 속삭임까지 누락 없이 분석합니다.
이 모든 게 백엔드 메모리에서 로컬로 동작하므로 클라우드 비용은 0원입니다.

가운데 SttWorkerService는 VAD가 잘라준 유효 음성 세그먼트들을 순회하면서 gpt-4o-mini-transcribe에 전송합니다.
100밀리초 미만의 초단 노이즈는 스킵하고, 의미 있는 길이의 세그먼트만 STT API를 호출합니다.

오른쪽 ClaudeSummarizer는 프롬프트 엔지니어링의 핵심입니다.
순수 JSON 객체만 출력하라는 강력한 지시와 함께 정확한 스키마를 제시하고, Claude가 반환한 JSON을 objectMapper.readValue로 역직렬화해서 스키마와 맞지 않으면 즉시 파싱 예외가 발생하도록 설계했습니다.
이 검증을 통과해야만 DB에 구조화된 데이터로 적재될 자격이 주어집니다.

---

## Slide 13 — Code: 웹과 모바일 실시간 알림 이중 채널

notification-service의 SSE와 FCM 이중 채널 구현입니다.

SSE 멀티탭 부분을 보시면, Map 안에 Map을 넣는 이중 구조로 사용자별 다중 탭과 디바이스를 독립적으로 관리합니다.
30초마다 heartbeat를 보내서 끊어진 연결을 자동으로 정리합니다.

FCM Dispatch는 Auth 서비스에서 관리하는 최신 토큰과 로컬 DB의 토큰을 Set으로 합쳐 중복 없이 멀티캐스트로 발송합니다.
실패 시 최대 3회 지수 백오프로 재시도하고, 무효 토큰은 자동 삭제합니다.

Notification Toggle은 사용자별로 회의, 팀, 회의록 3가지 카테고리를 개별적으로 켜고 끌 수 있고, 토글이 꺼져 있으면 발송 자체를 차단하는 구조입니다.

---

## Slide 14 — Code: 왜 Go를 썼는지, 보안까지 잡은 파일 서버

file-service의 Go 전환 기술 선택과 결과입니다.

보안 쪽 코드를 보시면, http.DetectContentType으로 파일의 실제 바이너리를 검사해서 MIME 스푸핑을 방지합니다.
확장자를 jpg로 바꿔서 업로드해도 실제 바이너리가 이미지가 아니면 차단됩니다.
S3 Key 인젝션 방어는 ownerType을 화이트리스트로 제한하고 ownerId를 숫자 정규식으로 검증합니다.

성능 쪽은 Go의 채널을 세마포어로 활용해서 동시 업로드를 최대 10개 goroutine으로 제한하고, 1MB 기준으로 캐시와 스트리밍을 분기합니다.

하단의 결과 수치가 인상적인데요.
응답 속도가 Kotlin의 약 450밀리초에서 Go의 약 4밀리초로 100배 이상 개선되었고, 메모리는 JVM의 512메가에서 Go의 30메가로 17배 줄었습니다.
Docker 이미지 크기도 300메가에서 20메가로 줄었고요.
파일 업로드와 다운로드는 I/O 바운드 작업이 대부분이라 JVM의 무거운 부트스트랩이 불필요했고, Go의 경량성이 이 유스케이스에 완벽히 맞았습니다.

---

## Slide 15 — Code: 안전한 이메일 발송과 OAuth2 토큰 관리

email-service의 보안과 토큰 효율성 코드입니다.

왼쪽의 Template 보안을 보시면, ALLOWED_TEMPLATES라는 Set에 허용된 템플릿 이름만 미리 등록해둡니다.
사용자 입력에서 templateName이 오면 이 Set에 포함되어 있는지 먼저 체크하고, 없으면 즉시 거부합니다.
이렇게 하면 상위 디렉토리로 이동하는 Path Traversal 공격이 원천 차단됩니다.

오른쪽의 토큰 캐시 전략은 Gmail OAuth2 토큰을 매번 Google에 요청하면 레이턴시와 rate limit 문제가 있어서, volatile과 synchronized로 double-checked locking을 구현했습니다.
핵심 디테일은 TTL을 실제 만료 시간 60분이 아닌 55분으로 설정한 것입니다. 이 5분의 마진이 토큰 만료 시점과 갱신 요청이 겹치는 레이스 컨디션을 방지합니다.

---

## Slide 16 — Team: 팀 구성 및 역할

OnMeet을 만든 4명의 팀원 소개입니다.

저 최승은은 팀장이자 Full-Stack으로, 프로젝트 총괄과 Video Service 개발, 프론트엔드 전반을 담당했습니다.
박영진님은 Backend Lead로, MSA 전체 설계와 Auth, Gateway, Notification 서비스 개발, CI/CD 파이프라인 구축, Go 전환 최적화를 리드했습니다.
조예성님은 Backend로, Notification Service 개발과 Flyway 마이그레이션, 인프라 안정화를 담당했습니다.
양진영님은 AI Backend로, AI Service 전담으로 STT 음성 인식과 LLM 회의록 생성, AI 파이프라인 전체 구축을 담당했습니다.

4명의 개발자가 7개 마이크로서비스를 5개 언어로 만들었습니다. 지금부터는 각자가 어떤 고민과 시행착오를 거쳤는지, 회고 형식으로 말씀드리겠습니다.

---

## Slide 17 — Team: 최승은 회고

앞서 프론트엔드 아키텍처와 기술 선택 결과를 보여드렸는데, 이번에는 그 뒤에 있었던 고민과 시행착오를 말씀드리겠습니다.

가장 오래 고민한 문제는 상태를 어디에 담을 것인가였습니다.
화상회의 플랫폼은 한 화면에서 카메라 on/off, 마이크 상태, 참여자 입퇴장, 채팅, AI 실시간 전사가 동시에 벌어집니다.
처음에는 모든 걸 useState로 처리했습니다.
그러다 livekit 실시간 자막 STT 데이터가 초당 수십 번 업데이트되면서 리렌더 폭풍을 맞았습니다.
React가 매 프레임마다 리렌더를 돌리니까 UI가 완전히 버벅이더라고요.

이 문제를 겪고 나서야, 상태의 성격에 따라 저장소를 분리해야 한다는 걸 체감했습니다.
STT처럼 고빈도 데이터는 useRef 버퍼에 쌓아두고, 200에서 300밀리초 간격의 Throttle로 끊어서 렌더링합니다.
debounce를 걸면 되지 않느냐고 생각하실 수 있는데, debounce는 마지막 입력 후 일정 시간이 지나야 반영되므로 실시간 자막에서는 사용자가 멈춘 것처럼 느낍니다.
Throttle이 맞습니다.
SSE 스트림 데이터도 고민이었는데, 스트림이 살아있는 동안은 로컬 버퍼가 진실의 원천이고, 스트림이 종료되면 invalidateQueries로 서버 데이터로 전환합니다.
이걸 Buffer, Throttle, Snapshot 패턴이라 부르기로 했습니다.
처음부터 이 분류가 있었던 건 아니고, 문제를 직접 맞은 다음에야 정리된 결론입니다.

이 프로젝트에서 가장 자부심을 느끼는 부분은 SSE 파서 직접 구현입니다.
브라우저 표준 EventSource API가 커스텀 헤더를 지원하지 않습니다.
저희는 X-User-Id 헤더로 사용자를 식별하기 때문에 EventSource를 쓸 수 없었고, fetch와 ReadableStream으로 SSE 프로토콜을 직접 파싱하는 훅을 만들었습니다.
가장 어려웠던 건 청크 경계 문제였습니다.
TCP 패킷 경계와 SSE 메시지 경계가 일치하지 않아서, 메시지가 반만 들어올 수 있습니다.
이걸 모르고 바로 파싱하면 JSON.parse가 터집니다.
버퍼에 쌓아두고 줄바꿈 두 번이 나올 때까지 기다리는 로직을 넣어서 해결했고, 5초 자동 재연결과 AbortController 클린업까지 포함시켰습니다.
이 경험이 나중에 대용량 파일 다운로드의 ReadableStream 진행률 추적에도 그대로 응용되었습니다.

디자인 패턴에 대해서는, 패턴을 공부하면 전부 다 적용해야 할 것 같은 유혹이 옵니다.
하지만 패턴은 문제가 있을 때 꺼내 쓰는 도구이지 미리 심어두는 장식이 아닙니다.
VAC 패턴은 CompanyManagement가 597줄까지 불어나서 하나를 고치려면 파일 전체를 읽어야 했을 때 도입했습니다.
Custom Hooks도 MeetingPreparationModal이 435줄이 되어서야 추출했고요.
3번 반복되기 전에는 추상화하지 않는다는 원칙을 지키려고 노력했고,
각 페이지의 로딩 스켈레톤 같은 건 비슷해 보여도 각 레이아웃에 맞춰져 있어서 의도적으로 통합하지 않았습니다.

인증에서 가장 까다로웠던 건 동시 다발적 401 처리였습니다.
페이지 로드 시 3개 API가 동시에 401을 반환하면 refresh도 3번 나가서 race condition이 생깁니다.
첫 번째 401의 refresh Promise를 공유하는 Promise 싱글턴 패턴으로 해결했습니다.
이건 직접 겪기 전까지는 왜 이런 걸 고민해야 하지 싶은 종류의 문제입니다.

마지막으로, 25개 파일에 253개 테스트 케이스를 운영하면서,
API 레이어 테스트가 가장 가성비가 좋고 커스텀 훅은 수동 테스트만으로 edge case 검증이 어렵다는 걸 배웠습니다.
이 프로젝트에서 가장 크게 성장한 부분은 왜 이렇게 하는가에 답하는 능력입니다.
처음에는 좋다고 해서 도구를 골랐지만, 프로젝트가 복잡해지면서 도구를 선택하는 것보다 도구의 경계를 정하는 것이 더 중요하다는 걸 깨달았습니다.
auth/api.ts가 471줄까지 불어난 다음에야 분리의 필요성을 체감했고, 리렌더 폭풍을 맞은 다음에야 useRef 버퍼링의 가치를 이해했습니다.
문제를 직접 겪어야 해결책의 의미를 진짜로 알게 됩니다. 그게 이 프로젝트의 가장 큰 수확입니다. 감사합니다.

---

## Slide 18 — Team: 박영진 회고

박영진 Backend Lead의 회고입니다. 앞서 아키텍처와 코드로 결과를 보여드렸는데, 여기서는 의사결정 과정과 장애 경험을 중심으로 말씀드립니다.

MSA를 처음부터 선택한 건 아닙니다. 모놀리스로 시작했다가 빌드 시간이 점점 길어지고, AI 모듈 변경이 인증 서비스 재배포를 유발하는 문제가 터졌습니다. 7개 서비스로 분리하면서 Kafka 이벤트 버스를 도입했고, 그 결과 서비스별로 최적의 기술을 독립적으로 선택할 수 있게 된 것이 MSA의 실질적 이점이었습니다.

인증에서 JWKS를 선택한 핵심 이유는 비밀키를 서비스 간에 공유하지 않기 위해서입니다. 대칭키 기반이면 모든 서비스가 비밀키를 가져야 하고, 하나라도 뚫리면 전체가 위험해집니다. 비대칭키와 JWKS로 공개키만 배포하는 구조가 MSA에 맞습니다.

앞서 Go 전환으로 응답 속도 100배 개선이라는 결과를 보여드렸는데, 이 결정이 쉽지는 않았습니다. 팀원 대부분이 Java와 Kotlin에 익숙한 상황에서 새 언어를 도입하는 건 리스크입니다. 하지만 file-service의 특성이 I/O 바운드이고 비즈니스 로직이 단순하며 동시성 요구가 높다는 점에서 Go의 강점과 정확히 맞아떨어졌기에 결정했습니다.

장애 대응에서 가장 아픈 경험은 CORS는 CORS가 아니다라는 교훈입니다. 프론트에서 CORS 에러가 터지면 보통 CORS 설정을 의심하는데, 실제로는 Nginx DNS 캐시 문제, DB 커넥션 풀 고갈, email 서비스 3중 장애 등 완전히 다른 원인이 CORS 에러로 포장되어 나타났습니다. 표면적 증상과 근본 원인이 다를 수 있다는 걸 몸으로 체득했습니다.

CI/CD는 510커밋 17회 릴리즈를 무중단으로 배포한 경험이고, 변경 서비스만 빌드하는 선택적 빌드와 Jib 레이어 캐싱이 결정적이었습니다.

---

## Slide 19 — Team: 조예성 회고

조예성 Backend의 회고입니다. 앞서 알림 시스템의 기술 구현을 보여드렸는데, 여기서는 직면한 문제들과 해결 과정을 말씀드립니다.

가장 당황스러웠던 건 알림이 안 온다는 리포트였습니다. 원인을 추적해보니, 사용자가 새 탭을 열면 단일 Map에서 기존 연결이 덮어써져서 이전 탭의 알림이 유실되고 있었습니다. Map 하나면 되지라는 단순한 가정이 깨진 순간이었고, 이중 Map 구조로 전환하면서 30초 heartbeat로 끊어진 연결을 정리하는 로직도 추가했습니다.

FCM 쪽에서는 토큰 관리가 예상보다 복잡했습니다. Auth 서비스의 최신 토큰과 로컬 DB 토큰이 불일치하는 경우가 있어서, Set으로 합쳐 중복을 제거하는 방식을 택했습니다. 발송 실패 시 3회 지수 백오프로 재시도하고, 무효 토큰은 자동 삭제해서 쓸데없는 재시도를 줄였습니다.

장애 대응에서는 500/503 간헐 에러가 발생했는데, 원인이 CORS SecurityConfig와 RestTemplate 타임아웃이 겹쳐서 스레드가 대기 상태에 빠지는 것이었습니다. 앞서 박영진이 말한 CORS는 CORS가 아니다와 같은 맥락인데, 저도 에러 메시지만 보고 SecurityConfig만 만지다가 실제로는 타임아웃 조정이 답이었습니다.

운영 자동화로는 10분 주기 예약 발송, 15일 지난 알림 새벽 3시 자동 정리, Kafka Dead Letter Topic 에러 라우팅으로 무인 운영 시스템을 구축했습니다. 사람이 안 봐도 돌아가는 시스템을 만드는 것의 가치를 체감한 부분입니다.

---

## Slide 20 — Team: 양진영 회고

양진영 AI Backend의 회고입니다. 앞서 AI 파이프라인의 구조와 코드를 보여드렸는데, 여기서는 3번의 아키텍처 전면 수정을 거쳐야 했던 고민 과정을 말씀드립니다.

처음에는 가장 쉬운 방법인 혼합 음성 통짜 STT를 시도했습니다. 트래픽도 적고 구현도 단순했지만, 결과물을 보는 순간 누가 이 말을 했는지 알 수 없다는 치명적 한계에 부딪혔습니다. 화자 분리 없는 회의록은 의미가 없었기에, 참여자별 개별 트랙 분리 구독으로 전면 수정했습니다.

그런데 개별 트랙을 분리하니까 이번에는 침묵 과금 폭발이라는 새로운 문제가 터졌습니다. 6명 회의에서 2명만 말하고 있으면 나머지 4명의 빈 오디오까지 전부 STT API로 전송되어 과금이 6배로 늘어났습니다. 이 문제를 해결하기 위해 VAD를 도입했고, 어디에 배치할지도 고민이었습니다. video-service에 넣으면 미디어 처리와 딥러닝 연산이 뒤섞여 메모리 과부하 우려가 있었기 때문에, MSA 관심사 분리 원칙에 따라 ai-service 입구에 배치했습니다. 이게 3번의 전면 수정 끝에 도달한 최종 아키텍처입니다.

모델 선정도 쉽지 않았습니다. whisper-1로 시작했지만 한국어 인식률이 아쉬웠습니다. gpt-4o-mini-transcribe로 교체한 건 단순한 업그레이드가 아니라, 테스트 비용을 감수하면서 한국어 품질을 검증한 결과였습니다. Claude도 GPT-4o와 비교 테스트를 거쳤고, JSON 포맷 준수율에서 Claude가 월등했습니다.

포용성 설계도 말씀드리고 싶습니다. 도서관이나 오픈 스페이스처럼 마이크를 켤 수 없는 참가자도 있습니다. 이들의 채팅이 회의록에서 소외되면 안 됩니다. 음성과 채팅이라는 완전히 다른 매체를 하나의 대화 흐름으로 합치기 위해 Redis ZSET을 선택했습니다. 타임스탬프를 score로 사용해서 삽입 즉시 시간순 정렬이 보장되는 구조입니다. 기술적으로도 의미 있지만, 모든 참가자의 의견이 요약에 반영된다는 제품 철학적 가치도 큽니다.

마지막으로 무중단 전환인데, S3에서 DB로 저장소를 바꿀 때 이미 S3에 저장된 기존 회의록이 깨지면 안 됩니다. DB를 먼저 조회하되, 없으면 S3 파싱으로 Fallback하는 이중화 체계를 심어서 배포 시점의 100% 하위 호환성을 확보했습니다. 라이브 운영 중에 저장소를 교체하는 건 매우 위험한 작업인데, Fallback 덕분에 무사고로 전환할 수 있었습니다.

---

## Slide 21 — Thank You / Q&A

마지막 슬라이드입니다.

정리하면, OnMeet은 7개 마이크로서비스, 5개 프로그래밍 언어, 4개 데이터베이스, 1개 AI 파이프라인으로 구성된 B2B 화상회의 플랫폼입니다.

저희가 가장 자부하는 세 가지를 말씀드리면, 첫째는 지갑을 지킨 아키텍처입니다.
로컬 VAD ONNX와 PCM 정규화로 STT 비용을 70% 이상 방어했습니다.
둘째는 다중 LLM 앙상블 체계입니다.
VAD로 최단 딜레이, gpt-4o-mini로 오디오 특화, Claude로 NLP 구조화 특화해서, 각 단계에 가장 뾰족한 도구를 조립했습니다.
셋째는 기술부채 없는 마이그레이션입니다.
S3에서 DB로의 전환을 Fallback 패턴으로 무중단 완료해서 기존 사용자 경험을 온전히 지켰습니다.

GitHub 리포지토리와 Swagger API 문서, 그리고 이 발표자료 링크는 QR코드로 확인하실 수 있습니다.

발표를 경청해주셔서 감사합니다. 질문 있으시면 말씀해주세요.
