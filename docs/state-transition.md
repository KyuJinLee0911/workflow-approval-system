# 상태 전이 설계 상세

> 관련 문서: design.md §5, authorization.md
> 핵심 대상 엔티티: ApprovalRequest, ApprovalStep

---

## 목차

1. [상태 정의](#1-상태-정의)
2. [상태 전이 다이어그램](#2-상태-전이-다이어그램)
3. [전이 규칙 상세](#3-전이-규칙-상세)
4. [금지 전이 및 이유](#4-금지-전이-및-이유)
5. [ApprovalStep 상태 설계](#5-approvalstep-상태-설계)
6. [구현 전략](#6-구현-전략)
7. [상태 전이 테스트 매트릭스](#7-상태-전이-테스트-매트릭스)

---

## 1. 상태 정의

### 1.1 ApprovalRequest 상태

| 상태 코드 | 한국어 | 의미 | 종료 상태 여부 |
|-----------|--------|------|--------------|
| DRAFT | 초안 | 기안자가 작성 중. 아직 제출하지 않음 | 아니오 |
| PENDING | 결재 대기 | 제출 완료. 첫 번째 결재자 처리 대기 | 아니오 |
| IN_PROGRESS | 결재 진행 중 | 1단계 이상 완료, 다음 단계 처리 중 | 아니오 |
| APPROVED | 승인 완료 | 모든 결재 단계 승인 완료 | 예 (최종) |
| REJECTED | 반려 | 하나 이상의 결재 단계에서 반려됨 | 아니오 (재제출 가능) |
| WITHDRAWN | 회수 | 기안자가 결재 진행 중 직접 회수 | 아니오 (재제출 가능) |
| ESCALATED | 에스컬레이션 | 장기 미처리로 인해 상위 관리자에게 이관됨 (배치 처리) | 아니오 |
| EXPIRED | 만료 | 기한 초과로 시스템이 자동 만료 처리 | 예 (최종) |

**설계 결정: REJECTED는 종료 상태가 아니다**

REJECTED 후에 기안자가 내용을 수정하고 재제출(RESUBMIT)할 수 있도록 설계했다. 일부 결재 시스템은 반려를 최종 상태로 처리하지만, 실무에서는 반려 후 수정 재제출이 일반적이므로 이를 지원한다.

**설계 결정: ESCALATED는 결재 흐름이 계속된다**

에스컬레이션은 상태가 변경될 뿐, 결재 흐름이 중단되지 않는다. ESCALATED 상태에서도 결재자가 승인/반려 처리를 계속할 수 있다. 단, 부서 관리자에게 이관 알림이 발생한다.

### 1.2 ApprovalStep 상태

| 상태 코드 | 한국어 | 의미 |
|-----------|--------|------|
| WAITING | 대기 | 이전 단계 완료 전. 아직 처리 차례가 아님 |
| IN_PROGRESS | 처리 중 | 현재 처리 차례. 결재자 액션 대기 |
| APPROVED | 승인 | 해당 단계 승인 완료 |
| REJECTED | 반려 | 해당 단계 반려 처리됨 |
| SKIPPED | 건너뜀 | (확장 기능) 위임/대결 처리로 건너뜀 |

---

## 2. 상태 전이 다이어그램

```
                    [기안자 작성 중]
                         |
                       DRAFT
                         |
               [기안자 제출 - submit()]
                         |
                         v
                      PENDING ◄─────────────────────────────┐
                         |                                   |
           ┌─────────────┼──────────────────┐               |
           |             |                  |               |
    [기안자 회수]   [1단계 결재자 승인]  [1단계 결재자 반려]  [재제출]
           |             |                  |               |
           v             v                  v               |
       WITHDRAWN   IN_PROGRESS ──────► REJECTED ───────────┘
           |             |                  |
      [재제출]    [중간 단계 반려]      [재제출]
           |             |                  |
           └─────────────┘                  |
                                           └──► (위와 동일 재제출 흐름)
                   IN_PROGRESS
                         |
           [마지막 단계 결재자 승인]
                         |
                         v
                      APPROVED (최종)


   [배치: 7일 이상 미처리]     [배치: 30일 이상 미처리]
   PENDING/IN_PROGRESS         PENDING/IN_PROGRESS/ESCALATED
             |                            |
             v                            v
         ESCALATED ──────────────────► EXPIRED (최종)
             |
   [결재자 계속 처리 가능]
             |
    [승인 완료] / [반려]
       APPROVED / REJECTED
```

---

## 3. 전이 규칙 상세

### 3.1 DRAFT → PENDING (제출)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalRequest.submit()` |
| 전이 주체 | 기안자 본인 (ROLE_DRAFTER + 문서 소유자) |
| 사전 조건 | ① 현재 상태 = DRAFT, ② 제목 필수, ③ 내용 필수, ④ 결재선 1단계 이상 구성 |
| 실행 내용 | status = PENDING, submittedAt = now(), 1단계 ApprovalStep = IN_PROGRESS |
| 실패 예외 | `ApprovalValidationException(APPROVAL_LINE_REQUIRED)` - 결재선 없음 |
| 실패 예외 | `InvalidStateTransitionException` - DRAFT가 아닌 상태에서 제출 시도 |
| HTTP 상태코드 | 성공: 200, 유효성 실패: 400, 상태 전이 실패: 409 |

### 3.2 PENDING → WITHDRAWN (회수)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalRequest.withdraw()` |
| 전이 주체 | 기안자 본인 (ROLE_DRAFTER + 문서 소유자) |
| 사전 조건 | ① 현재 상태 = PENDING, ② 처리된 단계(APPROVED/REJECTED) 없음 |
| 실행 내용 | status = WITHDRAWN, 1단계 ApprovalStep = WAITING으로 복원 |
| 실패 예외 | `InvalidStateTransitionException(CANNOT_WITHDRAW_IN_PROGRESS)` - 이미 처리 중 |
| 실패 예외 | `ApprovalAuthorizationException(NOT_REQUEST_OWNER)` - 소유자 아님 |
| HTTP 상태코드 | 성공: 200, 권한 실패: 403, 상태 전이 실패: 409 |

**설계 이유**: PENDING 상태에서만 회수를 허용한다. IN_PROGRESS 상태(이미 일부 결재 완료)에서는 회수를 허용하지 않는다. 이미 처리된 결재자의 액션을 무효화하는 것은 신뢰성 문제를 야기하기 때문이다.

### 3.3 PENDING → IN_PROGRESS (1단계 승인)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalStep.approve()` → `ApprovalRequest.onStepApproved()` |
| 전이 주체 | 1단계 결재자 (ROLE_APPROVER + 해당 단계 assignee) |
| 사전 조건 | ① 현재 단계 상태 = IN_PROGRESS, ② 결재선에 2단계 이상 존재 |
| 실행 내용 | 현재 Step.status = APPROVED, ApprovalHistory 생성, 다음 Step.status = IN_PROGRESS, Request.status = IN_PROGRESS |
| 실패 예외 | `ApprovalAuthorizationException(NOT_CURRENT_STEP_APPROVER)` - 현재 단계 담당자 아님 |
| HTTP 상태코드 | 성공: 200, 권한 실패: 403 |

### 3.4 IN_PROGRESS → APPROVED (최종 단계 승인)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalStep.approve()` → `ApprovalRequest.onStepApproved()` |
| 전이 주체 | 마지막 단계 결재자 |
| 사전 조건 | ① 현재 단계 = 마지막 단계, ② 단계 상태 = IN_PROGRESS |
| 실행 내용 | 현재 Step.status = APPROVED, ApprovalHistory 생성, Request.status = APPROVED, Request.completedAt = now() |
| 실패 예외 | `ApprovalAuthorizationException(NOT_CURRENT_STEP_APPROVER)` |
| HTTP 상태코드 | 성공: 200, 권한 실패: 403 |

**구현 포인트**: 마지막 단계 여부 판별은 `approvalLine.isLastStep(currentStep)`으로 판단한다. 이 로직이 Service가 아닌 ApprovalLine 또는 ApprovalRequest 도메인 객체 내부에 있어야 테스트가 쉽다.

### 3.5 ANY → REJECTED (반려)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalStep.reject()` → `ApprovalRequest.onStepRejected()` |
| 전이 주체 | 현재 IN_PROGRESS 단계의 결재자 |
| 사전 조건 | ① 현재 단계 상태 = IN_PROGRESS, ② 반려 코멘트 필수 |
| 실행 내용 | 현재 Step.status = REJECTED, ApprovalHistory 생성(comment 포함), Request.status = REJECTED, 이후 단계 모두 WAITING 유지, Request.completedAt = now() |
| 실패 예외 | `ApprovalValidationException(REJECT_COMMENT_REQUIRED)` - 코멘트 없음 |
| 실패 예외 | `ApprovalAuthorizationException(NOT_CURRENT_STEP_APPROVER)` - 담당자 아님 |
| HTTP 상태코드 | 성공: 200, 유효성 실패: 400, 권한 실패: 403 |

**설계 이유**: 반려 시 코멘트를 필수로 요구한다. 기안자가 수정 포인트를 알 수 없으면 재작업이 불가능하기 때문이다. 이는 비즈니스 규칙이므로 도메인 객체에서 검증한다.

### 3.6 REJECTED / WITHDRAWN → PENDING (재제출)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalRequest.resubmit()` |
| 전이 주체 | 기안자 본인 |
| 사전 조건 | ① 현재 상태 = REJECTED 또는 WITHDRAWN, ② 결재선 재구성 완료 |
| 실행 내용 | status = PENDING, 기존 결재선 초기화 (모든 Step = WAITING), 1단계 = IN_PROGRESS, submittedAt 갱신 |
| 실패 예외 | `ApprovalValidationException(APPROVAL_LINE_REQUIRED)` |
| HTTP 상태코드 | 성공: 200, 유효성 실패: 400 |

**설계 결정**: 재제출 시 기존 ApprovalHistory는 삭제하지 않고 보존한다. 반려 이력은 감사 관점에서 중요한 기록이다. 대신 결재 단계 자체는 초기화(WAITING)하고 새로 처리를 시작한다.

### 3.7 배치 전이: → ESCALATED (에스컬레이션)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalRequest.escalate()` |
| 전이 주체 | 시스템 (배치 Job) |
| 사전 조건 | ① 현재 상태 = PENDING 또는 IN_PROGRESS, ② 제출 후 7일 이상 경과 |
| 실행 내용 | status = ESCALATED, AuditLog 기록(actorId = SYSTEM) |
| 실패 예외 | 없음 (배치는 예외 발생 시 로깅 후 다음 문서로 진행) |
| 배치 Job | EscalationJob (매일 09:00) |

### 3.8 배치 전이: → EXPIRED (만료)

| 항목 | 내용 |
|------|------|
| 전이 메서드 | `ApprovalRequest.expire()` |
| 전이 주체 | 시스템 (배치 Job) |
| 사전 조건 | ① 현재 상태 = PENDING, IN_PROGRESS, 또는 ESCALATED, ② 제출 후 30일 이상 경과 |
| 실행 내용 | status = EXPIRED, completedAt = now(), AuditLog 기록 |
| 멱등성 | 이미 EXPIRED인 경우 아무 작업 없이 정상 종료 |
| 배치 Job | ExpiredRequestJob (매일 00:00) |

---

## 4. 금지 전이 및 이유

| 시도된 전이 | 금지 이유 |
|------------|----------|
| APPROVED → any | 최종 승인 완료 후 상태 변경은 데이터 무결성 훼손. 신규 문서 작성으로 대체 |
| EXPIRED → any | 만료 후 상태 복구 불가. 신규 문서 작성으로 대체 |
| DRAFT → IN_PROGRESS | 결재선 없이 진행 중 상태 불가 |
| DRAFT → APPROVED | 결재 없이 승인 불가 |
| DRAFT → REJECTED | 결재 없이 반려 불가 |
| IN_PROGRESS → WITHDRAWN | 결재 진행 중 기안자 회수 불가 (이미 처리된 단계 존재) |
| PENDING → APPROVED | 단계 없이 직접 완료 불가. 반드시 결재 단계를 거쳐야 함 |
| WITHDRAWN → APPROVED | 회수된 문서는 재제출 없이 완료 불가 |

---

## 5. ApprovalStep 상태 설계

### 5.1 ApprovalStep 상태 전이

```
WAITING ──[이전 단계 완료]──► IN_PROGRESS
                                    |
                         ┌──────────┴──────────┐
                    [승인]                  [반려]
                         |                      |
                      APPROVED              REJECTED
```

### 5.2 Step 상태와 Request 상태의 관계

| 이벤트 | Step 변화 | Request 상태 변화 |
|--------|----------|-----------------|
| 제출 | 1단계: WAITING → IN_PROGRESS | DRAFT → PENDING |
| 1단계 승인 (중간 단계 존재) | 1단계: IN_PROGRESS → APPROVED, 2단계: WAITING → IN_PROGRESS | PENDING → IN_PROGRESS |
| 중간 단계 승인 (마지막 아님) | 현재: IN_PROGRESS → APPROVED, 다음: WAITING → IN_PROGRESS | IN_PROGRESS 유지 |
| 마지막 단계 승인 | 현재: IN_PROGRESS → APPROVED | IN_PROGRESS → APPROVED |
| 임의 단계 반려 | 현재: IN_PROGRESS → REJECTED, 이후 단계: WAITING 유지 | PENDING/IN_PROGRESS → REJECTED |
| 재제출 | 모든 APPROVED/REJECTED 단계: WAITING으로 초기화, 1단계: WAITING → IN_PROGRESS | REJECTED/WITHDRAWN → PENDING |

### 5.3 현재 처리 단계 판별 로직

```java
// ApprovalLine.java
public ApprovalStep getCurrentStep() {
    return steps.stream()
        .filter(step -> step.getStatus() == StepStatus.IN_PROGRESS)
        .findFirst()
        .orElseThrow(() -> new IllegalStateException("현재 처리 단계를 찾을 수 없습니다."));
}

public boolean isLastStep(ApprovalStep step) {
    return step.getStepOrder() == steps.size();
}

public boolean hasNoProcessedStep() {
    return steps.stream()
        .noneMatch(step -> step.getStatus() == StepStatus.APPROVED
                       || step.getStatus() == StepStatus.REJECTED);
}
```

---

## 6. 구현 전략

### 6.1 상태 머신 응집 원칙

**원칙**: 상태 전이 검증 로직과 실행 로직은 반드시 ApprovalRequest 도메인 객체 내부에 위치한다.

**이유**:
1. Service 계층에 상태 전이 if-else가 쌓이면 도메인 지식이 인프라 코드에 오염된다.
2. 도메인 객체 내부에 있어야 Spring Context 없이 단위 테스트가 가능하다.
3. 상태 전이 규칙 변경 시 수정 범위가 하나의 클래스로 제한된다.

```java
// ApprovalRequest.java - 도메인 객체에 상태 전이 응집

public class ApprovalRequest {

    private ApprovalStatus status;

    // 허용 전이 매핑 (선언적 정의)
    private static final Map<ApprovalStatus, Set<ApprovalStatus>> ALLOWED_TRANSITIONS;

    static {
        ALLOWED_TRANSITIONS = new EnumMap<>(ApprovalStatus.class);
        ALLOWED_TRANSITIONS.put(DRAFT, EnumSet.of(PENDING));
        ALLOWED_TRANSITIONS.put(PENDING, EnumSet.of(IN_PROGRESS, REJECTED, WITHDRAWN, ESCALATED, EXPIRED));
        ALLOWED_TRANSITIONS.put(IN_PROGRESS, EnumSet.of(APPROVED, REJECTED, ESCALATED, EXPIRED));
        ALLOWED_TRANSITIONS.put(REJECTED, EnumSet.of(PENDING));
        ALLOWED_TRANSITIONS.put(WITHDRAWN, EnumSet.of(PENDING));
        ALLOWED_TRANSITIONS.put(ESCALATED, EnumSet.of(APPROVED, REJECTED, EXPIRED));
        ALLOWED_TRANSITIONS.put(APPROVED, EnumSet.noneOf(ApprovalStatus.class));
        ALLOWED_TRANSITIONS.put(EXPIRED, EnumSet.noneOf(ApprovalStatus.class));
    }

    // 내부 전이 메서드 (상태 변경의 유일한 진입점)
    private void transitionTo(ApprovalStatus newStatus) {
        Set<ApprovalStatus> allowed = ALLOWED_TRANSITIONS.getOrDefault(this.status, Set.of());
        if (!allowed.contains(newStatus)) {
            throw new InvalidStateTransitionException(this.status, newStatus);
        }
        this.status = newStatus;
    }

    // 공개 도메인 메서드들
    public void submit() {
        validateApprovalLine();       // 결재선 있는지 검증
        transitionTo(PENDING);
        this.submittedAt = LocalDateTime.now();
        this.approvalLine.activateFirstStep();
    }

    public void withdraw() {
        if (!approvalLine.hasNoProcessedStep()) {
            throw new InvalidStateTransitionException(
                "이미 처리된 단계가 있어 회수할 수 없습니다.", this.status, WITHDRAWN
            );
        }
        transitionTo(WITHDRAWN);
        this.approvalLine.resetAllSteps();
    }

    public void onStepApproved(ApprovalStep completedStep) {
        if (approvalLine.isLastStep(completedStep)) {
            transitionTo(APPROVED);
            this.completedAt = LocalDateTime.now();
        } else {
            transitionTo(IN_PROGRESS);
            approvalLine.activateNextStep(completedStep);
        }
    }

    public void onStepRejected(ApprovalStep rejectedStep) {
        transitionTo(REJECTED);
        this.completedAt = LocalDateTime.now();
    }

    public void resubmit() {
        validateApprovalLine();
        transitionTo(PENDING);
        this.submittedAt = LocalDateTime.now();
        this.approvalLine.resetAndActivateFirstStep();
    }

    public void escalate() {
        // 이미 ESCALATED면 멱등적으로 처리
        if (this.status == ESCALATED) return;
        transitionTo(ESCALATED);
    }

    public void expire() {
        // 이미 EXPIRED면 멱등적으로 처리
        if (this.status == EXPIRED) return;
        transitionTo(EXPIRED);
        this.completedAt = LocalDateTime.now();
    }
}
```

### 6.2 Service 레벨 역할 분리

Service는 다음 세 가지만 담당한다. 상태 전이 로직은 담당하지 않는다.

```java
@Service
@Transactional
public class ApprovalRequestService {

    public void approve(Long requestId, Long stepId, Long actorId, String comment) {
        // 1. 데이터 조회
        ApprovalRequest request = findRequestById(requestId);
        ApprovalStep step = findStepById(stepId);

        // 2. 권한 검증 (소유권, 현재 단계 검증)
        validateStepApprover(step, actorId);

        // 3. 도메인 메서드 호출 (상태 전이 로직은 여기에 없음)
        step.approve(comment);
        request.onStepApproved(step);

        // 4. 감사 로그 기록
        auditLogService.record(AuditAction.APPROVE, request, actorId, comment);
    }
}
```

---

## 7. 상태 전이 테스트 매트릭스

모든 상태 전이 케이스를 단위 테스트로 커버한다. Spring Context 불필요.

| From | To | 전이 주체 | 테스트 케이스 |
|------|----|----------|------------|
| DRAFT | PENDING | 기안자 | `submit_success_whenValidRequest` |
| DRAFT | PENDING | - | `submit_fail_whenNoApprovalLine` |
| DRAFT | PENDING | - | `submit_fail_whenTitleEmpty` |
| PENDING | WITHDRAWN | 기안자 | `withdraw_success_whenNoProceedStep` |
| PENDING | WITHDRAWN | 기안자 | `withdraw_fail_whenStepAlreadyProcessed` |
| PENDING | WITHDRAWN | 타인 | `withdraw_fail_whenNotOwner` (Service 테스트) |
| PENDING | IN_PROGRESS | 1단계 결재자 | `approve_success_firstStep_multipleSteps` |
| IN_PROGRESS | APPROVED | 마지막 결재자 | `approve_success_lastStep` |
| PENDING | REJECTED | 1단계 결재자 | `reject_success_withComment` |
| PENDING | REJECTED | - | `reject_fail_whenCommentMissing` |
| IN_PROGRESS | REJECTED | 중간 결재자 | `reject_success_middleStep_withComment` |
| REJECTED | PENDING | 기안자 | `resubmit_success_afterRejection` |
| WITHDRAWN | PENDING | 기안자 | `resubmit_success_afterWithdrawal` |
| APPROVED | any | - | `transition_fail_whenApproved` (모든 전이 시도) |
| EXPIRED | any | - | `transition_fail_whenExpired` (모든 전이 시도) |
| PENDING | ESCALATED | 시스템 | `escalate_success_afterSevenDays` |
| ESCALATED | EXPIRED | 시스템 | `expire_success_whenEscalated` |
| EXPIRED | - | - | `expire_isIdempotent_whenAlreadyExpired` |

**총 단위 테스트 목표**: 상태 전이 관련 최소 20개 이상의 독립 테스트 케이스 (모두 순수 Java, 외부 의존성 없음)
