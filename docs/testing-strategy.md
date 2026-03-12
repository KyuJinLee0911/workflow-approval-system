# 테스트 전략

> 관련 문서: design.md §13, state-transition.md §7, architecture.md
> 기술 스택: JUnit 5, Mockito, Spring Boot Test, MockMvc, Testcontainers, Spring Batch Test

---

## 목차

1. [테스트 전략 개요](#1-테스트-전략-개요)
2. [테스트 계층 분류](#2-테스트-계층-분류)
3. [상태 전이 테스트 전략](#3-상태-전이-테스트-전략)
4. [권한 테스트 전략](#4-권한-테스트-전략)
5. [예외 테스트 전략](#5-예외-테스트-전략)
6. [감사 로그 테스트 전략](#6-감사-로그-테스트-전략)
7. [배치 테스트 전략](#7-배치-테스트-전략)
8. [Testcontainers 활용 포인트](#8-testcontainers-활용-포인트)
9. [테스트 작성 원칙](#9-테스트-작성-원칙)
10. [테스트 실행 방법](#10-테스트-실행-방법)

---

## 1. 테스트 전략 개요

이 프로젝트의 테스트는 **상태 전이와 권한 제어의 정확성**을 최우선으로 검증한다. 두 영역이 이 시스템의 핵심 비즈니스 규칙이기 때문이다.

**기본 원칙**:
- 정상 케이스와 실패 케이스를 반드시 함께 작성한다. 성공 케이스만 있는 테스트는 완성된 테스트가 아니다.
- 상태 전이 테스트는 Spring Context 없이 순수 단위 테스트로 작성한다. 이것이 도메인 객체에 로직을 응집시킨 이유다.
- 통합 테스트는 Testcontainers로 실제 PostgreSQL을 사용한다. H2 인메모리 DB는 사용하지 않는다.

---

## 2. 테스트 계층 분류

| 계층 | 대상 | 사용 도구 | 외부 의존성 | 실행 속도 |
|------|------|----------|-----------|----------|
| 단위 테스트 | 도메인 엔티티 (상태 전이, 소유권 메서드) | JUnit 5 | 없음 | 빠름 |
| 서비스 테스트 | Service 계층 (권한, 비즈니스 흐름) | JUnit 5, Mockito | Repository 모킹 | 빠름 |
| Repository 통합 테스트 | Repository + 실제 DB | JUnit 5, Testcontainers | PostgreSQL | 느림 |
| API 테스트 | Controller + 전체 레이어 | MockMvc, Testcontainers | PostgreSQL | 느림 |
| 배치 테스트 | Batch Job 전체 플로우 | Spring Batch Test, Testcontainers | PostgreSQL | 느림 |

### 테스트 패키지 구조

```
src/test/java/com/example/approval/
├── domain/
│   ├── approval/
│   │   ├── ApprovalRequestTest.java        # 단위: 상태 전이
│   │   └── ApprovalLineTest.java           # 단위: 결재선 로직
│   └── user/
│       └── UserTest.java
├── service/
│   ├── ApprovalRequestServiceTest.java     # 서비스: 권한 + 비즈니스
│   └── AuditLogServiceTest.java
├── repository/
│   └── ApprovalRequestRepositoryTest.java  # 통합: Testcontainers
├── api/
│   └── ApprovalRequestApiTest.java         # API: MockMvc + Testcontainers
└── batch/
    ├── ExpiredRequestJobTest.java           # 배치: 만료 처리
    ├── EscalationJobTest.java
    └── OverdueReminderJobTest.java
```

---

## 3. 상태 전이 테스트 전략

**Spring Context 없이 순수 Java 단위 테스트로 작성한다.** 상태 전이 로직이 도메인 객체에 응집되어 있으므로 가능하다.

### 3.1 허용 전이 테스트

각 허용 전이마다 성공 케이스를 작성한다.

```java
class ApprovalRequestTest {

    @Test
    void submit_정상제출_DRAFT에서PENDING으로전이() {
        // given: 결재선이 구성된 DRAFT 문서
        ApprovalRequest request = 결재선포함_초안문서_생성();

        // when
        request.submit();

        // then
        assertThat(request.getStatus()).isEqualTo(ApprovalStatus.PENDING);
        assertThat(request.getSubmittedAt()).isNotNull();
    }

    @Test
    void approve_마지막단계승인_IN_PROGRESS에서APPROVED로전이() { ... }

    @Test
    void approve_중간단계승인_PENDING에서IN_PROGRESS로전이() { ... }

    @Test
    void reject_반려처리_코멘트포함_REJECTED로전이() { ... }

    @Test
    void withdraw_회수_PENDING에서WITHDRAWN으로전이() { ... }

    @Test
    void resubmit_재제출_REJECTED에서PENDING으로전이() { ... }

    @Test
    void resubmit_재제출_WITHDRAWN에서PENDING으로전이() { ... }

    @Test
    void escalate_에스컬레이션_PENDING에서ESCALATED로전이() { ... }

    @Test
    void expire_만료처리_PENDING에서EXPIRED로전이() { ... }
}
```

### 3.2 금지 전이 테스트

각 금지 전이마다 `InvalidStateTransitionException`이 발생하는지 검증한다.

```java
@Test
void submit_APPROVED상태에서제출시도_예외발생() {
    ApprovalRequest request = 승인완료_문서_생성();
    assertThatThrownBy(() -> request.submit())
        .isInstanceOf(InvalidStateTransitionException.class);
}

@Test
void withdraw_IN_PROGRESS상태에서회수시도_예외발생() { ... }

@Test
void submit_EXPIRED상태에서제출시도_예외발생() { ... }
```

### 3.3 경계 조건 테스트

```java
@Test
void submit_결재선없이제출시도_유효성예외발생() {
    ApprovalRequest request = 결재선없는_초안문서_생성();
    assertThatThrownBy(() -> request.submit())
        .isInstanceOf(ApprovalValidationException.class)
        .hasMessageContaining(ErrorCode.APPROVAL_LINE_REQUIRED.getCode());
}

@Test
void reject_코멘트없이반려시도_유효성예외발생() { ... }

@Test
void withdraw_처리된단계있을때회수시도_예외발생() { ... }

@Test
void expire_이미만료된문서에expire호출_멱등적종료() {
    ApprovalRequest request = 만료된_문서_생성();
    // 예외 없이 정상 종료해야 함
    assertThatCode(() -> request.expire()).doesNotThrowAnyException();
    assertThat(request.getStatus()).isEqualTo(ApprovalStatus.EXPIRED);
}
```

### 3.4 목표 테스트 케이스 수

상태 전이 단위 테스트: **최소 20개** (허용 전이 10개 + 금지 전이 6개 + 경계 조건 4개)

---

## 4. 권한 테스트 전략

Service 계층 테스트에서 Repository를 Mockito로 모킹하여 권한 검증 로직을 단독 검증한다.

### 4.1 역할별 허용/거부 케이스

```java
class ApprovalRequestServiceTest {

    @Mock ApprovalRequestRepository requestRepository;
    @Mock AuditLogService auditLogService;
    @InjectMocks ApprovalRequestService service;

    @Test
    void submit_기안자본인_정상제출() {
        // given
        Long actorId = 1L;
        ApprovalRequest request = 기안자ID가_actorId인_DRAFT문서();
        when(requestRepository.findById(anyLong())).thenReturn(Optional.of(request));

        // when / then: 예외 없이 정상 실행
        assertThatCode(() -> service.submit(request.getId(), actorId))
            .doesNotThrowAnyException();
    }

    @Test
    void submit_타인이제출시도_권한예외발생() {
        Long actorId = 99L; // 기안자 아님
        ApprovalRequest request = 기안자ID가_1L인_DRAFT문서();
        when(requestRepository.findById(anyLong())).thenReturn(Optional.of(request));

        assertThatThrownBy(() -> service.submit(request.getId(), actorId))
            .isInstanceOf(ApprovalAuthorizationException.class)
            .extracting("errorCode").isEqualTo(ErrorCode.NOT_REQUEST_OWNER);
    }

    @Test
    void approve_담당결재자_정상승인() { ... }

    @Test
    void approve_타인이승인시도_권한예외발생() { ... }

    @Test
    void approve_현재단계아닌결재자_권한예외발생() { ... }

    @Test
    void withdraw_기안자본인_PENDING상태_정상회수() { ... }

    @Test
    void withdraw_타인이회수시도_권한예외발생() { ... }
}
```

### 4.2 소유권 검증 케이스

| 테스트 케이스 | 검증 포인트 | 기대 예외 |
|-------------|-----------|---------|
| 기안자 본인 제출 | isDraftedBy() true | 없음 |
| 타인 제출 시도 | isDraftedBy() false | ApprovalAuthorizationException(NOT_REQUEST_OWNER) |
| 담당 결재자 승인 | isAssignedTo() true + isCurrentlyProcessable() true | 없음 |
| 미담당 결재자 승인 시도 | isAssignedTo() false | ApprovalAuthorizationException(NOT_ASSIGNED_APPROVER) |
| 현재 단계 아닌 결재자 | isCurrentlyProcessable() false | ApprovalAuthorizationException(NOT_CURRENT_STEP_APPROVER) |

---

## 5. 예외 테스트 전략

비즈니스 예외와 시스템 예외를 구분하여 테스트한다.

### 5.1 비즈니스 예외 단위 테스트

도메인 객체 레벨에서 발생하는 예외를 단위 테스트로 검증한다.

```java
// 올바른 예외 타입, ErrorCode, 메시지를 함께 검증
assertThatThrownBy(() -> request.submit())
    .isInstanceOf(InvalidStateTransitionException.class)
    .satisfies(ex -> {
        InvalidStateTransitionException e = (InvalidStateTransitionException) ex;
        assertThat(e.getFromStatus()).isEqualTo(ApprovalStatus.APPROVED);
        assertThat(e.getToStatus()).isEqualTo(ApprovalStatus.PENDING);
    });
```

### 5.2 GlobalExceptionHandler API 테스트

MockMvc로 예외가 올바른 HTTP 상태코드와 에러 응답 포맷으로 변환되는지 검증한다.

```java
@Test
void submit_APPROVED상태에서제출시도_409반환() throws Exception {
    // given: APPROVED 상태 문서 준비, submit 호출 세팅

    mockMvc.perform(post("/api/v1/approval-requests/{id}/submit", requestId)
            .header("Authorization", "Bearer " + token))
        .andExpect(status().isConflict())  // 409
        .andExpect(jsonPath("$.success").value(false))
        .andExpect(jsonPath("$.error.code").value("INVALID_STATE_TRANSITION"));
}

@Test
void approve_담당아닌결재자_403반환() throws Exception { ... }

@Test
void submit_결재선없음_400반환() throws Exception { ... }

@Test
void getRequest_존재하지않는문서_404반환() throws Exception { ... }
```

### 5.3 예외별 HTTP 상태코드 검증 매트릭스

| 예외 클래스 | HTTP 상태코드 | 테스트 방식 |
|-----------|-------------|-----------|
| InvalidStateTransitionException | 409 Conflict | MockMvc |
| ApprovalAuthorizationException | 403 Forbidden | MockMvc |
| ApprovalValidationException | 400 Bad Request | MockMvc |
| ApprovalNotFoundException | 404 Not Found | MockMvc |
| AccessDeniedException (Spring Security) | 403 Forbidden | MockMvc |
| AuthenticationException (Spring Security) | 401 Unauthorized | MockMvc |

---

## 6. 감사 로그 테스트 전략

### 6.1 기록 여부 검증

주요 액션 수행 후 `AuditLog`가 실제로 저장됐는지 검증한다.

```java
@Test
void submit_감사로그_기록여부_검증() {
    // Service 테스트 (Mockito)
    service.submit(requestId, actorId);

    verify(auditLogService, times(1))
        .record(eq(AuditAction.SUBMIT), any(ApprovalRequest.class), eq(actorId));
}
```

### 6.2 기록 내용 정확성 검증

통합 테스트에서 실제 DB에 저장된 AuditLog의 내용을 검증한다.

```java
@Test
void submit_감사로그_내용정확성_검증(Testcontainers) {
    // given: DRAFT 문서 생성, submit 호출
    service.submit(requestId, actorId);

    // then: DB에서 AuditLog 조회 후 내용 검증
    AuditLog log = auditLogRepository.findLatestByEntityId(requestId);
    assertThat(log.getAction()).isEqualTo("SUBMIT");
    assertThat(log.getActorId()).isEqualTo(actorId);
    assertThat(log.getActorName()).isEqualTo("김철수");  // 비정규화 이름 확인
    assertThat(log.getEntityType()).isEqualTo("ApprovalRequest");
    assertThat(log.getAfterState()).isEqualTo("PENDING");
}
```

### 6.3 감사 로그 테스트 케이스 목록

| 액션 | 검증 포인트 |
|------|-----------|
| SUBMIT | 로그 존재 여부, actorId, afterState=PENDING |
| APPROVE | 로그 존재 여부, actorId, afterState 정확성 |
| REJECT | 로그 존재 여부, comment 포함 여부 |
| WITHDRAW | 로그 존재 여부, afterState=WITHDRAWN |
| EXPIRE (배치) | actorId=SYSTEM으로 기록 여부 |

---

## 7. 배치 테스트 전략

Spring Batch Test + Testcontainers로 실제 DB와 함께 배치 Job 전체 플로우를 검증한다.

### 7.1 멱등성 테스트

```java
@SpringBatchTest
@Testcontainers
class ExpiredRequestJobTest {

    @Test
    void 만료대상문서_EXPIRED로전환() throws Exception {
        // given: 30일 이상 경과한 PENDING 문서 3건 준비
        JobExecution execution = jobLauncherTestUtils.launchJob(jobParameters);
        assertThat(execution.getStatus()).isEqualTo(BatchStatus.COMPLETED);

        // then: 3건 모두 EXPIRED 상태 확인
        List<ApprovalRequest> expired = requestRepository.findByStatus(ApprovalStatus.EXPIRED);
        assertThat(expired).hasSize(3);
    }

    @Test
    void 만료미대상문서_상태유지() throws Exception {
        // given: 10일 경과 PENDING 문서 (30일 미만)
        jobLauncherTestUtils.launchJob(jobParameters);

        // then: PENDING 상태 유지 확인
        ApprovalRequest request = requestRepository.findById(requestId).orElseThrow();
        assertThat(request.getStatus()).isEqualTo(ApprovalStatus.PENDING);
    }

    @Test
    void 멱등성_같은Job_두번실행_결과동일() throws Exception {
        // 1회 실행
        jobLauncherTestUtils.launchJob(jobParameters);
        // 2회 실행 (이미 EXPIRED인 문서에 다시 실행)
        jobLauncherTestUtils.launchJob(jobParameters2);

        // then: 상태 변화 없음, 예외 없음
        List<ApprovalRequest> expired = requestRepository.findByStatus(ApprovalStatus.EXPIRED);
        assertThat(expired).hasSize(/* 1회 실행과 동일한 수 */3);
    }
}
```

### 7.2 배치별 테스트 케이스

| 배치 Job | 검증 항목 |
|---------|---------|
| ExpiredRequestJob | 30일 이상 → EXPIRED 전환, 30일 미만 → 상태 유지, 멱등성 |
| EscalationJob | 7일 이상 → ESCALATED 전환, 7일 미만 → 상태 유지, 멱등성 |
| OverdueReminderJob | 3일 이상 → AuditLog 기록 여부, 중복 기록 방지 |
| StatisticsJob | 전일 통계 집계 정확성, 재실행 시 중복 방지 |

### 7.3 대상 선별 정확성 테스트

배치 Reader의 쿼리가 올바른 대상만 선별하는지 검증한다.

```java
@Test
void ExpiredRequestJob_Reader_대상선별_정확성() {
    // given: 만료 대상 3건 + 미대상 2건 DB에 저장

    // when: Job 실행

    // then: 정확히 3건만 처리됨 (2건은 상태 유지)
}
```

---

## 8. Testcontainers 활용 포인트

H2 인메모리 DB가 아닌 실제 PostgreSQL을 사용하는 이유:
- H2는 PostgreSQL 전용 문법(배열 타입, JSON 연산자 등)을 지원하지 않을 수 있다.
- 인덱스 실제 사용 여부는 실제 DB에서만 검증 가능하다.
- 배치 Job의 Spring Batch 메타 테이블이 실제 DB 환경에서 동작해야 한다.

### 설정 예시

```java
@Testcontainers
@SpringBootTest
class ApprovalRequestRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("approval_test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### Testcontainers 적용 범위

| 테스트 클래스 | Testcontainers 필요 여부 | 이유 |
|-------------|------------------------|------|
| ApprovalRequestTest (단위) | 불필요 | 순수 Java, DB 없음 |
| ApprovalRequestServiceTest | 불필요 | Repository 모킹 |
| ApprovalRequestRepositoryTest | 필요 | 실제 DB 쿼리 검증 |
| ApprovalRequestApiTest | 필요 | 전체 레이어 통합 |
| ExpiredRequestJobTest | 필요 | 배치 메타 테이블 + 실제 데이터 |

---

## 9. 테스트 작성 원칙

### 필수 원칙

1. **정상 케이스와 실패 케이스를 반드시 함께 작성한다.** 성공 케이스만 있는 테스트 클래스는 작성하지 않는다.
2. **테스트 메서드명은 한국어로 작성한다.** 의도를 명확히 표현한다. 예: `submit_결재선없이제출시도_유효성예외발생()`
3. **given / when / then 구조를 주석으로 명시한다.**
4. **하나의 테스트는 하나의 동작만 검증한다.**
5. **테스트 간 공유 상태를 만들지 않는다.** `@BeforeEach`에서 각 테스트에 필요한 상태를 독립적으로 준비한다.

### 테스트 메서드명 규칙

```
[대상 메서드]_[입력/상황]_[기대 결과]

예:
submit_결재선없이제출시도_유효성예외발생
approve_담당결재자_마지막단계_APPROVED로전이
withdraw_이미처리된단계있음_상태전이예외발생
expire_이미만료된문서_멱등적종료
```

---

## 10. 테스트 실행 방법

```bash
# 전체 테스트 실행 (Testcontainers 포함 — Docker 필요)
./gradlew test

# 단위 테스트만 실행 (Docker 불필요)
./gradlew test --tests "*.domain.*"

# 서비스 테스트만 실행
./gradlew test --tests "*.service.*"

# 특정 테스트 클래스 실행
./gradlew test --tests "com.example.approval.domain.approval.ApprovalRequestTest"

# 배치 테스트만 실행
./gradlew test --tests "*.batch.*"
```

> Docker가 실행 중이어야 Testcontainers 기반 통합 테스트가 동작한다.
