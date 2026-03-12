# API 설계 상세

> 관련 문서: design.md §8, authorization.md
> Base URL: `/api/v1`
> 인증 방식: Bearer JWT Token
> 응답 형식: JSON

---

## 목차

1. [공통 규칙](#1-공통-규칙)
2. [인증 API](#2-인증-api)
3. [사용자 관리 API](#3-사용자-관리-api)
4. [결재 문서 API](#4-결재-문서-api)
5. [결재 처리 API](#5-결재-처리-api)
6. [감사 로그 API](#6-감사-로그-api)
7. [에러 코드 목록](#7-에러-코드-목록)

---

## 1. 공통 규칙

### 1.1 요청 헤더

| 헤더 | 필수 여부 | 설명 |
|------|----------|------|
| Authorization | 필수 (로그인 제외) | `Bearer {jwt_token}` |
| Content-Type | POST/PUT 시 필수 | `application/json` |

### 1.2 성공 응답 구조

```json
{
  "success": true,
  "data": { /* 응답 본문 */ },
  "timestamp": "2026-03-11T09:00:00"
}
```

목록 응답:
```json
{
  "success": true,
  "data": {
    "content": [ /* 아이템 목록 */ ],
    "totalElements": 100,
    "totalPages": 10,
    "size": 10,
    "number": 0
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

### 1.3 에러 응답 구조

```json
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

유효성 검증 에러 (400):
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "입력값이 올바르지 않습니다.",
    "details": [
      { "field": "title", "message": "제목은 필수입니다." },
      { "field": "content", "message": "내용은 필수입니다." }
    ]
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

### 1.4 HTTP 상태코드 정책

| 상황 | HTTP 상태코드 |
|------|-------------|
| 성공 (조회) | 200 OK |
| 성공 (생성) | 201 Created |
| 성공 (액션, 수정) | 200 OK |
| 인증 실패 | 401 Unauthorized |
| 권한 없음 | 403 Forbidden |
| 리소스 없음 | 404 Not Found |
| 유효성 오류 | 400 Bad Request |
| 상태 전이 오류 | 409 Conflict |
| 서버 오류 | 500 Internal Server Error |

---

## 2. 인증 API

### 2.1 로그인

```
POST /api/v1/auth/login
```

**요청자**: 모든 사용자 (인증 불필요)

**요청 본문**:
```json
{
  "username": "kim.chulsu",
  "password": "password123!"
}
```

**응답 (200)**:
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "user": {
      "id": 1,
      "username": "kim.chulsu",
      "name": "김철수",
      "roles": ["DRAFTER", "APPROVER"]
    }
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| 존재하지 않는 사용자 | 401 | INVALID_CREDENTIALS |
| 잘못된 비밀번호 | 401 | INVALID_CREDENTIALS |
| 비활성 계정 | 401 | ACCOUNT_INACTIVE |

---

## 3. 사용자 관리 API

### 3.1 사용자 등록

```
POST /api/v1/users
```

**요청자**: ROLE_ADMIN
**인증**: 필요

**요청 본문**:
```json
{
  "username": "lee.younghee",
  "email": "younghee@company.com",
  "password": "initialPass123!",
  "name": "이영희",
  "departmentId": 2,
  "roles": ["DRAFTER", "APPROVER"]
}
```

**응답 (201)**:
```json
{
  "success": true,
  "data": {
    "id": 5,
    "username": "lee.younghee",
    "email": "younghee@company.com",
    "name": "이영희",
    "department": {
      "id": 2,
      "name": "개발팀",
      "code": "DEV-01"
    },
    "roles": ["DRAFTER", "APPROVER"],
    "status": "ACTIVE",
    "createdAt": "2026-03-11T09:00:00"
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| 중복 username | 400 | DUPLICATE_USERNAME |
| 중복 email | 400 | DUPLICATE_EMAIL |
| 존재하지 않는 부서 | 404 | DEPARTMENT_NOT_FOUND |
| ADMIN 권한 없음 | 403 | ACCESS_DENIED |

### 3.2 사용자 조회

```
GET /api/v1/users/{userId}
```

**요청자**: ROLE_ADMIN
**응답**: UserResponse (등록과 동일 구조)

---

## 4. 결재 문서 API

### 4.1 결재 문서 초안 작성

```
POST /api/v1/approval-requests
```

**요청자**: ROLE_DRAFTER
**목적**: 결재 문서를 DRAFT 상태로 생성한다. 결재선은 선택적으로 함께 구성할 수 있다.

**요청 본문**:
```json
{
  "title": "2026년 3월 연차 휴가 신청",
  "content": "2026년 3월 20일(금) 연차 휴가를 신청합니다.",
  "category": "LEAVE",
  "dueDate": "2026-03-15",
  "approvalSteps": [
    {
      "stepOrder": 1,
      "approverId": 10,
      "roleLabel": "팀장"
    },
    {
      "stepOrder": 2,
      "approverId": 15,
      "roleLabel": "부서장"
    }
  ]
}
```

**응답 (201)**:
```json
{
  "success": true,
  "data": {
    "id": 42,
    "title": "2026년 3월 연차 휴가 신청",
    "content": "2026년 3월 20일(금) 연차 휴가를 신청합니다.",
    "category": "LEAVE",
    "status": "DRAFT",
    "drafter": {
      "id": 1,
      "name": "김철수"
    },
    "department": {
      "id": 2,
      "name": "개발팀"
    },
    "approvalLine": {
      "steps": [
        { "stepOrder": 1, "approver": { "id": 10, "name": "이영희" }, "roleLabel": "팀장", "status": "WAITING" },
        { "stepOrder": 2, "approver": { "id": 15, "name": "박민준" }, "roleLabel": "부서장", "status": "WAITING" }
      ]
    },
    "dueDate": "2026-03-15",
    "submittedAt": null,
    "completedAt": null,
    "createdAt": "2026-03-11T09:00:00"
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| 제목/내용 누락 | 400 | VALIDATION_FAILED |
| 유효하지 않은 결재자 ID | 404 | USER_NOT_FOUND |
| 결재자가 APPROVER 역할 없음 | 400 | INVALID_APPROVER |
| 결재 순서 중복 | 400 | INVALID_STEP_ORDER |

### 4.2 결재 문서 목록 조회

```
GET /api/v1/approval-requests?status=PENDING&category=LEAVE&page=0&size=10&sort=submittedAt,desc
```

**요청자**: 역할에 따라 조회 범위 다름
**목적**: 조건에 맞는 결재 문서 목록을 조회한다.

**Query Parameters**:
| 파라미터 | 타입 | 설명 | 기본값 |
|---------|------|------|--------|
| status | String | 상태 필터 (DRAFT, PENDING, ...) | 없음 |
| category | String | 카테고리 필터 | 없음 |
| drafterId | Long | 기안자 ID 필터 | 없음 |
| page | int | 페이지 번호 (0부터) | 0 |
| size | int | 페이지 크기 | 10 |
| sort | String | 정렬 기준 | createdAt,desc |

**조회 범위 (역할별)**:
- DRAFTER: 본인이 기안한 문서
- APPROVER: 본인이 결재자로 포함된 문서
- DEPT_MANAGER: 소속 부서 내 모든 문서
- ADMIN: 전체 문서

**응답 (200)**: 페이지네이션 목록 응답

### 4.3 결재 문서 상세 조회

```
GET /api/v1/approval-requests/{requestId}
```

**요청자**: 해당 문서에 접근 권한이 있는 사용자
**목적**: 결재선, 이력, 처리 현황을 포함한 상세 정보를 반환한다.

**응답 (200)**:
```json
{
  "success": true,
  "data": {
    "id": 42,
    "title": "2026년 3월 연차 휴가 신청",
    "content": "2026년 3월 20일(금) 연차 휴가를 신청합니다.",
    "category": "LEAVE",
    "status": "IN_PROGRESS",
    "drafter": { "id": 1, "name": "김철수" },
    "department": { "id": 2, "name": "개발팀" },
    "approvalLine": {
      "steps": [
        {
          "stepOrder": 1,
          "approver": { "id": 10, "name": "이영희" },
          "roleLabel": "팀장",
          "status": "APPROVED",
          "history": {
            "action": "APPROVED",
            "comment": "승인합니다.",
            "processedAt": "2026-03-11T10:30:00"
          }
        },
        {
          "stepOrder": 2,
          "approver": { "id": 15, "name": "박민준" },
          "roleLabel": "부서장",
          "status": "IN_PROGRESS",
          "history": null
        }
      ]
    },
    "dueDate": "2026-03-15",
    "submittedAt": "2026-03-11T09:00:00",
    "completedAt": null,
    "createdAt": "2026-03-11T08:30:00"
  },
  "timestamp": "2026-03-11T11:00:00"
}
```

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| 존재하지 않는 문서 | 404 | REQUEST_NOT_FOUND |
| 접근 권한 없음 | 403 | ACCESS_DENIED |

### 4.4 결재 문서 수정

```
PUT /api/v1/approval-requests/{requestId}
```

**요청자**: 기안자 본인 (DRAFTER)
**조건**: DRAFT 상태인 문서만 수정 가능

**요청 본문**: 작성과 동일 구조

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| DRAFT가 아닌 상태 | 409 | INVALID_STATE_TRANSITION |
| 기안자 본인 아님 | 403 | NOT_REQUEST_OWNER |

### 4.5 결재 문서 제출

```
POST /api/v1/approval-requests/{requestId}/submit
```

**요청자**: 기안자 본인 (DRAFTER)
**목적**: DRAFT 상태의 문서를 PENDING으로 전환하고 결재를 시작한다.

**요청 본문**: 없음

**응답 (200)**:
```json
{
  "success": true,
  "data": {
    "id": 42,
    "status": "PENDING",
    "submittedAt": "2026-03-11T09:00:00"
  },
  "timestamp": "2026-03-11T09:00:00"
}
```

**검증 내용**:
1. 현재 상태 = DRAFT
2. 기안자 본인 확인
3. 결재선 1단계 이상 존재
4. 제목, 내용 필수 값 존재

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| DRAFT 아님 | 409 | INVALID_STATE_TRANSITION |
| 결재선 없음 | 400 | APPROVAL_LINE_REQUIRED |
| 기안자 본인 아님 | 403 | NOT_REQUEST_OWNER |

### 4.6 결재 문서 회수

```
POST /api/v1/approval-requests/{requestId}/withdraw
```

**요청자**: 기안자 본인 (DRAFTER)
**목적**: PENDING 상태이고 아직 처리된 단계가 없는 문서를 WITHDRAWN으로 회수한다.

**요청 본문**: 없음

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| PENDING 아님 | 409 | INVALID_STATE_TRANSITION |
| 처리된 단계 존재 | 409 | CANNOT_WITHDRAW_IN_PROGRESS |
| 기안자 본인 아님 | 403 | NOT_REQUEST_OWNER |

### 4.7 결재 문서 재제출

```
POST /api/v1/approval-requests/{requestId}/resubmit
```

**요청자**: 기안자 본인 (DRAFTER)
**목적**: REJECTED 또는 WITHDRAWN 상태의 문서를 수정 후 재제출한다. 결재선이 초기화되고 처음부터 재시작된다.

**요청 본문** (수정 내용 포함):
```json
{
  "title": "수정된 제목",
  "content": "수정된 내용",
  "approvalSteps": [
    { "stepOrder": 1, "approverId": 10, "roleLabel": "팀장" },
    { "stepOrder": 2, "approverId": 15, "roleLabel": "부서장" }
  ]
}
```

**처리 내용**:
- 기존 결재 이력(ApprovalHistory)은 보존
- 결재 단계(ApprovalStep) 상태 초기화 (모두 WAITING)
- 1단계 ApprovalStep → IN_PROGRESS
- ApprovalRequest.status → PENDING

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| REJECTED/WITHDRAWN 아님 | 409 | INVALID_STATE_TRANSITION |
| 결재선 없음 | 400 | APPROVAL_LINE_REQUIRED |
| 기안자 본인 아님 | 403 | NOT_REQUEST_OWNER |

---

## 5. 결재 처리 API

### 5.1 내 결재 목록 조회

```
GET /api/v1/approval-requests/my-tasks?page=0&size=10
```

**요청자**: ROLE_APPROVER
**목적**: 본인이 현재 처리해야 할 결재 단계(IN_PROGRESS)가 있는 문서 목록을 반환한다.

**응답 (200)**: 페이지네이션 목록. 각 아이템에 문서 요약 + 현재 내 담당 단계 정보 포함.

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "requestId": 42,
        "title": "2026년 3월 연차 휴가 신청",
        "category": "LEAVE",
        "drafter": { "id": 1, "name": "김철수" },
        "myStep": {
          "stepId": 88,
          "stepOrder": 2,
          "roleLabel": "부서장",
          "status": "IN_PROGRESS"
        },
        "submittedAt": "2026-03-11T09:00:00",
        "dueDate": "2026-03-15"
      }
    ],
    "totalElements": 3,
    "totalPages": 1,
    "size": 10,
    "number": 0
  }
}
```

### 5.2 결재 단계 승인

```
POST /api/v1/approval-requests/{requestId}/steps/{stepId}/approve
```

**요청자**: 해당 단계의 결재자 (ROLE_APPROVER)
**목적**: 지정된 결재 단계를 승인 처리한다.

**요청 본문**:
```json
{
  "comment": "검토 후 승인합니다."
}
```
> comment는 선택 사항 (승인 시 생략 가능)

**응답 (200)**:
```json
{
  "success": true,
  "data": {
    "requestId": 42,
    "requestStatus": "APPROVED",
    "stepId": 88,
    "stepStatus": "APPROVED",
    "processedAt": "2026-03-11T11:00:00"
  },
  "timestamp": "2026-03-11T11:00:00"
}
```

**검증 내용**:
1. 존재하는 결재 문서 및 단계
2. 결재 단계의 담당 결재자 = 현재 로그인 사용자
3. 해당 단계 상태 = IN_PROGRESS

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| 문서/단계 없음 | 404 | REQUEST_NOT_FOUND / STEP_NOT_FOUND |
| 담당 결재자 아님 | 403 | NOT_ASSIGNED_APPROVER |
| 현재 처리 단계 아님 | 403 | NOT_CURRENT_STEP_APPROVER |
| 이미 처리된 단계 | 409 | ALREADY_PROCESSED |

### 5.3 결재 단계 반려

```
POST /api/v1/approval-requests/{requestId}/steps/{stepId}/reject
```

**요청자**: 해당 단계의 결재자 (ROLE_APPROVER)
**목적**: 지정된 결재 단계를 반려 처리한다.

**요청 본문**:
```json
{
  "comment": "예산 초과로 인해 반려합니다. 금액을 조정 후 재제출 바랍니다."
}
```
> comment는 **필수** (반려 사유 명시 필요)

**응답 (200)**:
```json
{
  "success": true,
  "data": {
    "requestId": 42,
    "requestStatus": "REJECTED",
    "stepId": 88,
    "stepStatus": "REJECTED",
    "processedAt": "2026-03-11T11:00:00"
  },
  "timestamp": "2026-03-11T11:00:00"
}
```

**예외 케이스**:
| 상황 | HTTP | ErrorCode |
|------|------|-----------|
| 반려 코멘트 없음 | 400 | REJECT_COMMENT_REQUIRED |
| 담당 결재자 아님 | 403 | NOT_ASSIGNED_APPROVER |
| 현재 처리 단계 아님 | 403 | NOT_CURRENT_STEP_APPROVER |
| 이미 처리된 단계 | 409 | ALREADY_PROCESSED |

---

## 6. 감사 로그 API

### 6.1 감사 로그 목록 조회

```
GET /api/v1/audit-logs?entityType=ApprovalRequest&action=REJECT&from=2026-03-01&to=2026-03-31&page=0&size=20
```

**요청자**: ROLE_ADMIN 전용
**목적**: 시스템 전체 감사 로그를 조건으로 조회한다.

**Query Parameters**:
| 파라미터 | 타입 | 설명 |
|---------|------|------|
| entityType | String | 엔티티 타입 필터 |
| action | String | 액션 타입 필터 |
| actorId | Long | 행위자 ID 필터 |
| from | LocalDate | 조회 시작 날짜 |
| to | LocalDate | 조회 종료 날짜 |
| page, size | int | 페이지네이션 |

**응답 (200)**:
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 1001,
        "entityType": "ApprovalRequest",
        "entityId": 42,
        "action": "REJECT",
        "actorId": 15,
        "actorName": "박민준",
        "beforeState": "{\"status\": \"IN_PROGRESS\"}",
        "afterState": "{\"status\": \"REJECTED\"}",
        "ipAddress": "192.168.1.100",
        "occurredAt": "2026-03-11T11:00:00"
      }
    ],
    "totalElements": 1,
    "totalPages": 1
  }
}
```

### 6.2 특정 엔티티의 감사 로그 조회

```
GET /api/v1/audit-logs/ApprovalRequest/42
```

**요청자**: ROLE_ADMIN
**목적**: 특정 결재 문서(ID=42)의 전체 변경 이력을 시간순으로 반환한다.

---

## 7. 에러 코드 목록

| ErrorCode | HTTP | 설명 |
|-----------|------|------|
| INVALID_STATE_TRANSITION | 409 | 허용되지 않는 상태 전이 |
| ALREADY_PROCESSED | 409 | 이미 처리된 결재 단계 |
| CANNOT_WITHDRAW_IN_PROGRESS | 409 | 처리된 단계가 있어 회수 불가 |
| NOT_REQUEST_OWNER | 403 | 결재 문서의 기안자가 아님 |
| NOT_ASSIGNED_APPROVER | 403 | 해당 단계의 담당 결재자가 아님 |
| NOT_CURRENT_STEP_APPROVER | 403 | 현재 처리 중인 단계의 담당자가 아님 |
| ACCESS_DENIED | 403 | 일반적 접근 거부 |
| APPROVAL_LINE_REQUIRED | 400 | 결재선이 없음 |
| REJECT_COMMENT_REQUIRED | 400 | 반려 시 코멘트 필수 |
| INVALID_STEP_ORDER | 400 | 결재 순서 오류 (중복, 불연속) |
| INVALID_APPROVER | 400 | 결재자로 지정 불가한 사용자 |
| DUPLICATE_USERNAME | 400 | 중복 사용자명 |
| DUPLICATE_EMAIL | 400 | 중복 이메일 |
| REQUEST_NOT_FOUND | 404 | 결재 문서 없음 |
| STEP_NOT_FOUND | 404 | 결재 단계 없음 |
| USER_NOT_FOUND | 404 | 사용자 없음 |
| DEPARTMENT_NOT_FOUND | 404 | 부서 없음 |
| INVALID_CREDENTIALS | 401 | 잘못된 인증 정보 |
| ACCOUNT_INACTIVE | 401 | 비활성 계정 |
| VALIDATION_FAILED | 400 | 입력값 유효성 오류 |
| INTERNAL_SERVER_ERROR | 500 | 서버 내부 오류 |
