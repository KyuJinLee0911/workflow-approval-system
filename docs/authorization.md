# 권한 설계 상세

> 관련 문서: design.md §6, state-transition.md
> 핵심 원칙: 3단 검증 구조 (인증 → 역할 → 소유권/비즈니스 규칙)

---

## 목차

1. [권한 설계 원칙](#1-권한-설계-원칙)
2. [역할(Role) 정의](#2-역할role-정의)
3. [Permission 정의](#3-permission-정의)
4. [역할별 허용 동작 매트릭스](#4-역할별-허용-동작-매트릭스)
5. [3단 검증 구조 상세](#5-3단-검증-구조-상세)
6. [소유권 검증 설계](#6-소유권-검증-설계)
7. [구현 예시](#7-구현-예시)
8. [보안 고려사항](#8-보안-고려사항)

---

## 1. 권한 설계 원칙

### 1.1 핵심 설계 원칙

**원칙 1: 역할은 의미 있는 비즈니스 역할로 정의한다**
단순 USER/ADMIN 이분법이 아니라, 결재 시스템의 실제 역할(기안자, 결재자, 참조자, 부서 관리자)을 반영한다.

**원칙 2: 3단 검증 구조**
```
1단계 (API 레벨): 인증 여부 - JWT 토큰 유효성
2단계 (API 레벨): 역할(Role) - @PreAuthorize("hasRole('APPROVER')")
3단계 (Service 레벨): 소유권 + 비즈니스 규칙 - "현재 단계의 담당자인가?"
```

**원칙 3: API 레벨과 Service 레벨을 명확히 분리한다**
- API 레벨 검증: 선언적, 횡단 관심사, 역할 존재 여부만 확인
- Service 레벨 검증: 명시적, 비즈니스 컨텍스트 필요, 구체적 소유권 확인

**원칙 4: 실패 예외는 의미 있는 정보를 포함한다**
단순 "접근 금지"가 아니라 왜 거부됐는지 ErrorCode로 전달한다.

### 1.2 한 사용자의 복수 역할

한 사용자가 여러 역할을 동시에 보유할 수 있다.

```
팀장 김철수:
  - ROLE_DRAFTER     (본인 문서 기안 가능)
  - ROLE_APPROVER    (팀원 문서 결재 가능)
  - ROLE_DEPT_MANAGER (팀 내 현황 조회 가능)

신입 이지민:
  - ROLE_DRAFTER     (문서 기안 가능)
  - ROLE_VIEWER      (지정된 문서 열람 가능)

시스템 관리자:
  - ROLE_ADMIN       (전체 권한)
```

---

## 2. 역할(Role) 정의

| 역할 코드 | DB 저장값 | 역할명 | 설명 |
|-----------|-----------|--------|------|
| ROLE_DRAFTER | DRAFTER | 기안자 | 결재 문서를 작성하고 제출할 수 있는 역할 |
| ROLE_APPROVER | APPROVER | 결재자 | 결재선에 포함되어 승인/반려 처리하는 역할 |
| ROLE_VIEWER | VIEWER | 참조자 | 지정된 문서를 열람할 수 있는 역할 (처리 권한 없음) |
| ROLE_DEPT_MANAGER | DEPT_MANAGER | 부서 관리자 | 소속 부서 내 결재 현황 조회 및 에스컬레이션 관리 |
| ROLE_ADMIN | ADMIN | 시스템 관리자 | 전체 시스템 권한 보유, 감사 로그 조회 |

### 2.1 역할 부여 정책

- 역할 부여: ROLE_ADMIN만 가능
- 역할 변경: ROLE_ADMIN만 가능, 변경 이력은 AuditLog에 기록
- 역할 제거: ROLE_ADMIN만 가능 (단, 처리 중인 결재 단계가 있으면 즉시 제거 불가 - 확장 기능)

---

## 3. Permission 정의

역할(Role)과 권한(Permission)을 분리한다. Role은 사용자에게 부여하고, Permission은 Role에 매핑된다.

| Permission | 설명 | 보유 Role |
|-----------|------|----------|
| REQUEST_CREATE | 결재 문서 초안 작성 | DRAFTER, ADMIN |
| REQUEST_SUBMIT | 결재 문서 제출 | DRAFTER, ADMIN |
| REQUEST_WITHDRAW | 결재 문서 회수 | DRAFTER, ADMIN |
| REQUEST_RESUBMIT | 결재 문서 재제출 | DRAFTER, ADMIN |
| REQUEST_UPDATE | 결재 문서 수정 (DRAFT 상태) | DRAFTER, ADMIN |
| STEP_APPROVE | 결재 단계 승인 | APPROVER, ADMIN |
| STEP_REJECT | 결재 단계 반려 | APPROVER, ADMIN |
| REQUEST_VIEW_OWN | 본인 결재 문서 조회 | DRAFTER, APPROVER, VIEWER, DEPT_MANAGER, ADMIN |
| REQUEST_VIEW_DEPT | 부서 내 결재 문서 조회 | DEPT_MANAGER, ADMIN |
| REQUEST_VIEW_ALL | 전체 결재 문서 조회 | ADMIN |
| AUDIT_LOG_VIEW | 감사 로그 조회 | ADMIN |
| USER_MANAGE | 사용자 관리 | ADMIN |
| BATCH_MONITOR | 배치 실행 현황 조회 | ADMIN |

**MVP에서 Permission 레벨 구현 여부**: MVP에서는 Role 기반 @PreAuthorize로 구현하고, Permission 개념은 문서에만 정의한다. 확장 시 Spring Security의 `hasAuthority()`로 Permission 레벨 세분화 가능.

---

## 4. 역할별 허용 동작 매트릭스

### 4.1 결재 문서 관련

| 동작 | DRAFTER | APPROVER | VIEWER | DEPT_MANAGER | ADMIN |
|------|:-------:|:--------:|:------:|:------------:|:-----:|
| 초안 작성 | O (본인) | X | X | O | O |
| 문서 수정 | O (본인, DRAFT만) | X | X | X | O |
| 제출 | O (본인) | X | X | X | O |
| 회수 | O (본인*) | X | X | X | O |
| 재제출 | O (본인) | X | X | X | O |
| 내 문서 조회 | O | O (담당) | O (지정) | O (부서 내) | O |
| 전체 문서 조회 | X | X | X | X | O |

> *PENDING 상태이고 아직 처리된 단계 없는 경우만

### 4.2 결재 처리 관련

| 동작 | DRAFTER | APPROVER | VIEWER | DEPT_MANAGER | ADMIN |
|------|:-------:|:--------:|:------:|:------------:|:-----:|
| 내 결재 목록 조회 | X | O | X | X | O |
| 단계 승인 | X | O (담당 단계) | X | X | O |
| 단계 반려 | X | O (담당 단계) | X | X | O |

### 4.3 관리 기능

| 동작 | DRAFTER | APPROVER | VIEWER | DEPT_MANAGER | ADMIN |
|------|:-------:|:--------:|:------:|:------------:|:-----:|
| 감사 로그 조회 | X | X | X | X | O |
| 사용자 등록 | X | X | X | X | O |
| 역할 부여 | X | X | X | X | O |
| 배치 모니터링 | X | X | X | X | O |
| 에스컬레이션 이관 | X | X | X | O (부서 내) | O |

---

## 5. 3단 검증 구조 상세

### 5.1 1단계: JWT 인증 (Spring Security Filter)

```
모든 요청 → JwtAuthenticationFilter
  → JWT 토큰 존재 여부 확인
  → 토큰 유효성 검증 (서명, 만료시간)
  → SecurityContext에 Authentication 저장
  → 실패 시: 401 Unauthorized
```

보호 예외: `/api/v1/auth/login` (인증 불필요)

### 5.2 2단계: 역할 검증 (@PreAuthorize)

```java
// Controller 레벨에서 선언적 역할 검증
@PostMapping("/{id}/submit")
@PreAuthorize("hasRole('DRAFTER') or hasRole('ADMIN')")
public ResponseEntity<ApiResponse<Void>> submit(@PathVariable Long id, ...) { ... }

@PostMapping("/{id}/steps/{stepId}/approve")
@PreAuthorize("hasRole('APPROVER') or hasRole('ADMIN')")
public ResponseEntity<ApiResponse<Void>> approve(...) { ... }

@GetMapping("/audit-logs")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<ApiResponse<List<AuditLogResponse>>> getAuditLogs() { ... }
```

**실패 처리**: `AccessDeniedException` → GlobalExceptionHandler → 403 Forbidden

### 5.3 3단계: 소유권 및 비즈니스 규칙 검증 (Service 레벨)

2단계 역할 검증을 통과해도, 비즈니스 컨텍스트에 따른 추가 검증이 필요하다.

```
예시:
- 역할 검증: "이 사람은 APPROVER 역할을 가지고 있는가?" (2단계)
- 소유권 검증: "이 사람이 이 결재 단계의 담당자인가?" (3단계)
- 비즈니스 검증: "이 단계가 현재 IN_PROGRESS 상태인가?" (3단계)
```

**3단계 검증 예외 타입**: `ApprovalAuthorizationException` (HTTP 403)

---

## 6. 소유권 검증 설계

### 6.1 소유권 유형

| 소유권 유형 | 검증 대상 | 검증 로직 |
|-----------|----------|---------|
| 기안자 소유권 | 결재 문서 회수, 수정, 재제출 | `request.getDrafter().getId().equals(actorId)` |
| 결재 단계 소유권 | 단계 승인/반려 | `step.getApprover().getId().equals(actorId)` |
| 현재 단계 검증 | 단계 승인/반려 | `step.getStatus() == StepStatus.IN_PROGRESS` |
| 부서 소속 검증 | 부서 관리자의 부서 내 접근 | `request.getDepartment().getId().equals(actor.getDepartment().getId())` |

### 6.2 소유권 검증 헬퍼 메서드

소유권 검증 로직은 도메인 객체나 전용 검증 클래스에 응집한다.

```java
// ApprovalRequest.java 내 소유권 메서드
public boolean isDraftedBy(Long userId) {
    return this.drafter.getId().equals(userId);
}

// ApprovalStep.java 내 소유권 메서드
public boolean isAssignedTo(Long userId) {
    return this.approver.getId().equals(userId);
}

public boolean isCurrentlyProcessable() {
    return this.status == StepStatus.IN_PROGRESS;
}
```

### 6.3 복합 검증 예시

```java
// ApprovalRequestService.java
private void validateStepApprover(ApprovalStep step, Long actorId) {
    // 담당자 검증
    if (!step.isAssignedTo(actorId)) {
        throw new ApprovalAuthorizationException(
            ErrorCode.NOT_ASSIGNED_APPROVER,
            "해당 결재 단계의 담당자가 아닙니다."
        );
    }
    // 현재 처리 단계 검증
    if (!step.isCurrentlyProcessable()) {
        throw new ApprovalAuthorizationException(
            ErrorCode.NOT_CURRENT_STEP_APPROVER,
            "현재 처리 중인 단계가 아닙니다. (현재 단계 상태: " + step.getStatus() + ")"
        );
    }
}
```

---

## 7. 구현 예시

### 7.1 SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)  // 401
                .accessDeniedHandler(jwtAccessDeniedHandler)             // 403
            );
        return http.build();
    }
}
```

### 7.2 Controller 레벨 선언적 권한

```java
@RestController
@RequestMapping("/api/v1/approval-requests")
public class ApprovalRequestController {

    // DRAFTER만 문서 생성 가능
    @PostMapping
    @PreAuthorize("hasRole('DRAFTER') or hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<ApprovalRequestResponse>> create(...) { ... }

    // 제출: 기안자 역할 + Service에서 소유권 확인
    @PostMapping("/{id}/submit")
    @PreAuthorize("hasRole('DRAFTER') or hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<Void>> submit(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal userPrincipal) {
        approvalRequestService.submit(id, userPrincipal.getId());
        return ResponseEntity.ok(ApiResponse.success());
    }

    // 승인: APPROVER 역할 + Service에서 단계 담당자 확인
    @PostMapping("/{id}/steps/{stepId}/approve")
    @PreAuthorize("hasRole('APPROVER') or hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<Void>> approve(
            @PathVariable Long id,
            @PathVariable Long stepId,
            @RequestBody ApproveRequest request,
            @AuthenticationPrincipal UserPrincipal userPrincipal) {
        approvalRequestService.approve(id, stepId, userPrincipal.getId(), request.getComment());
        return ResponseEntity.ok(ApiResponse.success());
    }
}
```

### 7.3 Service 레벨 소유권 검증

```java
@Service
@Transactional
public class ApprovalRequestService {

    public void submit(Long requestId, Long actorId) {
        ApprovalRequest request = findRequestById(requestId);

        // 3단계: 소유권 검증
        if (!request.isDraftedBy(actorId)) {
            throw new ApprovalAuthorizationException(
                ErrorCode.NOT_REQUEST_OWNER, "기안자 본인만 제출할 수 있습니다."
            );
        }

        // 도메인 메서드 호출 (상태 전이 + 비즈니스 검증)
        request.submit();

        // 감사 로그
        auditLogService.record(AuditAction.SUBMIT, request, actorId);
    }

    public void reject(Long requestId, Long stepId, Long actorId, String comment) {
        ApprovalRequest request = findRequestById(requestId);
        ApprovalStep step = findStepById(stepId);

        // 3단계: 단계 담당자 검증
        validateStepApprover(step, actorId);

        // 도메인 메서드 호출
        step.reject(comment);
        request.onStepRejected(step);

        // 감사 로그
        auditLogService.record(AuditAction.REJECT, request, actorId, comment);
    }
}
```

---

## 8. 보안 고려사항

### 8.1 IDOR (Insecure Direct Object Reference) 방지

URL에 포함된 리소스 ID를 그대로 신뢰하지 않는다. 항상 서비스 레벨에서 소유권을 재검증한다.

```
나쁜 예: /api/v1/approval-requests/123/submit
→ requestId=123만 검증하고 실행 (IDOR 취약)

좋은 예: requestId=123 조회 후 request.isDraftedBy(actorId) 검증 (IDOR 방지)
```

### 8.2 역할 에스컬레이션 방지

- 사용자는 자신의 역할을 변경할 수 없다.
- 역할 변경 API는 ADMIN만 호출 가능하다.
- 역할 변경은 반드시 AuditLog에 기록된다.

### 8.3 에러 응답의 정보 최소화

권한 오류 시 리소스 존재 여부를 노출하지 않는다.

```java
// 나쁜 예: 리소스 존재를 노출
throw new ApprovalNotFoundException("ID=123인 결재 문서가 없습니다.");  // 403 상황에서 404 반환

// 좋은 예: 권한 오류 통일
// 리소스가 없거나 접근 권한이 없으면 동일하게 403 반환 (정보 최소화)
```

> **MVP 결정**: MVP에서는 명확한 오류 메시지를 우선한다. 보안 강화는 확장 기능으로 분류.

### 8.4 JWT 토큰 관리

| 항목 | 설정값 | 이유 |
|------|--------|------|
| Access Token 만료 | 1시간 | 탈취 시 피해 최소화 |
| Refresh Token | MVP에서는 미구현 | 복잡도 증가, 포트폴리오 목적 외 |
| 알고리즘 | HS256 | 단일 서비스 기준 충분, RS256은 분산 환경용 |
| 클레임 | userId, username, roles | 매 요청 DB 조회 없이 권한 판단 가능 |
