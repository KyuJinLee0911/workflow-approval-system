# 시스템 아키텍처 설계 상세

> 관련 문서: design.md §7
> 아키텍처 스타일: 계층형 아키텍처 (Layered Architecture) - 단일 서비스 기준
> 기술 스택: Java 17, Spring Boot 3.x, Spring Data JPA, PostgreSQL, Spring Batch, JUnit 5, Testcontainers

---

## 목차

1. [아키텍처 개요](#1-아키텍처-개요)
2. [계층 구조 상세](#2-계층-구조-상세)
3. [패키지 구조](#3-패키지-구조)
4. [기술 스택 선택 근거](#4-기술-스택-선택-근거)
5. [트랜잭션 경계 설계](#5-트랜잭션-경계-설계)
6. [감사 로그 처리 방식](#6-감사-로그-처리-방식)
7. [배치 처리 아키텍처](#7-배치-처리-아키텍처)
8. [도메인 설계 원칙](#8-도메인-설계-원칙)
9. [확장 포인트](#9-확장-포인트)

---

## 1. 아키텍처 개요

### 1.1 전체 구조

```
┌─────────────────────────────────────────────────┐
│                  Client                          │
│              (Postman / Browser)                 │
└──────────────────────┬──────────────────────────┘
                       │ HTTP/REST
┌──────────────────────▼──────────────────────────┐
│           Spring Boot Application               │
│  ┌─────────────────────────────────────────┐    │
│  │         Presentation Layer              │    │
│  │  Controllers, DTOs, GlobalExceptionHandler   │
│  └─────────────────────┬───────────────────┘    │
│                        │                         │
│  ┌─────────────────────▼───────────────────┐    │
│  │         Application Layer               │    │
│  │  Services, 트랜잭션 경계, 권한 검증        │    │
│  └─────────────────────┬───────────────────┘    │
│                        │                         │
│  ┌─────────────────────▼───────────────────┐    │
│  │           Domain Layer                  │    │
│  │  Entities, 상태 전이 로직, 도메인 예외     │    │
│  └─────────────────────┬───────────────────┘    │
│                        │                         │
│  ┌─────────────────────▼───────────────────┐    │
│  │        Infrastructure Layer             │    │
│  │  Repositories (JPA), AuditLog, Batch    │    │
│  └─────────────────────┬───────────────────┘    │
│                        │                         │
│  ┌─────────────────────▼───────────────────┐    │
│  │           Spring Batch                  │    │
│  │  ExpiredRequestJob, EscalationJob,       │    │
│  │  StatisticsJob                          │    │
│  └─────────────────────┬───────────────────┘    │
└────────────────────────┼────────────────────────┘
                         │
┌────────────────────────▼────────────────────────┐
│              PostgreSQL                          │
│  approval DB + Spring Batch 메타 테이블           │
└─────────────────────────────────────────────────┘
```

### 1.2 아키텍처 선택 근거

**계층형 아키텍처 선택 이유**:
- 포트폴리오 목적상 설계 의도가 명확하고 설명 가능해야 함
- 계층 책임이 명확해 면접에서 "이 로직이 왜 여기 있나요?" 질문에 답하기 쉬움
- 단일 서비스 기준에서 MSA나 헥사고날보다 단순하고 직관적
- Spring Boot + JPA와 자연스럽게 결합

**헥사고날 아키텍처를 선택하지 않은 이유**:
- 포트/어댑터 구조는 단일 서비스 규모에서 보일러플레이트가 과도하게 늘어남
- 추상화 레이어가 증가해 코드 추적이 어려워짐
- 면접에서 "왜 헥사고날인가?" 설명 부담이 있음

---

## 2. 계층 구조 상세

### 2.1 Presentation Layer (프레젠테이션 계층)

**책임**:
- HTTP 요청 수신 및 DTO 변환
- 입력값 기본 유효성 검증 (`@Valid`)
- HTTP 응답 반환
- `GlobalExceptionHandler`로 예외를 HTTP 응답으로 변환

**포함 컴포넌트**:
- `@RestController` 클래스들
- Request/Response DTO 클래스들
- `GlobalExceptionHandler`
- `ApiResponse<T>`, `ErrorResponse` 래퍼 클래스

**담당하지 않는 것**:
- 비즈니스 로직
- 권한 소유권 검증 (역할 검증은 @PreAuthorize로 선언적 처리)
- 상태 전이

### 2.2 Application Layer (애플리케이션 계층)

**책임**:
- **트랜잭션 경계 관리** (`@Transactional`)
- **소유권 검증** (비즈니스 컨텍스트가 필요한 3단계 권한 검증)
- 도메인 메서드 호출 (상태 전이 실행은 Domain Layer에 위임)
- 감사 로그 기록 (`auditLogService.record()`)
- Domain 객체와 Repository 조합

**포함 컴포넌트**:
- `ApprovalRequestService`: 결재 문서 상태 변경 액션
- `ApprovalQueryService`: 결재 문서 조회 전용 (CQRS 분리)
- `AuditLogService`: 감사 로그 기록
- `UserService`: 사용자 관리

**담당하지 않는 것**:
- 상태 전이 규칙 (Domain Layer 책임)
- DB 쿼리 직접 작성 (Infrastructure Layer 책임)

**CQRS 분리 이유**: `ApprovalRequestService`와 `ApprovalQueryService`를 분리한다. 조회 로직은 쓰기 로직과 달리 트랜잭션이 읽기 전용(`@Transactional(readOnly = true)`)이고, 복잡한 조인 쿼리가 필요하다. 두 관심사를 분리하면 서비스 클래스가 과도하게 비대해지는 것을 방지한다.

### 2.3 Domain Layer (도메인 계층)

**책임**:
- **상태 전이 로직 응집** (핵심)
- 비즈니스 규칙 검증 (결재선 필수, 반려 코멘트 필수 등)
- 도메인 예외 발생
- 엔티티 간 연관관계 관리

**포함 컴포넌트**:
- JPA 엔티티 클래스 (상태 전이 메서드 포함)
- Enum 클래스 (ApprovalStatus, StepStatus, ApprovalCategory 등)
- 도메인 예외 클래스 계층

**담당하지 않는 것**:
- DB 접근 (Repository 호출 없음)
- HTTP 관련 로직
- Spring Bean으로 등록되지 않음 (순수 Java 객체)

### 2.4 Infrastructure Layer (인프라 계층)

**책임**:
- 영속성 처리 (JPA Repository)
- 외부 시스템 연동 (확장 시)
- Spring Batch Job 구성 및 실행

**포함 컴포넌트**:
- `ApprovalRequestRepository`, `ApprovalStepRepository` 등
- `AuditLogRepository`
- Batch Job 설정 클래스

---

## 3. 패키지 구조

```
src/main/java/com/example/approval/
│
├── ApprovalApplication.java                        # 메인 클래스
│
├── common/                                         # 공통 모듈
│   ├── exception/
│   │   ├── ApprovalException.java                  # 기반 예외 (abstract)
│   │   ├── InvalidStateTransitionException.java
│   │   ├── ApprovalAuthorizationException.java
│   │   ├── ApprovalValidationException.java
│   │   ├── ApprovalNotFoundException.java
│   │   └── ErrorCode.java                          # 에러 코드 Enum
│   │
│   ├── response/
│   │   ├── ApiResponse.java                        # 공통 성공 응답 래퍼
│   │   └── ErrorResponse.java                      # 에러 응답 구조
│   │
│   ├── handler/
│   │   └── GlobalExceptionHandler.java             # @RestControllerAdvice
│   │
│   └── audit/
│       ├── AuditAction.java                        # 감사 액션 Enum
│       ├── Auditable.java                          # AOP 마커 어노테이션
│       └── AuditLogAspect.java                     # AOP 구현체
│
├── domain/
│   ├── user/
│   │   ├── entity/
│   │   │   ├── User.java
│   │   │   ├── Department.java
│   │   │   ├── UserRole.java
│   │   │   └── enums/
│   │   │       ├── UserStatus.java
│   │   │       └── RoleType.java
│   │   ├── repository/
│   │   │   ├── UserRepository.java
│   │   │   └── DepartmentRepository.java
│   │   ├── service/
│   │   │   └── UserService.java
│   │   └── controller/
│   │       └── UserController.java
│   │
│   ├── approval/
│   │   ├── entity/
│   │   │   ├── ApprovalRequest.java                # Aggregate Root
│   │   │   ├── ApprovalLine.java
│   │   │   ├── ApprovalStep.java
│   │   │   ├── ApprovalHistory.java
│   │   │   └── enums/
│   │   │       ├── ApprovalStatus.java
│   │   │       ├── StepStatus.java
│   │   │       ├── HistoryAction.java
│   │   │       └── ApprovalCategory.java
│   │   ├── repository/
│   │   │   ├── ApprovalRequestRepository.java
│   │   │   └── ApprovalStepRepository.java
│   │   ├── service/
│   │   │   ├── ApprovalRequestService.java         # 쓰기 서비스
│   │   │   └── ApprovalQueryService.java           # 읽기 서비스
│   │   └── controller/
│   │       └── ApprovalRequestController.java
│   │
│   └── auditlog/
│       ├── entity/
│       │   └── AuditLog.java
│       ├── repository/
│       │   └── AuditLogRepository.java
│       ├── service/
│       │   └── AuditLogService.java
│       └── controller/
│           └── AuditLogController.java             # ADMIN 전용
│
├── batch/                                          # Spring Batch
│   ├── job/
│   │   ├── ExpiredRequestJobConfig.java            # 만료 처리 배치
│   │   ├── EscalationJobConfig.java                # 에스컬레이션 배치
│   │   └── StatisticsJobConfig.java                # 통계 집계 배치
│   ├── tasklet/
│   │   └── OverdueReminderTasklet.java             # 리마인드 Tasklet
│   ├── reader/
│   │   └── OverdueRequestItemReader.java
│   ├── processor/
│   │   └── ExpiredRequestItemProcessor.java
│   └── writer/
│       └── ApprovalRequestItemWriter.java
│
├── security/
│   ├── JwtTokenProvider.java                       # JWT 생성/검증
│   ├── JwtAuthenticationFilter.java                # 요청 인터셉트
│   ├── JwtAuthenticationEntryPoint.java            # 401 처리
│   ├── JwtAccessDeniedHandler.java                 # 403 처리
│   ├── CustomUserDetailsService.java               # UserDetailsService 구현
│   ├── UserPrincipal.java                          # 인증된 사용자 정보
│   └── SecurityConfig.java
│
└── config/
    ├── JpaConfig.java                              # JPA 감사 설정
    ├── BatchConfig.java                            # Batch 공통 설정
    └── SwaggerConfig.java                          # OpenAPI 설정

src/main/resources/
├── application.yml                                 # 기본 설정
├── application-local.yml                           # 로컬 환경 설정
└── db/migration/
    ├── V1__create_initial_schema.sql
    ├── V2__create_indexes.sql
    └── V3__insert_initial_data.sql

src/test/java/com/example/approval/
├── domain/
│   ├── approval/
│   │   ├── entity/
│   │   │   └── ApprovalRequestTest.java            # 상태 전이 단위 테스트
│   │   ├── repository/
│   │   │   └── ApprovalRequestRepositoryTest.java  # Testcontainers 통합 테스트
│   │   └── service/
│   │       └── ApprovalRequestServiceTest.java     # 권한/비즈니스 규칙 테스트
│   └── auditlog/
│       └── AuditLogServiceTest.java
│
├── batch/
│   ├── ExpiredRequestJobTest.java                  # Testcontainers 배치 테스트
│   └── EscalationJobTest.java
│
└── controller/
    └── ApprovalRequestControllerTest.java          # MockMvc API 테스트
```

---

## 4. 기술 스택 선택 근거

### 4.1 핵심 기술

| 기술 | 버전 | 선택 이유 | 대안 대비 장점 |
|------|------|----------|--------------|
| Java | 17 LTS | 최신 LTS, Record, TextBlock, Sealed Class 지원 | Java 11 대비 언어 기능 풍부 |
| Spring Boot | 3.x | 자동 설정, Spring Security 6.x, 생산성 | 설정 최소화로 비즈니스 로직 집중 가능 |
| Spring Data JPA | Hibernate 6 | 선언적 쿼리, 엔티티 매핑, 낙관적 락 지원 | MyBatis 대비 도메인 모델 표현력 우수 |
| Spring Security | 6.x | JWT 인증, @PreAuthorize, 선언적 권한 제어 | 별도 구현 대비 검증된 보안 |
| Spring Batch | 5.x | 재실행 가능한 배치, 청크 처리, 실행 이력 관리 | Quartz만으로는 청크 처리, 재시작 지원 미흡 |
| PostgreSQL | 15+ | JSONB, 부분 인덱스, 트랜잭션 안정성 | MySQL 대비 JSON 처리 및 고급 인덱스 기능 우수 |
| Flyway | 9.x | SQL 마이그레이션 이력 관리, 팀 환경 일관성 | Liquibase 대비 단순한 SQL 파일 기반 |
| JUnit 5 | 5.x | 조건부 테스트, 파라미터화 테스트, 확장 모델 | JUnit 4 대비 표현력 우수 |
| Testcontainers | 1.x | 실제 PostgreSQL로 통합 테스트, CI/CD 친화적 | H2 인메모리 대비 실제 DB 동작 보장 |

### 4.2 의도적으로 제외한 기술

| 기술 | 제외 이유 |
|------|----------|
| Redis | MVP에서 캐싱 필요성 낮음. 확장 시 결재 처리 상태 캐싱에 추가 가능 |
| Kafka/RabbitMQ | 알림 시스템이 MVP 범위 외. 이벤트 발행은 로그 기록으로 대체 |
| Elasticsearch | 감사 로그 검색은 PostgreSQL 인덱스로 충분. 대용량 시 확장 가능 |
| Docker Swarm/K8s | 단일 서비스 기준. Docker Compose로 로컬 환경만 구성 |
| QueryDSL | Spring Data JPA의 Specification + JPQL로 MVP 수준 커버 가능 |

---

## 5. 트랜잭션 경계 설계

### 5.1 트랜잭션 경계 원칙

- **Service 메서드 = 트랜잭션 단위**
- 하나의 사용자 액션은 하나의 트랜잭션으로 완성
- 트랜잭션 내에서 AuditLog 저장도 함께 진행 (감사 로그 누락 방지)

### 5.2 주요 트랜잭션 경계

| 메서드 | 트랜잭션 유형 | 포함 범위 |
|--------|-------------|---------|
| `ApprovalQueryService.findById()` | `@Transactional(readOnly = true)` | 결재 문서 + 결재선 + 단계 조회 |
| `ApprovalRequestService.submit()` | `@Transactional` | ApprovalRequest 상태 변경 + ApprovalStep 상태 변경 + AuditLog 저장 |
| `ApprovalRequestService.approve()` | `@Transactional` | ApprovalStep 상태 변경 + ApprovalHistory 생성 + ApprovalRequest 상태 변경 + AuditLog 저장 |
| `ApprovalRequestService.reject()` | `@Transactional` | 위와 동일 |
| `ApprovalRequestService.withdraw()` | `@Transactional` | ApprovalRequest 상태 변경 + ApprovalStep 초기화 + AuditLog 저장 |
| `Batch ExpiredRequestJob` | 청크 단위 트랜잭션 | 청크 크기(100)마다 커밋 |

### 5.3 AuditLog 트랜잭션 전략

```java
// 선택 1 (MVP 채택): 동일 트랜잭션 - 감사 로그 실패 = 전체 롤백
@Transactional
public void approve(Long requestId, ...) {
    ...
    request.onStepApproved(step);
    auditLogService.record(...);  // 동일 트랜잭션 내
}

// 선택 2 (확장 시): AFTER_COMMIT 훅 - 감사 로그 실패 = 비즈니스 롤백 없음
// TransactionSynchronizationManager.registerSynchronization(
//     new TransactionSynchronizationAdapter() {
//         @Override
//         public void afterCommit() { auditLogService.record(...); }
//     }
// );
```

**MVP 결정**: 동일 트랜잭션 내 감사 로그 저장을 채택한다. 감사 로그 저장 실패가 비즈니스 트랜잭션을 롤백시키는 것이 "더 안전한" 방향이다. 감사 로그가 누락된 채 비즈니스 액션만 성공하는 것은 컴플라이언스 관점에서 더 큰 문제다.

---

## 6. 감사 로그 처리 방식

### 6.1 AOP + 명시적 호출 혼용 전략

```
중요 비즈니스 액션 (제출, 승인, 반려, 회수) → Service에서 명시적 auditLogService.record() 호출
공통 메타데이터 (IP, UserAgent, 시각) → AuditLogAspect에서 자동 수집
```

### 6.2 AuditLogAspect 역할

```java
@Aspect
@Component
public class AuditLogAspect {

    // HTTP 요청 메타데이터 자동 수집
    @Around("@annotation(Auditable)")
    public Object captureRequestMetadata(ProceedingJoinPoint joinPoint) throws Throwable {
        // IP, UserAgent를 RequestContextHolder에서 추출
        // AuditLogContext (ThreadLocal)에 저장
        Object result = joinPoint.proceed();
        // AuditLogContext 정리
        return result;
    }
}
```

### 6.3 AuditLogService 역할

```java
@Service
@Transactional(propagation = Propagation.MANDATORY)  // 반드시 기존 트랜잭션 내에서만 실행
public class AuditLogService {

    public void record(AuditAction action, ApprovalRequest request, Long actorId, String comment) {
        User actor = userRepository.findById(actorId).orElseThrow();

        AuditLog log = AuditLog.builder()
            .entityType("ApprovalRequest")
            .entityId(request.getId())
            .action(action.name())
            .actorId(actorId)
            .actorName(actor.getName())         // 비정규화
            .beforeState(serializeBeforeState(request))
            .afterState(serializeAfterState(request))
            .ipAddress(AuditLogContext.getIpAddress())  // AOP에서 설정
            .userAgent(AuditLogContext.getUserAgent())
            .occurredAt(LocalDateTime.now())
            .build();

        auditLogRepository.save(log);
    }
}
```

---

## 7. 배치 처리 아키텍처

### 7.1 Spring Batch 선택 근거

| 요구사항 | Spring Batch 지원 |
|---------|-----------------|
| 대량 데이터 처리 | 청크 기반 처리 (chunkSize 설정) |
| 실패 시 재시작 | JobExecution 이력 기반 재시작 지점 관리 |
| 중복 실행 방지 | JobInstance + JobParameters로 자동 관리 |
| 실행 이력 조회 | BATCH_JOB_EXECUTION 테이블 자동 관리 |
| 단계별 처리 | Step → Reader → Processor → Writer 구조 |

### 7.2 배치 Job 구조 예시 (ExpiredRequestJob)

```
ExpiredRequestJob
└── expireStep
    ├── Reader: JpaPagingItemReader
    │   └── SELECT * FROM approval_requests
    │       WHERE status IN ('PENDING', 'IN_PROGRESS', 'ESCALATED')
    │       AND submitted_at < now() - interval '30 days'
    │       ORDER BY id  ← 페이징 안정성을 위해 정렬 필수
    │
    ├── Processor: ExpiredRequestItemProcessor
    │   └── request.expire() 호출 (도메인 메서드)
    │
    └── Writer: ApprovalRequestItemWriter
        ├── JpaItemWriter (상태 저장)
        └── AuditLogService.record (감사 로그) - 배치 전용 처리
```

### 7.3 배치 실행 스케줄

```yaml
# application.yml
batch:
  jobs:
    expired-request:
      cron: "0 0 0 * * *"     # 매일 00:00
    escalation:
      cron: "0 0 9 * * *"     # 매일 09:00
    overdue-reminder:
      cron: "0 0 9 * * *"     # 매일 09:00
    statistics:
      cron: "0 0 1 * * *"     # 매일 01:00
```

---

## 8. 도메인 설계 원칙

### 8.1 Aggregate 경계

**ApprovalRequest Aggregate**:
```
ApprovalRequest (Aggregate Root)
├── ApprovalLine
│   └── List<ApprovalStep>
│       └── ApprovalHistory (optional)
```

**Aggregate 외부에서의 접근 규칙**:
- 외부에서 ApprovalLine, ApprovalStep, ApprovalHistory에 직접 접근하지 않는다.
- 반드시 ApprovalRequest를 통해서만 접근한다.
- Repository도 `ApprovalRequestRepository`만 외부에 노출한다. (`ApprovalLineRepository`는 불필요)

**예외**: `ApprovalStepRepository`는 "내 결재 목록 조회" (`status = IN_PROGRESS AND approver_id = ?`) 쿼리를 위해 별도 제공한다. 조회 목적이므로 Aggregate 경계 원칙의 예외로 허용.

### 8.2 도메인 객체의 순수성 유지

도메인 엔티티에는 다음을 포함하지 않는다:
- Spring Bean 의존성 (`@Autowired`, `@Service` 등)
- Repository 호출
- HTTP 관련 로직

이를 통해 도메인 객체의 상태 전이 테스트를 Spring Context 없이 순수 Java로 작성할 수 있다.

---

## 9. 확장 포인트

### 9.1 단기 확장 (코드 변경 최소)

| 확장 | 방법 |
|------|------|
| 결재선 템플릿 | ApprovalLineTemplate 엔티티 추가 |
| 알림 기능 | ApplicationEventPublisher로 도메인 이벤트 발행, 이벤트 리스너에서 알림 처리 |
| 첨부파일 | ApprovalAttachment 엔티티 추가, S3 연동 |

### 9.2 중장기 확장 (아키텍처 변경 필요)

| 확장 | 방법 |
|------|------|
| 알림 서비스 분리 | 이벤트 기반 → Kafka 도입, 알림 마이크로서비스 분리 |
| 감사 로그 분리 | audit_logs → Elasticsearch 이관 (검색 성능 향상) |
| 병렬 결재 | ApprovalStep에 parallel_group 필드 추가, 처리 로직 변경 |
| 외부 HR 연동 | User 정보를 외부 HR 시스템에서 동기화하는 배치 추가 |
| SSO 연동 | Spring Security OAuth2 Resource Server 설정 변경 |
