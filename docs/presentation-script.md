# Onmeet 기술 발표 대본

> 총 17 슬라이드 / 예상 발표 시간: 22~27분

---

## Slide 1 — Hero (1분)

안녕하세요, Onmeet 프로젝트를 발표하겠습니다.

Onmeet은 **AI가 회의를 기록하고 요약하는 B2B 화상회의 플랫폼**입니다.

기업 환경에서 필요한 실시간 화상회의, AI 기반 회의록 자동 생성, 그리고 실시간 알림 시스템까지 하나의 플랫폼에서 제공합니다.

이 프로젝트는 7개의 마이크로서비스로 구성되어 있고, Kotlin, Java, Go, TypeScript, SQL 총 5개의 언어를 사용했습니다.

---

## Slide 2 — System Architecture (2분)

전체 시스템 아키텍처입니다.

클라이언트는 React + TypeScript로 구현되었고, Cloudflare Pages에 배포됩니다.

모든 요청은 Nginx 리버스 프록시를 거쳐 Spring Cloud Gateway로 들어옵니다. Gateway는 WebFlux 기반의 리액티브 아키텍처로, JWT 검증과 요청 라우팅을 담당합니다.

Gateway 뒤에 6개의 마이크로서비스가 있습니다:
- **auth-service** — Kotlin으로 작성된 인증/사용자/팀/회사 관리 서비스
- **video-service** — LiveKit 기반 화상회의 관리
- **ai-service** — STT와 LLM을 활용한 AI 회의록 생성
- **notification-service** — SSE와 FCM 기반 실시간 알림
- **file-service** — Go로 작성된 고성능 파일 서비스
- **email-service** — 이메일 발송 서비스

모든 서비스는 **Database-per-service** 패턴을 따르고, **Apache Kafka**를 통해 이벤트 기반으로 통신합니다.

---

## Slide 3 — Key Features (2분)

핵심 기능은 크게 세 가지입니다.

첫째, **실시간 화상회의**입니다. LiveKit SFU 기반의 WebRTC로 초저지연 다자간 화상회의를 지원합니다. Gallery와 Speaker 뷰 전환, 화면 공유, 실시간 채팅, 대기실 시스템, 그리고 PIP 모드까지 지원합니다.

둘째, **AI 회의록 자동 생성**입니다. 회의가 끝나면 자동으로 음성을 텍스트로 변환하고, Claude AI가 구조화된 회의록을 생성합니다. 요약, 키워드, 결정사항, 액션 아이템을 자동으로 추출합니다.

셋째, **기업급 보안과 실시간 알림**입니다. RSA-256 JWT에 JWKS 기반 공개키 배포, AES-256-GCM으로 개인키를 암호화하여 저장합니다. 알림은 SSE로 실시간 전달하고 FCM으로 모바일 푸시까지 지원합니다.

---

## Slide 4 — AI Pipeline (2분)

이 프로젝트에서 기술적으로 가장 도전적이었던 AI 파이프라인입니다.

전체 흐름은 이렇습니다:

1. 참가자가 마이크를 켜면, LiveKit이 참가자별 개별 오디오 트랙을 S3에 녹음합니다.
2. 녹음이 완료되면 Kafka를 통해 `audio-chunk-ready` 이벤트가 발행됩니다.
3. ai-service가 이를 수신하면, 먼저 **Silero VAD**라는 ONNX 기반 음성 활동 감지 모델로 무음 구간을 필터링합니다. 이를 통해 STT 비용을 70~80% 절감했습니다.
4. 실제 발화 구간만 **OpenAI gpt-4o-mini-transcribe**에 보내 텍스트로 변환합니다.
5. 완성된 트랜스크립트를 **Claude API**에 전달해 구조화된 JSON 형태의 회의록을 생성합니다.
6. 최종 결과물은 file-service에 저장되고, SSE를 통해 사용자에게 알림이 전달됩니다.

모든 단계가 Kafka 토픽으로 연결되어 있어, 각 단계가 독립적으로 스케일링 가능합니다.

---

## Slide 5 — Tech Stack (1분)

사용한 기술 스택입니다.

백엔드는 Kotlin과 Java 기반 Spring Boot, Go의 Gin 프레임워크를 사용했습니다. 프론트엔드는 React 18에 TypeScript, Vite 7, TanStack Query v5와 Zustand v5로 상태 관리를 합니다.

AI 파이프라인에는 Silero VAD, OpenAI gpt-4o-mini-transcribe, Anthropic Claude를 조합했고, 인프라는 GCP VM 위에 Docker Compose로 운영하고 있습니다.

---

## Slide 6 — Security Architecture (2분)

보안 아키텍처는 세 가지 축으로 설계했습니다.

**첫째, 인증과 토큰 관리입니다.** RSA-2048 키 쌍을 서버 시작 시 자동 생성하고, 개인키는 AES-256-GCM으로 암호화하여 DB에 저장합니다. PBKDF2에 60만 번 반복으로 키를 유도합니다. JWT는 HttpOnly Secure 쿠키로 전달해 XSS 토큰 탈취를 원천 차단합니다.

**둘째, Gateway 보안입니다.** 모든 요청은 반드시 Gateway를 통과해야 합니다. Gateway가 JWT를 검증한 후 X-User-Id, X-User-Email 등의 헤더를 주입하고, X-Gateway-Secret 헤더를 추가합니다. 다운스트림 서비스는 이 시크릿이 없으면 요청을 거부합니다.

**셋째, 데이터 보안입니다.** file-service에서는 실제 파일 바이트를 검사해 MIME 스푸핑을 방어하고, S3 키 경로 순회 공격도 차단합니다. 회사 간 데이터 격리도 companyId 검증으로 보장합니다.

---

## Slide 7 — Frontend Architecture (1.5분)

프론트엔드는 Feature-Sliced Design 패턴으로 구조화했습니다.

핵심은 세 가지입니다.

첫째, **Service Fetch Factory**입니다. 4개 백엔드 마이크로서비스의 중복 fetch 로직을 팩토리 패턴으로 통합했습니다. Content-Type 자동 설정, X-User-Id 헤더 주입, 401 자동 토큰 갱신까지 처리합니다.

둘째, **커스텀 SSE 파서**입니다. 표준 EventSource는 커스텀 헤더를 지원하지 않아, fetch + ReadableStream으로 직접 구현했습니다. 실시간 알림, 대기실 승인/거절, 실시간 STT를 모두 이 파서로 처리합니다.

셋째, **Phase 상태 머신**입니다. 회의실은 Preparing → Joining → Waiting → Connected → Disconnected 다섯 단계의 생명주기를 가지며, Zustand 상태 머신으로 관리합니다.

성능 최적화로는 Route-level lazy loading, Manual chunk splitting, Route prefetching을 적용했고, CSP 화이트리스트 정책과 DOMPurify XSS 방지 등 보안에도 신경 썼습니다. PWA도 지원합니다.

---

## Slide 8 — Infrastructure & DevOps (1.5분)

인프라와 배포 파이프라인입니다.

코드가 GitHub에 push되면 GitHub Actions가 변경된 서비스만 감지해서 Jib으로 Docker 이미지를 빌드하고 Docker Hub에 푸시한 뒤, GCP VM에 자동 배포합니다.

GCP VM은 e2-standard-4 인스턴스에 Docker Compose로 7개 서비스, Kafka, MySQL, PostgreSQL, Redis, MinIO, LiveKit을 운영하고 있습니다.

네트워크는 Cloudflare DNS 프록시를 통해 api.onmeet.cloud은 Gateway로, rtc.onmeet.cloud은 LiveKit으로 라우팅됩니다.

---

## Slide 9 — 인증 서비스의 암호화 전략 (1.5분)

이제 주요 코드를 살펴보겠습니다.

auth-service의 핵심은 **KeyManager**입니다. RSA-2048 키 쌍을 생성한 후, 개인키를 절대 평문으로 저장하지 않습니다. AES-256-GCM으로 암호화할 때, 랜덤 Salt 16바이트와 IV 12바이트를 생성하고, PBKDF2로 60만 번 반복해 암호화 키를 유도합니다. 암호화된 결과는 Salt + IV + CipherText 구조로 결합해 Base64로 저장합니다.

이렇게 하면 데이터베이스가 유출되더라도 개인키를 복원할 수 없어, JWT 서명의 무결성이 보장됩니다.

---

## Slide 10 — 모든 요청의 보안 관문 역할 (1분)

Gateway는 **모든 요청의 보안 관문 역할**을 합니다.

CookieServerAuthenticationConverter는 HttpOnly 쿠키에서 accessToken을 추출해 BearerTokenAuthenticationToken으로 변환합니다.

UserHeaderFilter는 JWT 검증 후 claims에서 userId, email, roles를 추출해 X-User-Id 등의 헤더로 주입합니다. 덕분에 다운스트림 서비스는 JWT를 직접 다룰 필요가 없습니다.

SecureInternalFilter가 X-Gateway-Secret을 자동 주입하여, 내부 서비스는 Gateway를 통하지 않은 요청을 거부할 수 있습니다.

---

## Slide 11 — 화상회의 녹화 자동화 파이프라인 (1.5분)

**화상회의 녹화 자동화 파이프라인**입니다.

video-service는 LiveKit의 Webhook을 통해 이벤트를 수신합니다.

참가자가 마이크 트랙을 publish하면, 자동으로 해당 트랙의 오디오를 S3로 녹음하는 Egress를 시작합니다. 이것이 AI 파이프라인의 시작점입니다.

또한 LiveKit 서버와 통신하기 위한 JWT를 별도의 라이브러리 없이 javax.crypto.Mac으로 직접 HS256 서명을 구현했습니다.

---

## Slide 12 — 음성 → 텍스트 → 회의 요약 AI 파이프라인 (2분)

AI 파이프라인의 핵심, **음성에서 텍스트, 그리고 회의 요약까지 이어지는 파이프라인** 코드입니다.

SileroVadClient는 ONNX Runtime으로 Silero VAD v4 모델을 로드합니다. 16kHz 모노 PCM을 512 샘플(32ms) 단위로 처리하며, LSTM 상태를 유지하면서 각 윈도우의 음성 확률을 계산합니다.

SttWorkerService는 전체 파이프라인을 조율합니다. VAD가 감지한 발화 구간 중 100ms 미만은 노이즈로 판단해 건너뛰고, 나머지 구간만 OpenAI STT API에 전달합니다.

ClaudeSummarizerClient는 트랜스크립트를 Claude API에 보내되, 반드시 JSON 스키마에 맞는 출력만 받도록 프롬프트를 설계했습니다. 응답이 올바른 스키마인지 검증한 후에만 저장합니다.

---

## Slide 13 — 웹 + 모바일 실시간 알림 이중 채널 (1.5분)

**웹과 모바일을 동시에 커버하는 실시간 알림 이중 채널** 구조입니다.

첫째, **멀티탭 SSE 실시간 알림**입니다. ConcurrentHashMap을 이중으로 사용해서, 한 사용자가 여러 탭이나 디바이스에서 동시 접속해도 모든 연결에 알림이 전달됩니다. 30초마다 heartbeat를 보내 끊어진 연결은 자동으로 정리해서 메모리 누수를 방지합니다.

둘째, **FCM 모바일 푸시**입니다. 웹은 SSE, 모바일은 FCM으로 이중 채널 알림입니다. Auth 서비스의 최신 토큰과 로컬 DB 토큰을 Set으로 합쳐서 중복 없이 발송하고, 실패 시 최대 3회 자동 재시도합니다. 유효하지 않은 토큰은 자동 삭제됩니다.

셋째, **유저별 알림 설정 토글**입니다. 회의, 팀, 회의록 세 가지 카테고리를 유저가 개별적으로 ON/OFF할 수 있습니다. 알림 발송 전에 해당 타입의 토글이 OFF이면 발송 자체를 차단하는 구조입니다.

---

## Slide 14 — 왜 Go를 썼는지 + 보안까지 잡은 파일 서버 (1.5분)

**왜 Go를 선택했는지, 그리고 보안까지 잡은 파일 서버**입니다.

file-service는 성능을 위해 Go로 마이그레이션했으며, 보안에 특히 신경 썼습니다. 응답 속도는 Kotlin 450ms에서 Go 4ms로, 메모리는 JVM 512MB에서 Go 30MB로 대폭 개선되었습니다.

MIME 스푸핑 방어를 위해 Content-Type 헤더가 아닌 실제 파일 바이트를 검사합니다. S3 키 인젝션을 방어하기 위해 ownerType은 화이트리스트, ownerId는 숫자만 허용하며 경로 순회 문자를 차단합니다.

비동기 업로드는 Go의 채널을 세마포어로 활용해 동시 10개로 제한합니다. 파일 서빙은 1MB 미만은 인메모리 캐시에서, 이상은 S3에서 스트리밍합니다.

---

## Slide 15 — 안전한 이메일 발송 + OAuth2 토큰 효율적 관리 (1분)

**안전한 이메일 발송과 OAuth2 토큰의 효율적 관리** 전략입니다.

email-service는 간결하지만 보안 포인트가 있습니다.

Kafka에서 수신한 템플릿 이름을 화이트리스트로 검증해 경로 순회 공격을 방지합니다. 허용된 템플릿은 company-invitation, guest-invitation, temporary-password 세 가지뿐입니다.

Gmail OAuth2 토큰은 Double-checked locking 패턴으로 캐시하여, 55분마다 자동 갱신합니다. 실제 만료 시간(60분)보다 5분 일찍 갱신해 레이스 컨디션을 방지합니다.

---

## Slide 16 — 팀 구성 및 역할 (1분)

Onmeet을 만든 팀원들을 소개합니다.

- **최승은** — 팀장 / Full-Stack. 프로젝트 총괄, Video Service 개발, 화상회의/채팅/대기실, 프론트엔드 전반 개발을 담당했습니다.
- **박영진** — Backend Lead. MSA 전체 설계, Auth/Gateway/Notification 서비스, CI/CD 파이프라인 구축, Go 전환 최적화를 담당했습니다.
- **조예성** — Backend. Notification Service, Flyway 마이그레이션, 인프라 안정화, 모니터링 설정을 담당했습니다.
- **양진영** — AI Backend. AI Service 개발, STT 음성 인식, LLM 회의록 생성, AI 파이프라인 구축을 담당했습니다.

4명의 개발자가 7개의 마이크로서비스를 5개의 프로그래밍 언어로 개발했습니다.

---

## Slide 17 — Closing (1분)

정리하겠습니다.

Onmeet은 7개 마이크로서비스, 5개 프로그래밍 언어, 4개 데이터베이스, 그리고 VAD부터 STT, LLM까지 이어지는 AI 파이프라인을 갖춘 B2B 화상회의 플랫폼입니다.

화면의 QR 코드로 실제 서비스에 접속하실 수 있고, GitHub에서 백엔드, 프론트엔드, 이 발표 자료까지 모든 소스코드를 확인하실 수 있습니다.

발표를 경청해 주셔서 감사합니다. 질문 있으시면 말씀해 주세요.

---

## 발표 팁

- 각 슬라이드에서 핵심 키워드를 강조하며 발표
- 코드 슬라이드에서는 전체 코드를 읽지 말고 하이라이트된 부분만 설명
- 코드 슬라이드 상단의 한국어 제목을 먼저 언급하면 청중이 맥락을 잡기 쉬움
- 코드 슬라이드 서브 메뉴로 현재 어떤 서비스인지 시각적으로 확인 가능
- AI Pipeline 슬라이드가 기술적 하이라이트 — 여기서 청중의 관심을 끌기
- 보안 아키텍처에서는 "왜 이렇게 했는지" 동기를 설명하면 인상적
- 전체화면(F키)으로 발표하면 몰입도 향상
- 모바일에서도 스와이프로 슬라이드 이동 가능
