# workflow-approval-system 전체 설계 문서

> 작성일: 2026-03-11
> 목적: SI 기업 포트폴리오용 백엔드 프로젝트 - 결재/승인 워크플로우 시스템
> 기술 스택: Java 17, Spring Boot 3.x, Spring Data JPA, PostgreSQL, JUnit 5, Testcontainers, Docker

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [핵심 기능 정의](#2-핵심-기능-정의)
3. [사용자 역할 및 사용 시나리오](#3-사용자-역할-및-사용-시나리오)
4. [도메인 모델 설계](#4-도메인-모델-설계)
5. [상태 전이 설계](#5-상태-전이-설계)
6. [권한 설계](#6-권한-설계)
7. [시스템 아키텍처](#7-시스템-아키텍처)
8. [API 설계 초안](#8-api-설계-초안)
9. [DB 설계 초안](#9-db-설계-초안)
10. [예외 처리 전략](#10-예외-처리-전략)
11. [감사 로그 설계](#11-감사-로그-설계)
12. [배치 처리 설계](#12-배치-처리-설계)
13. [테스트 전략](#13-테스트-전략)
14. [개발 단계 계획](#14-개발-단계-계획)
15. [면접 어필 포인트](#15-면접-어필-포인트)
16. [Wallet Ledger System과의 차별점](#16-wallet-ledger-system과의-차별점)
17. [리스크 및 보완 포인트](#17-리스크-및-보완-포인트)

---

## 1. 프로젝트 개요

### 1.1 프로젝트 정의

workflow-approval-system은 기업 내 문서 결재 및 승인 흐름을 관리하는 백엔드 시스템이다. 기안자가 결재 문서를 작성하면, 미리 정의된 결재선(승인 체인)에 따라 단계별 결재자가 순차적으로 승인 또는 반려 처리를 진행한다. 모든 상태 변경과 처리 이력은 감사 로그로 추적된다.

### 1.2 포트폴리오 관점에서의 의도

이 프로젝트는 다음 역량을 코드와 설계로 명확히 보여주기 위해 설계되었다:

| 역량 | 구현 포인트 |
|------|------------|
| 상태 전이 설계 | ApprovalRequest의 상태 머신, 전이 규칙을 도메인 객체에 응집 |
| 권한 기반 접근 제어 | Role + Permission + 소유권 검증의 3단 구조 |
| 예외 처리 | 비즈니스 예외 계층화, GlobalExceptionHandler, ErrorCode Enum |
| 감사 로그 | AuditLog 전용 테이블, MVP는 명시적 호출로 구현. AOP는 확장 포인트로 분리 가능. |
| 배치 처리 | Spring Batch 기반 장기 미처리 문서 리마인드 및 통계 집계 |
| 테스트 가능한 구조 | 계층 분리, 의존성 역전, Testcontainers 통합 테스트 |

### 1.3 핵심 도메인 용어

| 용어 | 설명 |
|------|------|
| 결재 문서 (ApprovalRequest) | 결재 요청의 핵심 단위. 제목, 내용, 상태, 결재선을 포함 |
| 결재선 (ApprovalLine) | 결재 문서에 연결된 순서형 결재 단계 목록 |
| 결재 단계 (ApprovalStep) | 특정 결재자가 처리해야 할 단일 단계. 순번, 상태, 담당자 포함 |
| 기안자 (Drafter) | 결재 문서를 작성하고 제출하는 사용자 |
| 결재자 (Approver) | 결재선에 포함되어 승인/반려 권한을 가진 사용자 |
| 결재 이력 (ApprovalHistory) | 각 결재 단계의 처리 결과와 코멘트를 기록하는 이력 |
| 감사 로그 (AuditLog) | 모든 주요 액션에 대한 운영 추적 기록 |

---

## 2. 핵심 기능 정의

### 2.1 MVP 기능 (반드시 구현)

| 기능 그룹 | 세부 기능 |
|-----------|----------|
| 사용자 관리 | 사용자 등록, 역할 부여, 부서 배치 |
| 결재 문서 관리 | 문서 초안 작성, 임시저장, 제출, 회수, 재제출 |
| 결재선 관리 | 결재선 템플릿 정의, 문서별 결재선 구성 |
| 결재 처리 | 단계별 승인, 반려, 조건부 승인(코멘트 필수), 전체 완료/반려 처리 |
| 결재 이력 | 단계별 처리 이력 조회, 코멘트 기록 |
| 감사 로그 | 주요 액션 명시적 기록 (MVP), 감사 로그 조회 (관리자) |
| 권한 제어 | 역할별 API 접근 제어, 소유권 검증 |
| 예외 처리 | 비즈니스 예외 계층화, 일관된 에러 응답 |
| 배치 처리 | 장기 미처리 문서 감지 및 상태 정리 |

### 2.2 확장 기능 (MVP 이후 추가 가능)

| 기능 | 확장 이유 | 구현 복잡도 |
|------|----------|------------|
| 결재선 템플릿 | 부서별 표준 결재 라인 재사용 | 중 |
| 대결/위임 처리 | 결재자 부재 시 대리 처리 | 높음 |
| 알림 시스템 | 이메일/슬랙 결재 요청 알림 | 중 |
| 병렬 결재 | 복수 결재자 동시 처리 | 높음 |
| 첨부파일 관리 | S3 연동 파일 업로드 | 중 |
| 결재 위임 기간 설정 | 특정 기간 자동 위임 | 높음 |
| 통계 대시보드 API | 결재 처리 현황 집계 | 중 |
| 외부 연동 (ERP, HR) | 조직도 연동, SSO | 높음 |

**MVP 범위 결정 근거**: 포트폴리오 목적상 설계 의도와 구조의 명확성이 기능 수보다 중요하다. 핵심 플로우(기안-결재-완료/반려)가 완전하게 동작하고, 상태 전이/권한/감사/배치가 각각 구현되면 충분한 역량 시연이 가능하다.

---

## 3. 사용자 역할 및 사용 시나리오

### 3.1 사용자 역할 정의

| 역할 코드 | 역할명 | 주요 책임 |
|-----------|--------|----------|
| ROLE_DRAFTER | 기안자 | 결재 문서 작성, 제출, 회수 |
| ROLE_APPROVER | 결재자 | 담당 결재 단계 승인/반려 |
| ROLE_VIEWER | 참조자 | 문서 열람 (처리 권한 없음) |
| ROLE_DEPT_MANAGER | 부서 관리자 | 부서 내 결재 현황 조회, 강제 회수 |
| ROLE_ADMIN | 시스템 관리자 | 전체 권한, 감사 로그 조회, 배치 모니터링 |

> 주의: 한 사용자가 복수의 역할을 가질 수 있다. 예: 팀장은 ROLE_DRAFTER + ROLE_APPROVER + ROLE_DEPT_MANAGER를 동시에 보유한다.

### 3.2 핵심 사용 시나리오

**시나리오 1: 정상 결재 플로우**
```
1. 기안자(김철수)가 휴가 신청 문서를 초안 작성 (DRAFT 상태)
2. 기안자가 결재선을 구성 (팀장 → 부서장 순서)
3. 기안자가 문서를 제출 (PENDING 상태)
4. 팀장(이영희)에게 1단계 결재 알림
5. 팀장이 로그인 후 승인 처리 (1단계 APPROVED, 2단계 IN_PROGRESS)
6. 부서장(박민준)이 최종 승인 (2단계 APPROVED)
7. 전체 문서 상태가 APPROVED로 완료
```

**시나리오 2: 반려 후 재제출 플로우**
```
1. 기안자가 문서 제출 (PENDING)
2. 1단계 결재자가 반려 + 반려 사유 코멘트 입력 (REJECTED)
3. 문서 상태가 REJECTED로 전환
4. 기안자가 문서 내용 수정 후 재제출 (PENDING으로 복귀, 결재선 초기화)
5. 결재 프로세스 처음부터 재시작
```

**시나리오 3: 기안자 회수 플로우**
```
1. 기안자가 문서 제출 (PENDING)
2. 아직 아무 결재자도 처리하지 않은 상태에서 기안자가 회수 요청
3. 문서 상태가 WITHDRAWN으로 전환
4. 기안자가 내용 수정 후 재제출 가능
```

**시나리오 4: 배치 처리 시나리오**
```
매일 오전 9시 배치 실행:
1. 3일 이상 미처리된 결재 문서를 감지
2. 해당 결재자에게 리마인드 이벤트 발행 (로그 기록)
3. 7일 이상 미처리 문서는 ESCALATED 상태로 전환 + 부서 관리자에게 알림
4. 30일 이상 미처리 문서는 EXPIRED 상태로 자동 만료 처리
```

---

## 4. 도메인 모델 설계

### 4.1 엔티티 목록 및 책임

| 엔티티 | 역할 | Aggregate 위치 |
|--------|------|----------------|
| User | 시스템 사용자, 역할 보유 | User Aggregate Root |
| Department | 조직 단위, 사용자 소속 | User Aggregate 내 참조 |
| ApprovalRequest | 결재 문서, 상태 머신의 핵심 | ApprovalRequest Aggregate Root |
| ApprovalLine | 결재 문서에 속한 결재선 (순서 컬렉션) | ApprovalRequest Aggregate 내 |
| ApprovalStep | 결재선의 단일 단계 | ApprovalRequest Aggregate 내 |
| ApprovalHistory | 각 단계의 처리 결과 이력 | ApprovalRequest Aggregate 내 |
| AuditLog | 시스템 전체 감사 추적 | 독립 Aggregate |

### 4.2 엔티티 상세 설계

#### User
```
- id: Long (PK)
- username: String (UNIQUE, NOT NULL)
- email: String (UNIQUE, NOT NULL)
- passwordHash: String
- name: String
- department: Department (ManyToOne)
- roles: Set<UserRole> (OneToMany)
- status: UserStatus (ACTIVE / INACTIVE)
- createdAt, updatedAt
```

#### Department
```
- id: Long (PK)
- name: String
- code: String (UNIQUE)
- parentDepartment: Department (자기 참조, nullable - 최상위 부서)
- manager: User (ManyToOne, nullable)
- createdAt
```

#### ApprovalRequest (핵심 Aggregate Root)
```
- id: Long (PK)
- title: String
- content: String (TEXT)
- category: ApprovalCategory (ENUM: 휴가, 지출, 구매, 기타)
- status: ApprovalStatus (ENUM: 상태 머신의 핵심)
- drafter: User (ManyToOne, 기안자)
- department: Department (소속 부서)
- approvalLine: ApprovalLine (OneToOne, cascade)
- submittedAt: LocalDateTime (제출 시각)
- completedAt: LocalDateTime (완료/반려 시각)
- dueDate: LocalDate (처리 기한, nullable)
- createdAt, updatedAt
```

**설계 근거**: ApprovalRequest가 결재선과 결재 단계를 소유하는 Aggregate Root로 설계한다. 결재 단계는 결재 문서 없이 독립적으로 존재할 수 없으므로 같은 Aggregate 경계 안에 포함한다. 단, AuditLog는 결재 문서 외 다양한 엔티티의 변경을 추적하므로 독립 Aggregate로 분리한다.

#### ApprovalLine
```
- id: Long (PK)
- approvalRequest: ApprovalRequest (OneToOne)
- steps: List<ApprovalStep> (OneToMany, orderBy stepOrder)
- createdAt
```

#### ApprovalStep
```
- id: Long (PK)
- approvalLine: ApprovalLine (ManyToOne)
- stepOrder: int (결재 순서, 1부터 시작)
- approver: User (ManyToOne, 담당 결재자)
- status: StepStatus (WAITING / IN_PROGRESS / APPROVED / REJECTED / SKIPPED)
- role: String (해당 단계에서의 역할 명칭: 팀장, 부서장 등, 표시용)
- history: ApprovalHistory (OneToOne, nullable - 처리 후 생성)
- createdAt, updatedAt
```

#### ApprovalHistory
```
- id: Long (PK)
- approvalStep: ApprovalStep (OneToOne)
- approver: User (처리한 사용자)
- action: HistoryAction (APPROVED / REJECTED / WITHDRAWN / ESCALATED)
- comment: String (TEXT, nullable - 반려 시 필수)
- processedAt: LocalDateTime
```

#### AuditLog
```
- id: Long (PK)
- entityType: String (예: ApprovalRequest, User)
- entityId: Long
- action: String (예: SUBMIT, APPROVE, REJECT, WITHDRAW)
- actorId: Long (행위자 ID)
- actorName: String (행위자 이름, 비정규화 - 추후 사용자 삭제 시에도 이력 보존)
- beforeState: String (JSON, nullable - 변경 전 상태)
- afterState: String (변경 후 상태)
- ipAddress: String
- userAgent: String
- occurredAt: LocalDateTime
```

**비정규화 설계 근거**: actorName을 AuditLog에 직접 저장하는 이유는 사용자가 삭제되거나 이름이 변경되더라도 감사 이력의 정확성을 유지하기 위함이다. 이는 감사 로그의 불변성 원칙에 해당한다.

### 4.3 연관관계 정리

```
User (1) ─── (N) ApprovalRequest [drafter]
User (1) ─── (N) ApprovalStep [approver]
Department (1) ─── (N) User
Department (1) ─── (N) ApprovalRequest
ApprovalRequest (1) ─── (1) ApprovalLine [cascade ALL]
ApprovalLine (1) ─── (N) ApprovalStep [cascade ALL, orderBy stepOrder]
ApprovalStep (1) ─── (1) ApprovalHistory [cascade ALL]
```

---

## 5. 상태 전이 설계

상세 내용은 `state-transition.md` 참조.

### 5.1 ApprovalRequest 상태 정의

| 상태 | 코드 | 의미 |
|------|------|------|
| 초안 | DRAFT | 기안자가 작성 중인 상태. 미제출. |
| 결재 대기 | PENDING | 제출 완료, 첫 번째 결재자 처리 대기 중 |
| 결재 진행 중 | IN_PROGRESS | 일부 단계 완료, 다음 단계 처리 중 |
| 승인 완료 | APPROVED | 모든 결재 단계 승인 완료 |
| 반려 | REJECTED | 결재 단계 중 하나라도 반려 처리됨 |
| 회수 | WITHDRAWN | 기안자가 제출 후 회수 |
| 에스컬레이션 | ESCALATED | 장기 미처리로 상위 관리자에게 이관됨 (배치 처리) |
| 만료 | EXPIRED | 기한 초과로 자동 만료 처리됨 (배치 처리) |

### 5.2 상태 전이 표

| From \ To | DRAFT | PENDING | IN_PROGRESS | APPROVED | REJECTED | WITHDRAWN | ESCALATED | EXPIRED |
|-----------|-------|---------|-------------|----------|----------|-----------|-----------|---------|
| DRAFT | - | 기안자 제출 | X | X | X | X | X | X |
| PENDING | 기안자 회수* | - | 1단계 승인 | X | 1단계 반려 | 기안자 회수 | 배치(7일) | 배치(30일) |
| IN_PROGRESS | X | X | - | 마지막 단계 승인 | 중간 단계 반려 | X | 배치(7일) | 배치(30일) |
| APPROVED | X | X | X | - | X | X | X | X |
| REJECTED | X | 기안자 재제출 | X | X | - | X | X | X |
| WITHDRAWN | X | 기안자 재제출 | X | X | X | - | X | X |
| ESCALATED | X | X | - | 최종 단계 승인 | 중간 단계 반려 | X | - | 배치(30일) |
| EXPIRED | X | X | X | X | X | X | X | - |

> *PENDING → DRAFT 회수: DRAFT로 돌아가는 것이 아니라 WITHDRAWN으로 전환됨. 이후 재제출 시 PENDING으로 전환.

### 5.3 주요 전이 규칙 요약

| 전이 | 주체 | 사전 조건 | 실패 시 예외 |
|------|------|----------|------------|
| DRAFT → PENDING | 기안자 본인 | 결재선 1개 이상, 제목/내용 필수 | ApprovalValidationException (400) |
| PENDING → WITHDRAWN | 기안자 본인 | 아직 처리된 단계 없음 | InvalidStateTransitionException (409) |
| PENDING → IN_PROGRESS | 1단계 결재자 | 본인이 1단계 결재자 | ApprovalAuthorizationException (403) |
| IN_PROGRESS → APPROVED | 마지막 단계 결재자 | 본인이 현재 단계 결재자 | ApprovalAuthorizationException (403) |
| IN_PROGRESS → REJECTED | 현재 단계 결재자 | 반려 코멘트 필수 | ApprovalValidationException (400) |
| REJECTED → PENDING | 기안자 본인 | 결재선 재구성 필수 | ApprovalValidationException (400) |

### 5.4 상태 전이 로직 구현 전략

**문제**: 상태 전이 로직이 Service 계층에 if-else로 분산되면 유지보수가 어렵고 테스트하기 힘들다.

**해결**: 상태 전이 검증과 실행을 ApprovalRequest 도메인 객체 내부에 응집시킨다.

```java
// ApprovalRequest.java (도메인 객체)
public class ApprovalRequest {
    private ApprovalStatus status;

    public void submit() {
        validateCanSubmit();  // 전이 가능 여부 검증
        this.status = ApprovalStatus.PENDING;
        this.submittedAt = LocalDateTime.now();
    }

    public void withdraw() {
        if (!canWithdraw()) {
            throw new InvalidStateTransitionException(
                "PENDING 상태에서 처리된 단계가 없어야 회수 가능합니다.",
                this.status, ApprovalStatus.WITHDRAWN
            );
        }
        this.status = ApprovalStatus.WITHDRAWN;
    }

    private boolean canWithdraw() {
        return this.status == ApprovalStatus.PENDING
            && this.approvalLine.hasNoProcessedStep();
    }

    // 허용 전이 목록으로 상태 검증
    private static final Map<ApprovalStatus, Set<ApprovalStatus>> ALLOWED_TRANSITIONS = Map.of(
        ApprovalStatus.DRAFT, Set.of(ApprovalStatus.PENDING),
        ApprovalStatus.PENDING, Set.of(ApprovalStatus.IN_PROGRESS, ApprovalStatus.REJECTED, ApprovalStatus.WITHDRAWN, ApprovalStatus.ESCALATED, ApprovalStatus.EXPIRED),
        ...
    );
}
```

**장점**:
- 상태 전이 로직이 도메인 객체에 응집 → 단위 테스트가 Service 없이 가능
- Service는 권한 검증 + 도메인 메서드 호출 + 트랜잭션 경계만 담당
- 새로운 상태 추가 시 ALLOWED_TRANSITIONS 맵과 메서드만 수정하면 됨

---

## 6. 권한 설계

상세 내용은 `authorization.md` 참조.

### 6.1 권한 구조 설계 원칙

3단 검증 구조를 사용한다:

```
1단계 (API 레벨): 인증 여부 확인 (Spring Security - JWT 토큰)
2단계 (API 레벨): 역할(Role) 기반 접근 제어 (Spring Security @PreAuthorize)
3단계 (Service 레벨): 소유권 + 비즈니스 규칙 검증 (커스텀 검증)
```

**설계 근거**: 1, 2단계는 선언적으로 처리하여 횡단 관심사로 분리한다. 3단계는 비즈니스 규칙과 결합되어 있으므로 Service 레벨에서 명시적으로 검증한다. 이 구조는 "어디서 무엇을 검증하는가"가 명확하게 분리되어 테스트하기 좋다.

### 6.2 역할별 허용 동작

| 동작 | DRAFTER | APPROVER | VIEWER | DEPT_MANAGER | ADMIN |
|------|---------|----------|--------|--------------|-------|
| 문서 작성/임시저장 | 본인 | X | X | O | O |
| 문서 제출 | 본인 | X | X | X | O |
| 문서 회수 | 본인* | X | X | X | O |
| 결재 승인 | X | 담당 단계 | X | X | O |
| 결재 반려 | X | 담당 단계 | X | X | O |
| 문서 열람 | 본인/참조 | 담당 | 지정 | 부서 내 | 전체 |
| 감사 로그 조회 | X | X | X | X | O |
| 배치 모니터링 | X | X | X | X | O |
| 강제 상태 변경 | X | X | X | 부서 내* | O |

> *조건부 허용 - 비즈니스 규칙이 추가로 적용됨

### 6.3 소유권 검증 예시

```java
// ApprovalRequestService.java
public void withdrawRequest(Long requestId, Long actorId) {
    ApprovalRequest request = findById(requestId);

    // 3단계: 소유권 검증 (Service 레벨)
    if (!request.isDraftedBy(actorId)) {
        throw new ApprovalAuthorizationException(
            "기안자 본인만 회수할 수 있습니다.",
            ErrorCode.NOT_REQUEST_OWNER
        );
    }

    // 도메인 메서드 호출 (상태 전이 검증 포함)
    request.withdraw();
    auditLogService.record(AuditAction.WITHDRAW, request, actorId);
}
```

---

## 7. 시스템 아키텍처

상세 내용은 `architecture.md` 참조.

### 7.1 계층 구조

```
[Client / Postman]
        |
[Presentation Layer]  ← Controller, DTO, GlobalExceptionHandler
        |
[Application Layer]   ← Service (트랜잭션 경계, 권한 검증, 이벤트 발행)
        |
[Domain Layer]        ← Entity, 상태 전이 로직, 도메인 예외, 비즈니스 규칙
        |
[Infrastructure Layer] ← Repository (JPA), AuditLogRepository, BatchJob
        |
[DB: PostgreSQL]
```

### 7.2 패키지 구조

```
com.example.approval
├── common
│   ├── exception          # 비즈니스 예외 계층
│   │   ├── ApprovalException.java
│   │   ├── InvalidStateTransitionException.java
│   │   ├── ApprovalAuthorizationException.java
│   │   └── ErrorCode.java
│   ├── response           # 공통 응답 래퍼
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── audit              # 감사 로그 인프라
│       ├── AuditLogService.java
│       └── AuditLogAspect.java
│
├── domain
│   ├── user
│   │   ├── entity         # User, Department, UserRole
│   │   ├── repository
│   │   └── service
│   │
│   ├── approval
│   │   ├── entity         # ApprovalRequest, ApprovalLine, ApprovalStep, ApprovalHistory
│   │   │   ├── ApprovalRequest.java      # Aggregate Root
│   │   │   ├── ApprovalLine.java
│   │   │   ├── ApprovalStep.java
│   │   │   ├── ApprovalHistory.java
│   │   │   └── enums
│   │   │       ├── ApprovalStatus.java
│   │   │       └── StepStatus.java
│   │   ├── repository
│   │   │   ├── ApprovalRequestRepository.java
│   │   │   └── ApprovalStepRepository.java
│   │   ├── service
│   │   │   ├── ApprovalRequestService.java  # 핵심 서비스
│   │   │   └── ApprovalQueryService.java    # 조회 전용 서비스
│   │   └── controller
│   │       └── ApprovalRequestController.java
│   │
│   └── auditlog
│       ├── entity         # AuditLog
│       ├── repository
│       └── controller     # 관리자 전용 감사 로그 조회
│
├── batch
│   ├── job
│   │   ├── OverdueReminderJobConfig.java   # 장기 미처리 리마인드
│   │   ├── ExpiredRequestJobConfig.java    # 만료 처리
│   │   └── ApprovalStatisticsJobConfig.java # 통계 집계
│   ├── tasklet
│   └── listener
│
├── security
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java
│   └── SecurityConfig.java
│
└── config
    ├── JpaConfig.java
    ├── BatchConfig.java
    └── SwaggerConfig.java
```

**선택 이유**: 도메인 중심 패키지 구조를 채택했다. 기능 그룹(user, approval, auditlog)으로 최상위 패키지를 나누고, 그 안에서 계층을 나누는 방식이다. 이 구조는 도메인 간 경계가 명확하고, 향후 서비스 분리 시 패키지 단위로 분리하기 쉽다는 장점이 있다.

**대안**: 계층 중심 패키지 구조 (controller/, service/, repository/ 최상위)는 소규모 프로젝트에서는 단순하지만, 도메인이 늘어날수록 각 계층 내 파일이 뒤섞여 도메인 경계 파악이 어려워진다.

### 7.3 트랜잭션 경계 정의

| 트랜잭션 단위 | 위치 | 범위 |
|-------------|------|------|
| 결재 문서 제출 | ApprovalRequestService.submit() | ApprovalRequest 상태 변경 + AuditLog 저장 |
| 결재 단계 승인 | ApprovalRequestService.approve() | ApprovalStep 상태 변경 + ApprovalHistory 저장 + ApprovalRequest 상태 업데이트 + AuditLog 저장 |
| 결재 단계 반려 | ApprovalRequestService.reject() | 위와 동일 |
| 배치 만료 처리 | ExpiredRequestJobConfig | 청크 단위 (chunkSize=100) |

**중요**: AuditLog는 동일 트랜잭션 내에서 저장한다. 만약 별도 트랜잭션으로 분리하면 메인 트랜잭션 성공 후 AuditLog 저장 실패 시 감사 이력이 누락될 수 있다.

---

## 8. API 설계 초안

상세 내용은 `api-spec.md` 참조.

### 8.1 API 목록 (MVP 기준)

**사용자 / 인증**

| Method | Path | 설명 | 역할 |
|--------|------|------|------|
| POST | /api/v1/auth/login | 로그인, JWT 발급 | 전체 |
| POST | /api/v1/users | 사용자 등록 | ADMIN |
| GET | /api/v1/users/{id} | 사용자 조회 | ADMIN |

**결재 문서**

| Method | Path | 설명 | 역할 |
|--------|------|------|------|
| POST | /api/v1/approval-requests | 결재 문서 초안 작성 | DRAFTER |
| GET | /api/v1/approval-requests | 결재 문서 목록 조회 (필터 지원) | 전체 (역할별 범위 제한) |
| GET | /api/v1/approval-requests/{id} | 결재 문서 상세 조회 | 권한 있는 사용자 |
| PUT | /api/v1/approval-requests/{id} | 결재 문서 수정 (DRAFT 상태만) | 기안자 본인 |
| POST | /api/v1/approval-requests/{id}/submit | 결재 문서 제출 | 기안자 본인 |
| POST | /api/v1/approval-requests/{id}/withdraw | 결재 문서 회수 | 기안자 본인 |
| POST | /api/v1/approval-requests/{id}/resubmit | 결재 문서 재제출 (REJECTED/WITHDRAWN) | 기안자 본인 |

**결재 처리**

| Method | Path | 설명 | 역할 |
|--------|------|------|------|
| GET | /api/v1/approval-requests/my-tasks | 내가 처리해야 할 결재 목록 | APPROVER |
| POST | /api/v1/approval-requests/{id}/steps/{stepId}/approve | 단계 승인 | 해당 단계 결재자 |
| POST | /api/v1/approval-requests/{id}/steps/{stepId}/reject | 단계 반려 | 해당 단계 결재자 |

**감사 로그**

| Method | Path | 설명 | 역할 |
|--------|------|------|------|
| GET | /api/v1/audit-logs | 감사 로그 조회 | ADMIN |
| GET | /api/v1/audit-logs/{entityType}/{entityId} | 특정 엔티티 감사 로그 | ADMIN |

### 8.2 공통 응답 구조

```json
// 성공 응답
{
  "success": true,
  "data": { ... },
  "timestamp": "2026-03-11T09:00:00"
}

// 실패 응답
{
  "success": false,
  "error": {
    "code": "INVALID_STATE_TRANSITION",
    "message": "현재 상태(APPROVED)에서는 제출할 수 없습니다.",
    "details": null
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

---

## 9. DB 설계 초안

상세 내용은 `erd.md` 참조.

### 9.1 주요 테이블 목록

| 테이블명 | 설명 |
|---------|------|
| users | 시스템 사용자 |
| departments | 부서/조직 |
| user_roles | 사용자-역할 매핑 |
| approval_requests | 결재 문서 (핵심) |
| approval_lines | 결재선 |
| approval_steps | 결재 단계 |
| approval_histories | 결재 처리 이력 |
| audit_logs | 감사 로그 |

### 9.2 상태값 저장 방식 결정

**결정**: VARCHAR + Enum 매핑 방식 사용 (JPA `@Enumerated(EnumType.STRING)`)

| 방식 | 장점 | 단점 |
|------|------|------|
| VARCHAR (Enum 이름) | DB에서 바로 가독성 있음, JPA 매핑 단순 | Enum 이름 변경 시 DB 마이그레이션 필요 |
| INT (Enum 순서) | 저장 공간 효율 | DB에서 의미 파악 불가, Enum 순서 변경 시 위험 |
| 별도 코드 테이블 | 운영 중 코드 추가 유연 | 조인 필요, 복잡도 증가 |
| Check Constraint | DB 레벨 유효성 보장 | 변경 시 DDL 필요 |

**근거**: VARCHAR + Enum 매핑은 가독성과 구현 단순성의 균형이 좋다. Flyway로 마이그레이션을 관리하면 이름 변경 리스크도 제어 가능하다. INT 방식은 데이터 보안 감사 시 DB 직접 조회가 어려워 운영 불편이 크다.

### 9.3 인덱스 전략

```sql
-- approval_requests
CREATE INDEX idx_ar_status ON approval_requests(status);
CREATE INDEX idx_ar_drafter_id ON approval_requests(drafter_id);
CREATE INDEX idx_ar_department_id ON approval_requests(department_id);
CREATE INDEX idx_ar_submitted_at ON approval_requests(submitted_at);
CREATE INDEX idx_ar_status_submitted ON approval_requests(status, submitted_at); -- 배치 처리용

-- approval_steps
CREATE INDEX idx_as_approver_id_status ON approval_steps(approver_id, status); -- 내 결재 목록 조회

-- audit_logs
CREATE INDEX idx_al_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_al_actor_id ON audit_logs(actor_id);
CREATE INDEX idx_al_occurred_at ON audit_logs(occurred_at);
```

---

## 10. 예외 처리 전략

### 10.1 예외 계층 구조

```
RuntimeException
└── ApprovalException (추상 기반 클래스)
    ├── InvalidStateTransitionException   # 잘못된 상태 전이 시도
    ├── ApprovalAuthorizationException    # 권한 없는 접근
    ├── ApprovalValidationException       # 입력값/비즈니스 규칙 위반
    ├── ApprovalNotFoundException         # 리소스 없음
    └── ApprovalBatchException            # 배치 처리 오류
```

### 10.2 ErrorCode Enum

```java
public enum ErrorCode {
    // 상태 전이 오류 (409 Conflict)
    INVALID_STATE_TRANSITION("INVALID_STATE_TRANSITION", "허용되지 않는 상태 전이입니다."),
    ALREADY_PROCESSED("ALREADY_PROCESSED", "이미 처리된 결재입니다."),

    // 권한 오류 (403 Forbidden)
    NOT_REQUEST_OWNER("NOT_REQUEST_OWNER", "결재 문서의 기안자가 아닙니다."),
    NOT_ASSIGNED_APPROVER("NOT_ASSIGNED_APPROVER", "해당 단계의 결재자가 아닙니다."),
    NOT_CURRENT_STEP_APPROVER("NOT_CURRENT_STEP_APPROVER", "현재 처리 단계의 결재자가 아닙니다."),

    // 유효성 오류 (400 Bad Request)
    APPROVAL_LINE_REQUIRED("APPROVAL_LINE_REQUIRED", "결재선을 1개 이상 구성해야 합니다."),
    REJECT_COMMENT_REQUIRED("REJECT_COMMENT_REQUIRED", "반려 시 반려 사유를 입력해야 합니다."),
    INVALID_STEP_ORDER("INVALID_STEP_ORDER", "결재 단계 순서가 올바르지 않습니다."),

    // 리소스 없음 (404 Not Found)
    REQUEST_NOT_FOUND("REQUEST_NOT_FOUND", "결재 문서를 찾을 수 없습니다."),
    USER_NOT_FOUND("USER_NOT_FOUND", "사용자를 찾을 수 없습니다."),
    STEP_NOT_FOUND("STEP_NOT_FOUND", "결재 단계를 찾을 수 없습니다.");

    private final String code;
    private final String defaultMessage;
}
```

### 10.3 GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InvalidStateTransitionException.class)
    public ResponseEntity<ErrorResponse> handleInvalidStateTransition(InvalidStateTransitionException e) {
        // HTTP 409 Conflict
    }

    @ExceptionHandler(ApprovalAuthorizationException.class)
    public ResponseEntity<ErrorResponse> handleAuthorizationException(ApprovalAuthorizationException e) {
        // HTTP 403 Forbidden
    }

    @ExceptionHandler(ApprovalValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(ApprovalValidationException e) {
        // HTTP 400 Bad Request
    }

    @ExceptionHandler(ApprovalNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFoundException(ApprovalNotFoundException e) {
        // HTTP 404 Not Found
    }
}
```

**설계 원칙**: 예외 클래스가 HTTP 상태코드를 알지 못하도록 한다. HTTP 매핑은 GlobalExceptionHandler에서만 담당한다. 도메인 예외는 도메인 언어로만 말하고, HTTP 변환은 인프라 계층의 책임이다.

---

## 11. 감사 로그 설계

### 11.1 기록 대상 액션

| 액션 | 트리거 | 기록 필수 여부 |
|------|--------|--------------|
| SUBMIT | 결재 문서 제출 | 필수 |
| APPROVE | 결재 단계 승인 | 필수 |
| REJECT | 결재 단계 반려 | 필수 |
| WITHDRAW | 기안자 회수 | 필수 |
| RESUBMIT | 재제출 | 필수 |
| ESCALATE | 에스컬레이션 (배치) | 필수 |
| EXPIRE | 자동 만료 (배치) | 필수 |
| USER_CREATED | 사용자 등록 | 권장 |
| ROLE_CHANGED | 역할 변경 | 필수 |

### 11.2 구현 방식

#### MVP: 명시적 호출 방식

MVP 구현 범위에서 감사 로그는 **Service 계층에서 `auditLogService.record()`를 직접 호출**하는 방식으로 구현한다. 결재 상태 전이, 승인, 반려, 회수 등 도메인 행위의 의미가 명확한 액션은 어느 시점에 무엇을 기록하는지 코드에서 즉시 파악 가능해야 하고, 상태 전이 직후 정확한 컨텍스트를 포착해야 하기 때문이다.

```java
// ApprovalRequestService.withdraw() 예시
public void withdraw(Long requestId, Long actorId) {
    ApprovalRequest request = findById(requestId);
    request.validateOwner(actorId);      // 소유권 검증
    request.withdraw();                  // 도메인 상태 전이
    auditLogService.record(AuditAction.WITHDRAW, request, actorId); // 명시적 기록
}
```

명시적 호출 대상 액션 (MVP 필수): SUBMIT, APPROVE, REJECT, WITHDRAW, RESUBMIT, ESCALATE, EXPIRE, ROLE_CHANGED

**상태 전이/승인/반려/회수 등 도메인 의미가 강한 액션은 AOP에만 의존하지 않고 반드시 명시적으로 호출한다.** AOP로 처리하면 전이 순간의 before/after 컨텍스트를 정확히 포착하기 어렵고, 조건부 기록 로직 구현이 복잡해진다.

#### 확장 포인트: AOP (`@Auditable`)

AOP는 MVP 구현 범위가 아닌 확장 포인트다. 실행 시각, 호출자 IP, 응답 시간 등 **비즈니스 의미보다 운영/추적 목적이 강한 공통 메타데이터**를 일관 적용할 때 유용하다. MVP 이후 필요 시 `AuditLogAspect`를 추가하여 명시적 호출 방식을 보강하는 형태로 확장한다.

**MVP vs 확장 포인트 구분**:

| 구분 | 방식 | 대상 | 구현 시점 |
|------|------|------|---------|
| MVP 구현 | 명시적 호출 (`auditLogService.record()`) | SUBMIT, APPROVE, REJECT, WITHDRAW, RESUBMIT, ESCALATE, EXPIRE, ROLE_CHANGED | Phase 3 |
| 확장 포인트 | AOP (`@Auditable`) | 실행 시각, 호출자 IP 등 공통 운영 메타데이터 | MVP 이후 |

**방식 비교**:

| 방식 | 장점 | 단점 |
|------|------|------|
| AOP만 사용 | 선언적, 비즈니스 코드 분리 | 전이 순간의 세밀한 컨텍스트 포착 어려움, 디버깅 난이도 증가 |
| **명시적 호출 (MVP 채택)** | 의도 명확, 컨텍스트 정확, 조건부 기록 용이 | 횡단 관심사 코드가 Service에 노출됨 |
| 혼용 (확장 단계) | 핵심 액션은 명시적, 공통 메타데이터는 AOP | 두 방식의 역할 경계를 명확히 유지해야 함 |

---

## 12. 배치 처리 설계

### 12.1 배치 시나리오

| 배치 Job | 실행 주기 | 목적 |
|---------|---------|------|
| OverdueReminderJob | 매일 09:00 | 3일 이상 미처리 결재 단계 리마인드 이벤트 기록 |
| EscalationJob | 매일 09:00 | 7일 이상 미처리 문서를 ESCALATED로 전환 |
| ExpiredRequestJob | 매일 00:00 | 30일 이상 미처리 문서를 EXPIRED로 전환 |
| StatisticsJob | 매일 01:00 | 일별 결재 처리 통계 집계 및 저장 |

### 12.2 배치 필요성

**왜 배치인가?**
- 기한 초과 처리는 사용자 요청 없이 시스템이 자동으로 처리해야 하므로 배치 처리가 적합하다.
- 대량의 미처리 문서를 한 번에 처리하는 것이 효율적이다.
- 통계 집계는 실시간 처리가 불필요하다.

### 12.3 Spring Batch 구조

```
ExpiredRequestJob
├── Step 1: 만료 대상 문서 조회 (Reader)
│   └── JpaPagingItemReader (status IN (PENDING, IN_PROGRESS), submittedAt < now-30days)
├── Step 2: 상태 변경 (Processor)
│   └── ApprovalRequest.expire() 호출
└── Step 3: DB 저장 + 감사 로그 (Writer)
    └── JpaItemWriter + AuditLog 저장
```

### 12.4 운영 고려사항

**재실행 가능성 (Idempotency)**:
- 이미 EXPIRED인 문서를 다시 처리해도 상태가 변하지 않도록 도메인 메서드에서 멱등성 보장
- Spring Batch의 JobInstance + JobParameters로 중복 실행 방지
- `BATCH_JOB_EXECUTION` 테이블로 실행 이력 자동 관리

**중복 처리 방지**:
- Batch Job은 실행 날짜를 JobParameter로 사용
- 같은 날 같은 Job은 이미 COMPLETED 상태이면 재실행 불가 (Spring Batch 기본 동작)
- 실패 시 `FAILED` 상태로 기록되어 재시도 가능

---

## 13. 테스트 전략

### 13.1 테스트 계층 분리

| 계층 | 테스트 대상 | 사용 도구 | 특징 |
|------|-----------|----------|------|
| 단위 테스트 | 도메인 엔티티 (상태 전이 로직) | JUnit 5, Mockito | 외부 의존성 없음, 빠름 |
| 서비스 테스트 | Service 계층 (권한, 비즈니스) | JUnit 5, Mockito | Repository 모킹 |
| 통합 테스트 | Repository + DB | JUnit 5, Testcontainers | 실제 PostgreSQL |
| API 테스트 | Controller + 전체 레이어 | MockMvc, Testcontainers | Spring Context 로드 |
| 배치 테스트 | Batch Job 전체 플로우 | Spring Batch Test, Testcontainers | 실제 DB |

### 13.2 핵심 테스트 케이스

**상태 전이 단위 테스트 (가장 중요)**
```java
class ApprovalRequestTest {

    @Test
    void submit_shouldChangStatusToPending_whenDraft() { ... }

    @Test
    void submit_shouldThrowException_whenNotDraft() { ... }

    @Test
    void withdraw_shouldThrowException_whenStepAlreadyProcessed() { ... }

    @Test
    void approve_shouldChangeToApproved_whenLastStepApproved() { ... }

    @Test
    void reject_shouldChangeToRejected_whenCommentProvided() { ... }

    @Test
    void reject_shouldThrowException_whenCommentMissing() { ... }
}
```

**권한 테스트**
```java
class ApprovalRequestServiceTest {

    @Test
    void withdraw_shouldThrowAuthException_whenNotOwner() { ... }

    @Test
    void approve_shouldThrowAuthException_whenNotAssignedApprover() { ... }

    @Test
    void approve_shouldThrowAuthException_whenNotCurrentStep() { ... }
}
```

**배치 테스트 (Testcontainers)**
```java
@SpringBatchTest
@Testcontainers
class ExpiredRequestJobTest {

    @Test
    void job_shouldExpireOverdueRequests_whenOver30Days() { ... }

    @Test
    void job_shouldNotExpireRequests_whenUnder30Days() { ... }

    @Test
    void job_shouldBeIdempotent_whenRunTwice() { ... }
}
```

### 13.3 Testcontainers 활용 포인트

- Repository 통합 테스트: 실제 PostgreSQL에서 쿼리 검증
- 인덱스 실제 사용 여부 확인 (EXPLAIN ANALYZE)
- 배치 Job 전체 플로우 검증
- 상태 전이 후 DB 상태 검증

---

## 14. 개발 단계 계획

### Phase 1: 기반 설계 및 도메인 구축 (1주차)

| 순서 | 작업 | 산출물 |
|------|------|--------|
| 1 | 프로젝트 초기화 (Spring Boot, Gradle, Docker Compose) | build.gradle, docker-compose.yml |
| 2 | 도메인 엔티티 구현 (User, Department, ApprovalRequest, ApprovalLine, ApprovalStep, ApprovalHistory) | Entity 클래스 |
| 3 | 상태 전이 로직 구현 (ApprovalRequest 도메인 메서드) | 상태 전이 메서드 + 단위 테스트 |
| 4 | 예외 계층 구현 (ApprovalException 계층, ErrorCode, GlobalExceptionHandler) | 예외 클래스 |
| 5 | 데이터베이스 초기화 (Flyway 마이그레이션 스크립트) | V1__init.sql |

### Phase 2: 핵심 기능 구현 (2주차)

| 순서 | 작업 | 산출물 |
|------|------|--------|
| 6 | Repository 구현 및 통합 테스트 (Testcontainers) | Repository + 테스트 |
| 7 | JWT 인증/인가 구현 (Spring Security) | SecurityConfig, JwtTokenProvider |
| 8 | 사용자 관리 API (등록, 조회, 역할 부여) | UserController, UserService |
| 9 | 결재 문서 CRUD API (작성, 수정, 조회, 목록) | ApprovalRequestController |
| 10 | 결재 처리 API (제출, 승인, 반려, 회수) | 결재 처리 Service + Controller |

### Phase 3: 감사 로그 및 배치 (3주차)

| 순서 | 작업 | 산출물 |
|------|------|--------|
| 11 | 감사 로그 구현 (AuditLog 엔티티, Service, 명시적 호출) | AuditLog 계층 |
| 12 | 감사 로그 API (관리자 조회) | AuditLogController |
| 13 | Spring Batch 설정 및 만료 처리 Job | ExpiredRequestJob |
| 14 | 에스컬레이션 배치 Job | EscalationJob |
| 15 | 통계 집계 배치 Job | StatisticsJob |

### Phase 4: 테스트 완성 및 문서화 (4주차)

| 순서 | 작업 | 산출물 |
|------|------|--------|
| 16 | 통합 테스트 완성 (API 레벨, 배치 레벨) | 테스트 커버리지 70% 이상 |
| 17 | 상태 전이 단위 테스트 완성 | 상태 전이 케이스 전수 커버 |
| 18 | Swagger/OpenAPI 문서화 | api-spec.yaml |
| 19 | README, 설계 문서 마무리 | docs/ 완성 |
| 20 | Docker Compose 환경 완성 및 실행 검증 | 실행 가능한 로컬 환경 |

---

## 15. 면접 어필 포인트

### 15.1 핵심 설계 어필 포인트

**Q: 상태 전이 로직을 어떻게 설계했나요?**
> 상태 전이 검증 로직을 Service가 아닌 ApprovalRequest 도메인 객체 내부에 응집시켰습니다. ALLOWED_TRANSITIONS 맵으로 허용 전이를 선언적으로 정의하고, 각 전이마다 도메인 메서드(submit, withdraw, approve 등)를 제공합니다. 이 구조의 장점은 상태 전이 단위 테스트가 Spring Context 없이 순수 Java로 작성 가능하다는 점입니다.

**Q: 권한 검증을 어디에서 하나요?**
> 3단 검증 구조를 사용합니다. API 레벨에서 JWT 인증 → @PreAuthorize로 역할 검증 → Service 레벨에서 소유권과 비즈니스 규칙 검증입니다. API 레벨에서 선언적으로 처리 가능한 것은 거기서 끝내고, 비즈니스 컨텍스트가 필요한 검증(예: "기안자 본인인지", "현재 단계의 결재자인지")은 Service에서 명시적으로 합니다.

**Q: 감사 로그를 왜 별도 테이블로 설계했나요?**
> 두 가지 이유입니다. 첫째, 감사 로그는 불변 이력 데이터이므로 일반 엔티티와 수명주기가 다릅니다. 둘째, 미래에 감사 로그만 별도 저장소(예: Elasticsearch)로 이관할 때 영향 범위를 최소화할 수 있습니다. 또한 actorName을 비정규화해서 저장하는 방식으로, 사용자가 삭제되어도 이력이 보존되도록 설계했습니다.

**Q: 배치 처리에서 중복 실행 방지를 어떻게 하나요?**
> Spring Batch는 JobInstance + JobParameters로 실행 이력을 관리합니다. 실행 날짜를 JobParameter로 사용하면 같은 날 같은 Job은 COMPLETED 상태라면 재실행이 거부됩니다. 도메인 레벨에서도 멱등성을 보장해서, 이미 EXPIRED인 문서를 다시 처리해도 상태가 변하지 않습니다.

### 15.2 면접관 예상 질문

| 질문 | 답변 핵심 키워드 |
|------|----------------|
| "반려 후 재제출 시 결재선은 어떻게 되나요?" | 결재선 초기화, 처음부터 재시작, 이력은 보존 |
| "결재자가 여러 명일 때 동시에 승인하면?" | 순차 결재 구조 (병렬 결재는 확장 기능), 현재 단계 순번으로 제어 |
| "AuditLog가 실패하면 메인 트랜잭션도 롤백되나요?" | 동일 트랜잭션 내 저장, 의도적 설계 - 감사 로그 실패 = 트랜잭션 실패로 처리 |
| "배치 실패 시 어떻게 복구하나요?" | Spring Batch 재시작, 청크 단위 커밋, 부분 실패 시 실패 지점부터 재시작 |
| "테스트 커버리지 전략은?" | 상태 전이 100% 단위 테스트, 통합 테스트는 핵심 플로우 중심 |

---

## 16. Wallet Ledger System과의 차별점

| 비교 항목 | Wallet Ledger System | workflow-approval-system |
|-----------|---------------------|-------------------------|
| 핵심 문제 | 동시성 하에서 금융 정합성 보장 | 복잡한 비즈니스 프로세스 흐름 관리 |
| 상태 관리 | 잔액 변경 (원자적) | 다단계 상태 전이 (결재 플로우) |
| 동시성 제어 | 비관적 락, Idempotency Key | 순차 결재 구조, 단계별 소유권 |
| 권한 구조 | 단순 사용자/관리자 | 역할 계층 + 소유권 + 단계별 검증 |
| 이력 관리 | Double-entry Ledger | 결재 단계별 이력 + 감사 로그 |
| 배치 처리 | 없음 | 만료/에스컬레이션/통계 집계 |
| 테스트 전략 | 동시성 테스트 (CyclicBarrier) | 상태 전이 단위 테스트 + 배치 테스트 |
| 운영 관점 | 원장 정확성, 잔액 일치 | 감사 추적, 결재 현황 모니터링 |

**두 프로젝트를 묶어서 어필하는 방법**:
> "Wallet Ledger System에서는 금융 시스템에서 중요한 동시성 제어와 데이터 정합성을 집중적으로 다뤘습니다. workflow-approval-system에서는 기업 시스템에서 실무적으로 자주 마주치는 복잡한 상태 전이 설계, 역할 기반 권한 제어, 감사 추적, 배치 처리를 다뤘습니다. 두 프로젝트를 통해 데이터 정합성 중심의 시스템과 프로세스 흐름 중심의 시스템, 두 가지 성격의 백엔드 설계 역량을 보여드릴 수 있습니다."

---

## 17. 리스크 및 보완 포인트

### 17.1 설계 리스크

| 리스크 | 영향도 | 보완 방향 |
|--------|--------|----------|
| 병렬 결재 미지원 | 중 | MVP에서는 순차 결재만 지원, 확장 설계만 문서화 |
| 대결/위임 미지원 | 중 | 확장 기능으로 분류, ApprovalStep에 delegatee 필드 추가 구조 제시 |
| 알림 시스템 부재 | 저 | 감사 로그로 대체, 향후 이벤트 기반 알림 확장 포인트 명시 |
| 첨부파일 미지원 | 저 | 포트폴리오 목적상 제외, S3 연동 확장 포인트 제시 |
| 성능 테스트 부재 | 중 | 단일 서비스 기준이므로 적절한 인덱스 설계로 대응 |

### 17.2 기술적 보완 포인트

**JPA N+1 문제**:
- ApprovalRequest 목록 조회 시 approvalLine, steps를 함께 로딩하면 N+1 발생 가능
- 해결: `@EntityGraph` 또는 JPQL fetch join으로 필요 시점에 명시적 로딩

**배치와 일반 트랜잭션 충돌**:
- 배치가 만료 처리하는 동시에 결재자가 승인을 시도하는 경우
- 해결: 낙관적 락(Optimistic Lock) + version 필드로 충돌 감지 및 재시도

**감사 로그 저장 실패**:
- AuditLog 저장 실패가 메인 트랜잭션을 롤백시키는 것이 의도된 설계이지만, 과도한 가용성 영향 가능
- 보완: 감사 로그만 비동기 저장이 필요한 경우 TransactionSynchronization을 활용한 AFTER_COMMIT 훅으로 분리 가능 (단, MVP에서는 동기 저장 유지)

### 17.3 포트폴리오 관점 보완

- 실제 동작하는 Docker Compose 환경을 제공해야 평가자가 직접 실행 가능
- Swagger UI를 통해 API 탐색 가능하게 구성
- README에 "설계 의도" 섹션을 별도로 작성하여 단순 기능 목록 이상의 가치를 전달
