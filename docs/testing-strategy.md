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
8. [동시성 경합 테스트 전략](#8-동시성-경합-테스트-전략) — 구현 우선순위 포함
9. [Testcontainers 활용 포인트](#9-testcontainers-활용-포인트)
10. [테스트 작성 원칙](#10-테스트-작성-원칙)
11. [테스트 실행 방법](#11-테스트-실행-방법)

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
├── batch/
│   ├── ExpiredRequestJobTest.java           # 배치: 만료 처리
│   ├── EscalationJobTest.java
│   └── OverdueReminderJobTest.java
└── concurrency/
    └── ApprovalConcurrencyTest.java         # 통합: 동시성 경합 (@Version 낙관적 락)
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

## 8. 동시성 경합 테스트 전략

이 섹션은 상태 전이 정합성과 운영 상황 검증의 보강 포인트다. 결재 시스템은 배치 만료 처리와 사용자 승인이 동시에 발생하거나, 회수와 승인 요청이 거의 동시에 도달하는 상황이 실제 운영에서 발생할 수 있다. 이 시나리오들에서 `@Version` 낙관적 락이 올바르게 동작하는지 검증한다.

### 8.1 구현 우선순위

동시성 테스트는 구현 복잡도가 높은 보강 포인트다. MVP 단계에서 핵심 시나리오를 먼저 단위 테스트로 커버하고, 실제 DB 경합이 필요한 시나리오는 이후 단계에서 통합 테스트로 보완한다.

| 단계 | 시나리오 | 구현 방식 |
|------|---------|---------|
| MVP 우선 | 중복 요청/재시도로 상태 전이 2회 시도 | 단위 테스트 (`@Version` 낙관적 락) |
| MVP 우선 | 같은 결재 요청 동시 승인/반려 | 단위 테스트 + 통합 테스트 |
| 확장 단계 | 회수 vs 승인 요청 동시 발생 | 통합 테스트 (Testcontainers) |
| 고도화 단계 | 배치 만료 vs 사용자 승인 경쟁 | 통합 테스트 (Testcontainers) |

> MVP 우선 항목은 도메인 단위 테스트만으로도 상태 전이 불변 조건을 검증할 수 있다. 확장/고도화 단계 항목은 실제 DB 행 버전 충돌이 발생해야 의미 있는 검증이 가능하므로 Testcontainers 환경이 필수다.

### 8.2 동시성 제어 전략 개요

이 프로젝트는 **낙관적 락(Optimistic Lock)** 전략을 채택한다.

| 항목 | 이 프로젝트 (workflow-approval-system) | Wallet Ledger System |
|------|--------------------------------------|----------------------|
| 충돌 대상 | 결재 문서 상태 전이 | 계좌 잔액 |
| 락 전략 | 낙관적 락 (`@Version`) | 비관적 락 (`SELECT FOR UPDATE`) |
| 동시성 도구 | `OptimisticLockException` 처리 | `CyclicBarrier` 기반 동시 실행 제어 |
| 충돌 처리 | 예외를 비즈니스 예외로 변환, 클라이언트 재시도 유도 | 트랜잭션 직렬화로 충돌 원천 차단 |
| 선택 근거 | 결재 충돌 빈도가 낮고, 재시도 비용이 허용 범위 내 | 금융 잔액은 충돌 시 데이터 정합성 손실이 치명적 |

**낙관적 락 선택 근거**: 결재 문서는 동시 경합 빈도가 낮다. 같은 문서에 두 명의 결재자가 거의 동시에 처리하는 상황은 드물고, 발생해도 한쪽이 재시도하면 해결된다. 비관적 락의 DB 행 잠금은 불필요한 대기를 유발한다.

### 8.3 테스트 시나리오

#### 시나리오 1: 같은 결재 요청에 대한 동시 승인/반려 시도

동일한 결재 단계에 두 스레드가 거의 동시에 처리 요청을 보낼 때, 한쪽만 성공하고 나머지는 `OptimisticLockException`이 발생해야 한다.

```java
@Test
void 동시_승인_시도_한쪽만_성공하고_나머지는_충돌예외() throws InterruptedException {
    // given: 동일 결재 단계를 처리하는 두 스레드 준비
    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch latch = new CountDownLatch(1);
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger conflictCount = new AtomicInteger(0);

    for (int i = 0; i < 2; i++) {
        executor.submit(() -> {
            try {
                latch.await();
                approvalRequestService.approve(requestId, stepId, actorId, "승인");
                successCount.incrementAndGet();
            } catch (OptimisticLockConflictException e) {
                conflictCount.incrementAndGet();
            }
        });
    }

    latch.countDown();
    executor.awaitTermination(5, TimeUnit.SECONDS);

    // then: 정확히 1건 성공, 1건 충돌
    assertThat(successCount.get()).isEqualTo(1);
    assertThat(conflictCount.get()).isEqualTo(1);
}
```

#### 시나리오 2: 배치 만료 처리 vs 결재자 승인 경쟁

`ExpiredRequestJob`이 문서를 EXPIRED로 전환하는 동시에, 결재자가 같은 문서를 승인 처리하는 경우다.

```java
@Test
void 배치만료처리와_결재자승인_동시진행_낙관적락으로_정합성보장() throws InterruptedException {
    // given: PENDING 상태 문서 준비 (만료 기준 초과)
    CountDownLatch latch = new CountDownLatch(1);
    AtomicReference<Throwable> batchError = new AtomicReference<>();
    AtomicReference<Throwable> approveError = new AtomicReference<>();

    Thread batchThread = new Thread(() -> {
        try { latch.await(); expiredRequestJob.process(request); }
        catch (Throwable e) { batchError.set(e); }
    });
    Thread approveThread = new Thread(() -> {
        try { latch.await(); approvalRequestService.approve(requestId, stepId, actorId, "승인"); }
        catch (Throwable e) { approveError.set(e); }
    });

    batchThread.start(); approveThread.start();
    latch.countDown();
    batchThread.join(); approveThread.join();

    // then: 두 처리 중 정확히 하나만 성공. 문서 상태는 EXPIRED 또는 APPROVED 중 하나.
    ApprovalRequest result = requestRepository.findById(requestId).orElseThrow();
    boolean exactlyOneSucceeded = (batchError.get() == null) ^ (approveError.get() == null);
    assertThat(exactlyOneSucceeded).isTrue();
    assertThat(result.getStatus()).isIn(ApprovalStatus.EXPIRED, ApprovalStatus.APPROVED);
}
```

#### 시나리오 3: 회수(WITHDRAWN)와 승인(APPROVED)의 동시 요청

기안자의 회수 요청과 결재자의 승인 요청이 거의 동시에 도달하는 경우다. `@Version`이 증가된 쪽이 먼저 커밋되고, 나머지는 충돌 예외를 받아야 한다.

```java
@Test
void 회수와_승인_동시_시도_한쪽만_성공() throws InterruptedException {
    // given: PENDING 상태 문서
    // when: 기안자 회수 + 결재자 승인 동시 실행
    // then: 최종 상태는 WITHDRAWN 또는 APPROVED 중 하나. 중간 상태 없음.
}
```

#### 시나리오 4: 중복 요청으로 상태 전이가 두 번 수행되는 경우

동일한 결재 요청이 네트워크 재시도 등으로 두 번 도달했을 때, 두 번째 처리는 `OptimisticLockException` 또는 이미 전이된 상태를 감지하여 안전하게 처리되어야 한다.

```java
@Test
void 중복_승인_요청_두번째는_충돌예외_또는_멱등처리() {
    // given: PENDING 문서에 첫 번째 승인 처리 완료
    approvalRequestService.approve(requestId, stepId, actorId, "승인");

    // when: 동일 요청 재시도
    assertThatThrownBy(() -> approvalRequestService.approve(requestId, stepId, actorId, "승인"))
        .isInstanceOf(InvalidStateTransitionException.class)
        .extracting("errorCode").isEqualTo(ErrorCode.INVALID_STATE_TRANSITION);
    // 또는 OptimisticLockConflictException — @Version 버전 불일치 시
}
```

### 8.4 테스트 수준 구분

| 시나리오 | 단위 테스트 (순수 Java) | 통합 테스트 (Testcontainers + 실제 DB) |
|---------|----------------------|--------------------------------------|
| 시나리오 1 (동시 승인) | 상태 전이 로직 자체는 단위 검증 가능 | 실제 DB 행 버전 충돌 검증 필요 |
| 시나리오 2 (배치 vs 승인) | 불가 (배치 + DB 필요) | Testcontainers 필수 |
| 시나리오 3 (회수 vs 승인) | 상태 전이 불변 조건 검증 | 동시 실행 정합성은 통합 테스트 |
| 시나리오 4 (중복 요청) | 이미 전이된 상태에서 재전이 시 예외 단위 검증 | 옵션 |

**단위 테스트 역할**: `@Version` 및 DB 없이도 도메인 객체 내 상태 전이 불변 조건 — 이미 APPROVED 상태에서 두 번째 승인 시도 시 `InvalidStateTransitionException` 발생 — 을 검증할 수 있다.

**통합 테스트 역할**: 실제 PostgreSQL + JPA `@Version` 필드가 동시 스레드 환경에서 실제로 `OptimisticLockException`을 발생시키는지 검증한다. 단위 테스트만으로는 DB 레벨 경합을 검증할 수 없다.

---

## 9. Testcontainers 활용 포인트

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
| ApprovalConcurrencyTest | 필요 | @Version 낙관적 락 동작은 실제 DB에서만 검증 가능 |

---

## 10. 테스트 작성 원칙

### 필수 원칙

1. **정상 케이스와 실패 케이스를 반드시 함께 작성한다.** 성공 케이스만 있는 테스트 클래스는 작성하지 않는다.
2. **테스트 메서드명은 한국어로 작성한다.** 의도를 명확히 표현한다. 예: `submit_결재선없이제출시도_유효성예외발생()`
3. **given / when / then 구조를 주석으로 명시한다.**
4. **하나의 테스트는 하나의 동작만 검증한다.**
5. **테스트 간 공유 상태를 만들지 않는다.** `@BeforeEach`에서 각 테스트에 필요한 상태를 독립적으로 준비한다.
6. **정책 로직(상태 전이, 권한 검증, 예외 처리)은 테스트 없이 구현 완료로 간주하지 않는다.** 해당 로직과 대응하는 테스트가 같은 커밋 또는 바로 다음 커밋에 포함되어야 한다.

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

## 11. 테스트 실행 방법

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
