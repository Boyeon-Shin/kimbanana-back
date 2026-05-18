# Kimbanana

여러 사람이 동시에 슬라이드를 편집해도 충돌 없이 동기화되는 실시간 협업 프레젠테이션 백엔드.

[![Java](https://img.shields.io/badge/Java-21-007396?logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4-6DB33F?logo=spring-boot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![WebSocket](https://img.shields.io/badge/WebSocket-STOMP-010101)](https://stomp.github.io/)

- API 문서: [OpenAPI 명세](./src/main/resources/openapi/openapi.yaml)
- 배포 이력: 운영 후 종료

---

## 목차

- [한눈에 보기](#한눈에-보기)
- [핵심 기능](#핵심-기능)
- [시스템 아키텍처](#시스템-아키텍처)
- [기술 스택](#기술-스택)
- [프로젝트 구조](#프로젝트-구조)
- [실행 방법](#실행-방법)

---

## 한눈에 보기

협업 도구를 만들 때 가장 어려운 건 "동시에 같은 곳을 건드릴 때"의 동기화입니다.
Kimbanana는 슬라이드 단위로 채널을 쪼개고, 인증을 2단계로 분리하고, 변경 이력을 배치로 묶어 저장합니다.

| 항목 | 내용 |
|---|---|
| 프로젝트 유형 | 팀 프로젝트 (Team-KimBanana) |
| 본인 역할 | 팀 리드 · 실시간 동기화 모듈 · 인증/인가 모듈 |
| 개발 기간 | 2025.03 ~ 2025.11 |

---

## 핵심 기능

### 1. 실시간 슬라이드 협업 편집
- 한 슬라이드를 여러 명이 동시에 편집해도 모든 클라이언트가 즉시 동기화
- 사용자가 보고 있는 슬라이드 단위로 메시지 채널을 분리해, 다른 슬라이드를 보는 사용자에게는 불필요한 트래픽이 가지 않도록 설계

### 2. 다중 인증 (회원 / 게스트 / 소셜 로그인)
- 회원: JWT(Access 15분) + Refresh Token(14일)
- 게스트: 초대 링크로 로그인 없이 참여 (Redis TTL 8시간)
- 소셜: Google · GitHub OAuth2 → JWT 발급

### 3. 변경 이력 저장 & 복원
- 프레젠테이션 단위로 스냅샷 저장 (전체 슬라이드 묶음)
- 전체 복원 / 선택 슬라이드 부분 복원 지원
- 슬라이드 N개를 한 트랜잭션 + 한 쿼리(batchUpdate)로 처리

### 4. 이미지 업로드 & 자동 정리
- 슬라이드 이미지 / 프레젠테이션 썸네일 분리 업로드
- 썸네일은 자동으로 16:9 중앙 크롭 → 480×270 리사이징
- 만료된(180일 초과) 이미지를 매일 03:00 KST에 자동 삭제

### 5. 초대 링크
- 게스트가 회원가입 없이 참여 가능
- Redis 기반 토큰 (TTL 자동 만료)

---

## 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client (Web)                           │
└────────────────┬──────────────────────────┬─────────────────────┘
                 │                          │
          HTTPS (REST)              WSS (STOMP/WebSocket)
                 │                          │
┌────────────────▼──────────────────────────▼─────────────────────┐
│                       Spring Boot Application                   │
│                                                                 │
│  ┌──────────────────┐         ┌────────────────────────────┐    │
│  │ JwtAuthFilter    │         │ JwtHandshakeInterceptor    │    │
│  │ (REST 인증)      │         │ + StompAuthChannelInterceptor│   │
│  └────────┬─────────┘         └────────────┬───────────────┘    │
│           │                                │                    │
│  ┌────────▼────────────────────────────────▼───────────────┐    │
│  │                    Controller Layer                      │   │
│  │   Auth · Presentation(REST/WS) · Workspace · History     │   │
│  │            · Image · Invitation                          │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────▼─────────────────────────────────┐   │
│  │                     Service Layer                        │   │
│  │   비즈니스 로직 → ApplicationEventPublisher 로 이벤트 발행      │   │
│  └─────┬─────────────────────────────────────┬──────────────┘   │
│        │                                     │                  │
│        │                                     ▼                  │
│        │                       ┌────────────────────────┐       │
│        │                       │ PresentationEventListener│     │
│        │                       │  (@EventListener)       │      │
│        │                       │  → SimpMessagingTemplate│      │
│        │                       │  → /topic/...           │      │
│        │                       └─────────────────────────┘      │
│        ▼                                                        │
│  ┌─────────────────┐                                            │
│  │  Repository     │                                            │
│  │  (Spring JDBC)  │                                            │
│  └────┬──────┬─────┘                                            │
└───────│──────│──────────────────────────────────────────────────┘
        │      │
   ┌────▼──┐ ┌─▼─────────┐
   │ Postgres│   Redis   │
   │ (영속)   │ (세션/토큰) │
   └────────┘ └──────────┘
```

### 실시간 편집 메시지 흐름

```
[User A]          [Server]                       [User B / C / ...]
   │                  │                                 │
   │  SEND            │                                 │
   │  /app/slide.edit.presentation.{pid}.slide.{sid}    │
   ├─────────────────▶│                                 │
   │                  │ PresentationWSController        │
   │                  │   .editSlide()                  │
   │                  │      │                          │
   │                  │      ▼                          │
   │                  │ PresentationService             │
   │                  │   .updateSlide()                │
   │                  │      │                          │
   │                  │      ▼ publishEvent             │
   │                  │ PresentationEventListener       │
   │                  │   .handleSlideUpdated()         │
   │                  │      │ broadcast                │
   │                  │      ▼                          │
   │                  │ /topic/presentation.{pid}       │
   │                  │   .slide.{sid}    ──────────────▶│
   │                  │                                 │ (수신 & 렌더)
```

### 인증 흐름 (WebSocket 2단계)

```
1. HTTP Handshake
   Cookie: access_token=...
        │
        ▼
   JwtHandshakeInterceptor
   → 쿠키에서 토큰 검증
   → SessionAttributes 에 인증 정보 저장

2. STOMP CONNECT
   Authorization: Bearer ...
        │
        ▼
   StompAuthChannelInterceptor
   → SessionAttributes 인증 정보 확인
   → 없으면 헤더의 Bearer 토큰 검증
   → USER / GUEST 구분 (DB / Redis 조회)
   → 만료 게스트 거부
```

---

## 기술 스택

| 분류 | 기술 | 선택 이유 |
|---|---|---|
| 언어 | Java 21 | Record, Pattern Matching 등 최신 문법 활용 |
| 프레임워크 | Spring Boot 3.4 | 생태계 안정성 + WebSocket/Security 통합 |
| 영속성 | Spring JDBC + PostgreSQL | 명시적 SQL · JSONB 컬럼 활용 · 배치 처리에 유리 |
| 캐시/세션 | Redis | TTL 기반 게스트 세션 / 초대 토큰 |
| 실시간 통신 | WebSocket + STOMP | 구조화된 메시지 라우팅 (`/topic`, `/app`) |
| 인증 | JWT (HS512) + OAuth2 | 무상태 인증 + 소셜 로그인 |
| API 문서 | Springdoc OpenAPI 2.8 + 수기 spec | 클라이언트와 스키마 사전 합의 |
| 빌드 | Gradle 8 | |

---

## 프로젝트 구조

```
src/main/java/io/wisoft/kimbanana/
├── auth/                        # 회원 / 게스트 / OAuth 인증
│   ├── controller/
│   ├── service/                 # AuthService, UserService, GuestSessionService
│   ├── repository/
│   │   ├── jdbc/                # JdbcUserRepository
│   │   └── redis/               # RedisGuestSessionRepository
│   ├── entity/                  # User, GuestSession
│   └── dto/
│
├── presentation/                # 실시간 협업의 핵심 도메인
│   ├── controller/              # PresentationRestController
│   ├── websocket/               # PresentationWSController (@MessageMapping)
│   ├── service/                 # PresentationService, ActiveUserService
│   ├── event/                   # PresentationEvents (record)
│   ├── listener/                # PresentationEventListener (→ broadcast)
│   ├── repository/jdbc/
│   ├── entity/                  # Presentation, Slide
│   └── dto/
│
├── workspace/                   # 내 프레젠테이션 목록 관리
├── history/                     # 스냅샷 저장 / 복원 (batchUpdate + @Transactional)
├── image/                       # 업로드 / 리사이징 / 자동 정리 (Cron + parallelStream)
│   ├── service/                 # ImageService, StorageService, CleanupService
│   └── handler/                 # ImageCleanupScheduler
├── invitation/                  # 게스트 초대 링크 (Redis)
│
└── infrastructure/
    ├── config/                  # Security / WebSocket / Redis / Web 설정
    ├── security/
    │   ├── jwt/                 # 토큰 생성, REST 필터, WS Handshake / STOMP 인터셉터
    │   └── oauth/               # Google / GitHub OAuth2 핸들러
    └── exception/               # GlobalExceptionHandler
```

---

## 실행 방법

### 사전 요구사항
- JDK 21+
- PostgreSQL 15+
- Redis 7+

### 1. 환경 변수 설정

`src/main/resources/application.yml` 을 다음 형태로 생성:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/kimbanana
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}

jwt:
  secret: ${JWT_SECRET_BASE64}     # base64 인코딩된 시크릿
  access-validity: 900             # 초 단위 (15분)
  refresh-validity: 1209600        # 14일
  guest-validity: 28800            # 8시간
```

### 2. 빌드 & 실행

```bash
# 의존성 다운로드 & 빌드
./gradlew build

# 실행
./gradlew bootRun
```

기본 포트: `8080`
API 문서(Swagger UI): `http://localhost:8080/swagger-ui/index.html`

---

<div align="center">

**Team-KimBanana**

</div>
