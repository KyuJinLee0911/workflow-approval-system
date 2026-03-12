# DB 설계 상세 (ERD)

> 관련 문서: design.md §9
> DB: PostgreSQL
> ORM: Spring Data JPA (Hibernate)
> 마이그레이션: Flyway

---

## 목차

1. [테이블 목록](#1-테이블-목록)
2. [테이블 상세 정의](#2-테이블-상세-정의)
3. [ERD 관계도 (텍스트)](#3-erd-관계도-텍스트)
4. [인덱스 전략](#4-인덱스-전략)
5. [설계 결정 사항](#5-설계-결정-사항)
6. [Flyway 마이그레이션 스크립트](#6-flyway-마이그레이션-스크립트)

---

## 1. 테이블 목록

| 테이블명 | 역할 | 레코드 수명 |
|---------|------|-----------|
| departments | 부서/조직 | 반영구 |
| users | 시스템 사용자 | 반영구 (soft delete) |
| user_roles | 사용자-역할 매핑 | 변경 가능 |
| approval_requests | 결재 문서 (핵심) | 영구 보존 |
| approval_lines | 결재선 | approval_request와 동일 |
| approval_steps | 결재 단계 | approval_request와 동일 |
| approval_histories | 결재 처리 이력 | 영구 보존 (불변) |
| audit_logs | 감사 로그 | 영구 보존 (불변) |

---

## 2. 테이블 상세 정의

### 2.1 departments

```sql
CREATE TABLE departments (
    id              BIGSERIAL       PRIMARY KEY,
    name            VARCHAR(100)    NOT NULL,
    code            VARCHAR(50)     NOT NULL UNIQUE,
    parent_id       BIGINT          REFERENCES departments(id) ON DELETE SET NULL,
    manager_id      BIGINT          REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT now()
);

COMMENT ON TABLE departments IS '부서/조직 정보';
COMMENT ON COLUMN departments.code IS '부서 식별 코드 (예: DEV-01, SALES-02)';
COMMENT ON COLUMN departments.parent_id IS '상위 부서 ID (NULL이면 최상위 부서)';
```

### 2.2 users

```sql
CREATE TABLE users (
    id              BIGSERIAL       PRIMARY KEY,
    username        VARCHAR(50)     NOT NULL UNIQUE,
    email           VARCHAR(200)    NOT NULL UNIQUE,
    password_hash   VARCHAR(255)    NOT NULL,
    name            VARCHAR(100)    NOT NULL,
    department_id   BIGINT          REFERENCES departments(id) ON DELETE SET NULL,
    status          VARCHAR(20)     NOT NULL DEFAULT 'ACTIVE'
                                    CHECK (status IN ('ACTIVE', 'INACTIVE')),
    created_at      TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT now()
);

COMMENT ON TABLE users IS '시스템 사용자';
COMMENT ON COLUMN users.status IS 'ACTIVE: 활성, INACTIVE: 비활성 (soft delete)';
```

### 2.3 user_roles

```sql
CREATE TABLE user_roles (
    id          BIGSERIAL       PRIMARY KEY,
    user_id     BIGINT          NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role        VARCHAR(30)     NOT NULL
                                CHECK (role IN ('DRAFTER', 'APPROVER', 'VIEWER', 'DEPT_MANAGER', 'ADMIN')),
    granted_at  TIMESTAMP       NOT NULL DEFAULT now(),
    granted_by  BIGINT          REFERENCES users(id) ON DELETE SET NULL,

    UNIQUE (user_id, role)
);

COMMENT ON TABLE user_roles IS '사용자-역할 매핑 (한 사용자가 복수 역할 보유 가능)';
COMMENT ON COLUMN user_roles.granted_by IS '역할을 부여한 관리자 ID';
```

### 2.4 approval_requests

```sql
CREATE TABLE approval_requests (
    id              BIGSERIAL       PRIMARY KEY,
    title           VARCHAR(200)    NOT NULL,
    content         TEXT            NOT NULL,
    category        VARCHAR(30)     NOT NULL
                                    CHECK (category IN ('LEAVE', 'EXPENSE', 'PURCHASE', 'OTHER')),
    status          VARCHAR(20)     NOT NULL DEFAULT 'DRAFT'
                                    CHECK (status IN ('DRAFT', 'PENDING', 'IN_PROGRESS',
                                                      'APPROVED', 'REJECTED', 'WITHDRAWN',
                                                      'ESCALATED', 'EXPIRED')),
    drafter_id      BIGINT          NOT NULL REFERENCES users(id),
    department_id   BIGINT          NOT NULL REFERENCES departments(id),
    due_date        DATE,
    submitted_at    TIMESTAMP,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT now(),
    version         BIGINT          NOT NULL DEFAULT 0  -- 낙관적 락용
);

COMMENT ON TABLE approval_requests IS '결재 문서 (Aggregate Root)';
COMMENT ON COLUMN approval_requests.status IS '결재 문서 현재 상태';
COMMENT ON COLUMN approval_requests.version IS '낙관적 락: 배치와 API 동시 처리 충돌 감지용';
COMMENT ON COLUMN approval_requests.due_date IS '결재 처리 기한 (NULL이면 기한 없음)';
```

### 2.5 approval_lines

```sql
CREATE TABLE approval_lines (
    id                  BIGSERIAL   PRIMARY KEY,
    approval_request_id BIGINT      NOT NULL UNIQUE REFERENCES approval_requests(id) ON DELETE CASCADE,
    created_at          TIMESTAMP   NOT NULL DEFAULT now()
);

COMMENT ON TABLE approval_lines IS '결재 문서의 결재선 (1문서 = 1결재선)';
```

### 2.6 approval_steps

```sql
CREATE TABLE approval_steps (
    id                  BIGSERIAL       PRIMARY KEY,
    approval_line_id    BIGINT          NOT NULL REFERENCES approval_lines(id) ON DELETE CASCADE,
    step_order          INT             NOT NULL,
    approver_id         BIGINT          NOT NULL REFERENCES users(id),
    role_label          VARCHAR(50),
    status              VARCHAR(20)     NOT NULL DEFAULT 'WAITING'
                                        CHECK (status IN ('WAITING', 'IN_PROGRESS',
                                                          'APPROVED', 'REJECTED', 'SKIPPED')),
    created_at          TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP       NOT NULL DEFAULT now(),

    UNIQUE (approval_line_id, step_order)
);

COMMENT ON TABLE approval_steps IS '결재선의 단일 결재 단계';
COMMENT ON COLUMN approval_steps.step_order IS '결재 순서 (1부터 시작, 낮을수록 먼저 처리)';
COMMENT ON COLUMN approval_steps.role_label IS '표시용 역할명 (예: 팀장, 부서장). 실제 권한 검증에는 미사용';
```

### 2.7 approval_histories

```sql
CREATE TABLE approval_histories (
    id              BIGSERIAL       PRIMARY KEY,
    approval_step_id BIGINT         NOT NULL REFERENCES approval_steps(id) ON DELETE CASCADE,
    approver_id     BIGINT          NOT NULL REFERENCES users(id),
    approver_name   VARCHAR(100)    NOT NULL,   -- 비정규화: 이력 보존용
    action          VARCHAR(20)     NOT NULL
                                    CHECK (action IN ('APPROVED', 'REJECTED', 'WITHDRAWN',
                                                      'ESCALATED', 'DELEGATED')),
    comment         TEXT,
    processed_at    TIMESTAMP       NOT NULL DEFAULT now()
);

COMMENT ON TABLE approval_histories IS '결재 단계별 처리 이력 (불변 데이터)';
COMMENT ON COLUMN approval_histories.approver_name IS '비정규화: 사용자 삭제/이름 변경 후에도 이력 보존';
COMMENT ON COLUMN approval_histories.comment IS '반려 시 필수, 승인 시 선택적';
```

### 2.8 audit_logs

```sql
CREATE TABLE audit_logs (
    id              BIGSERIAL       PRIMARY KEY,
    entity_type     VARCHAR(50)     NOT NULL,   -- 예: ApprovalRequest, User
    entity_id       BIGINT          NOT NULL,
    action          VARCHAR(50)     NOT NULL,   -- 예: SUBMIT, APPROVE, REJECT, ROLE_CHANGED
    actor_id        BIGINT,                     -- NULL이면 시스템 (배치) 처리
    actor_name      VARCHAR(100)    NOT NULL,   -- 비정규화: 행위자 이름 보존
    before_state    TEXT,                       -- JSON: 변경 전 상태 (nullable)
    after_state     TEXT            NOT NULL,   -- JSON: 변경 후 상태
    ip_address      VARCHAR(45),                -- IPv4/IPv6 지원
    user_agent      VARCHAR(500),
    occurred_at     TIMESTAMP       NOT NULL DEFAULT now()
);

COMMENT ON TABLE audit_logs IS '시스템 전체 감사 추적 로그 (불변 데이터, 삭제 금지)';
COMMENT ON COLUMN audit_logs.actor_id IS 'NULL이면 시스템(배치 Job) 처리';
COMMENT ON COLUMN audit_logs.actor_name IS '비정규화: 사용자 삭제 후에도 감사 이력 보존';
COMMENT ON COLUMN audit_logs.before_state IS 'JSON 직렬화된 변경 전 상태 값';
COMMENT ON COLUMN audit_logs.after_state IS 'JSON 직렬화된 변경 후 상태 값';
```

---

## 3. ERD 관계도 (텍스트)

```
departments
┌────────────────┐
│ id (PK)        │◄─────────────────────────────────────────┐
│ name           │                                           │
│ code (UNIQUE)  │◄──────────────────────┐                  │
│ parent_id (FK) │──►departments(id)     │                  │
│ manager_id (FK)│──►users(id)           │                  │
│ created_at     │                       │                  │
│ updated_at     │                       │                  │
└────────────────┘                       │                  │
                                         │                  │
users                                    │                  │
┌────────────────┐                       │                  │
│ id (PK)        │◄──────────────────────┘                  │
│ username (UNI) │                                           │
│ email (UNI)    │                                           │
│ password_hash  │                       ┌──────────────────┤
│ name           │                       │                  │
│ department_id  │──►departments(id)─────┘                  │
│ status         │                                           │
│ created_at     │                                           │
│ updated_at     │                                           │
└───────┬────────┘                                           │
        │ 1:N                                                │
        │                                                    │
user_roles                                                   │
┌────────────────┐                                           │
│ id (PK)        │                                           │
│ user_id (FK)   │──►users(id)                               │
│ role           │                                           │
│ granted_at     │                                           │
│ granted_by(FK) │──►users(id)                               │
└────────────────┘                                           │
                                                             │
approval_requests                                            │
┌────────────────────┐                                       │
│ id (PK)            │                                       │
│ title              │                                       │
│ content            │                                       │
│ category           │                                       │
│ status             │  ← 핵심 상태 필드                      │
│ drafter_id (FK)    │──►users(id)                           │
│ department_id (FK) │──►departments(id)─────────────────────┘
│ due_date           │
│ submitted_at       │
│ completed_at       │
│ version            │  ← 낙관적 락
│ created_at         │
│ updated_at         │
└──────────┬─────────┘
           │ 1:1
           │
approval_lines
┌────────────────────────┐
│ id (PK)                │
│ approval_request_id(FK)│──►approval_requests(id)
│ created_at             │
└──────────┬─────────────┘
           │ 1:N (ordered by step_order)
           │
approval_steps
┌────────────────────┐
│ id (PK)            │
│ approval_line_id   │──►approval_lines(id)
│ step_order         │  ← 결재 순서
│ approver_id (FK)   │──►users(id)
│ role_label         │  ← 표시용 역할명
│ status             │  ← 단계별 상태
│ created_at         │
│ updated_at         │
└──────────┬─────────┘
           │ 1:1
           │
approval_histories
┌────────────────────┐
│ id (PK)            │
│ approval_step_id   │──►approval_steps(id)
│ approver_id (FK)   │──►users(id)
│ approver_name      │  ← 비정규화
│ action             │
│ comment            │
│ processed_at       │
└────────────────────┘

audit_logs (독립 테이블 - 다양한 엔티티 참조)
┌────────────────────┐
│ id (PK)            │
│ entity_type        │  ← 'ApprovalRequest', 'User', etc.
│ entity_id          │  ← 해당 엔티티의 PK
│ action             │
│ actor_id           │  ← users(id) (FK 없음 - 유연성)
│ actor_name         │  ← 비정규화
│ before_state       │  ← JSON
│ after_state        │  ← JSON
│ ip_address         │
│ user_agent         │
│ occurred_at        │
└────────────────────┘
```

**설계 근거: audit_logs의 entity_id에 FK 제약 없음**

audit_logs의 entity_id는 외래 키 제약을 걸지 않는다. 이유:
1. 다양한 엔티티를 단일 테이블로 추적하므로 다형 연관관계 FK는 불가능
2. 엔티티가 삭제되어도 감사 로그는 보존되어야 함
3. entity_type + entity_id 인덱스로 조회 성능 확보

---

## 4. 인덱스 전략

### 4.1 approval_requests 인덱스

```sql
-- 상태별 조회 (배치 처리, 목록 필터링)
CREATE INDEX idx_ar_status ON approval_requests(status);

-- 기안자별 조회 (내 결재 문서 목록)
CREATE INDEX idx_ar_drafter_id ON approval_requests(drafter_id);

-- 부서별 조회 (부서 관리자 뷰)
CREATE INDEX idx_ar_department_id ON approval_requests(department_id);

-- 배치 처리용 복합 인덱스 (만료 대상 조회: status + submitted_at)
CREATE INDEX idx_ar_status_submitted_at ON approval_requests(status, submitted_at)
WHERE status IN ('PENDING', 'IN_PROGRESS', 'ESCALATED');
-- 부분 인덱스: 종료 상태(APPROVED, EXPIRED)는 배치 대상 아님

-- 제출일시 범위 조회 (통계, 감사)
CREATE INDEX idx_ar_submitted_at ON approval_requests(submitted_at);
```

### 4.2 approval_steps 인덱스

```sql
-- 내 결재 목록 조회 (approver_id + IN_PROGRESS 상태)
CREATE INDEX idx_as_approver_status ON approval_steps(approver_id, status)
WHERE status = 'IN_PROGRESS';
-- 부분 인덱스: 처리 완료된 단계(APPROVED, REJECTED)는 조회 빈도 낮음
```

### 4.3 audit_logs 인덱스

```sql
-- 특정 엔티티의 감사 로그 조회
CREATE INDEX idx_al_entity ON audit_logs(entity_type, entity_id);

-- 행위자별 감사 로그 조회
CREATE INDEX idx_al_actor_id ON audit_logs(actor_id);

-- 시간순 조회 (최신 로그 조회)
CREATE INDEX idx_al_occurred_at ON audit_logs(occurred_at DESC);
```

### 4.4 인덱스 설계 원칙

1. **부분 인덱스 활용**: 자주 조회하는 상태 범위만 인덱스 대상으로 지정해 인덱스 크기를 줄인다.
2. **배치 쿼리 최적화**: 배치 처리 쿼리는 별도 복합 인덱스를 사전에 설계한다.
3. **카디널리티 고려**: status 컬럼은 카디널리티가 낮아 단독 인덱스 효과가 제한적. 복합 인덱스로 보완.

---

## 5. 설계 결정 사항

### 5.1 상태값 저장 방식: VARCHAR + Enum 매핑

**결정**: `VARCHAR(20) + CHECK CONSTRAINT + JPA @Enumerated(EnumType.STRING)`

**근거**:
- DB에서 직접 조회 시 가독성 (예: 'PENDING'이 1보다 의미 명확)
- 운영 중 SQL로 디버깅 및 감사 시 즉시 파악 가능
- JPA 매핑이 단순하고 직관적
- INT 방식은 Enum 순서 변경 시 데이터 파괴 위험

**트레이드오프**:
- Enum 이름 변경 시 DB 마이그레이션 필요 → Flyway로 관리
- 저장 공간은 INT보다 소폭 증가 → 결재 문서 규모에서 무시 가능

### 5.2 감사 로그 비정규화 (actor_name, approver_name)

**결정**: 행위자 이름을 감사 로그와 처리 이력에 직접 저장

**근거**:
- 사용자 계정 삭제/이름 변경 후에도 이력의 정확성 보존
- 감사 로그의 불변성 원칙: 기록 시점의 데이터를 영구 보존
- users 테이블 조인 없이 감사 로그 자체만으로 완결된 정보 제공

**트레이드오프**:
- 이름 변경 시 과거 이력은 변경 전 이름 유지 (의도된 설계)
- 저장 용량 소폭 증가 → 감사 로그 특성상 허용 가능

### 5.3 낙관적 락 (version 컬럼)

**결정**: `approval_requests.version` 컬럼으로 낙관적 락 구현

**이유**: 배치 Job과 사용자 API가 동시에 같은 결재 문서를 수정하는 경우 (예: 배치가 만료 처리 중 결재자가 승인 시도) 충돌을 감지하고 재시도할 수 있도록 한다.

**JPA 설정**:
```java
@Entity
public class ApprovalRequest {
    @Version
    private Long version;
}
```

**비관적 락을 선택하지 않은 이유**: 결재 처리는 동시 충돌이 매우 드물다 (순차 결재 구조). 비관적 락은 Lock 보유 시간 동안 다른 쿼리를 블록하므로 성능 저하가 더 크다.

### 5.4 approval_lines 별도 테이블 이유

approval_request 테이블에 directly 결재선 정보를 포함하지 않고 별도 테이블을 둔 이유:
- 결재선은 여러 단계(approval_steps)의 컨테이너 역할
- 향후 결재선 템플릿 기능 추가 시 확장 용이
- 책임 분리: 결재 문서의 메타 정보와 결재 처리 프로세스를 분리

---

## 6. Flyway 마이그레이션 스크립트

### 6.1 마이그레이션 파일 구조

```
src/main/resources/db/migration/
├── V1__create_initial_schema.sql       # 전체 스키마 초기 생성
├── V2__create_indexes.sql              # 인덱스 생성
└── V3__insert_initial_data.sql         # 초기 데이터 (관리자 계정, 기본 부서)
```

### 6.2 V1__create_initial_schema.sql (핵심 부분)

```sql
-- 순서 중요: 참조되는 테이블 먼저 생성

CREATE TABLE departments (
    id          BIGSERIAL       PRIMARY KEY,
    name        VARCHAR(100)    NOT NULL,
    code        VARCHAR(50)     NOT NULL UNIQUE,
    parent_id   BIGINT,
    manager_id  BIGINT,
    created_at  TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP       NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              BIGSERIAL       PRIMARY KEY,
    username        VARCHAR(50)     NOT NULL UNIQUE,
    email           VARCHAR(200)    NOT NULL UNIQUE,
    password_hash   VARCHAR(255)    NOT NULL,
    name            VARCHAR(100)    NOT NULL,
    department_id   BIGINT          REFERENCES departments(id) ON DELETE SET NULL,
    status          VARCHAR(20)     NOT NULL DEFAULT 'ACTIVE'
                                    CHECK (status IN ('ACTIVE', 'INACTIVE')),
    created_at      TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at      TIMESTAMP       NOT NULL DEFAULT now()
);

-- departments의 manager_id FK는 users 생성 후 추가
ALTER TABLE departments
    ADD CONSTRAINT fk_dept_manager FOREIGN KEY (manager_id) REFERENCES users(id) ON DELETE SET NULL,
    ADD CONSTRAINT fk_dept_parent FOREIGN KEY (parent_id) REFERENCES departments(id) ON DELETE SET NULL;

CREATE TABLE user_roles (
    id          BIGSERIAL   PRIMARY KEY,
    user_id     BIGINT      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role        VARCHAR(30) NOT NULL CHECK (role IN ('DRAFTER', 'APPROVER', 'VIEWER', 'DEPT_MANAGER', 'ADMIN')),
    granted_at  TIMESTAMP   NOT NULL DEFAULT now(),
    granted_by  BIGINT      REFERENCES users(id) ON DELETE SET NULL,
    UNIQUE (user_id, role)
);

CREATE TABLE approval_requests (
    id              BIGSERIAL   PRIMARY KEY,
    title           VARCHAR(200) NOT NULL,
    content         TEXT        NOT NULL,
    category        VARCHAR(30) NOT NULL CHECK (category IN ('LEAVE', 'EXPENSE', 'PURCHASE', 'OTHER')),
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT'
                                CHECK (status IN ('DRAFT', 'PENDING', 'IN_PROGRESS',
                                                  'APPROVED', 'REJECTED', 'WITHDRAWN',
                                                  'ESCALATED', 'EXPIRED')),
    drafter_id      BIGINT      NOT NULL REFERENCES users(id),
    department_id   BIGINT      NOT NULL REFERENCES departments(id),
    due_date        DATE,
    submitted_at    TIMESTAMP,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP   NOT NULL DEFAULT now(),
    updated_at      TIMESTAMP   NOT NULL DEFAULT now(),
    version         BIGINT      NOT NULL DEFAULT 0
);

CREATE TABLE approval_lines (
    id                      BIGSERIAL   PRIMARY KEY,
    approval_request_id     BIGINT      NOT NULL UNIQUE REFERENCES approval_requests(id) ON DELETE CASCADE,
    created_at              TIMESTAMP   NOT NULL DEFAULT now()
);

CREATE TABLE approval_steps (
    id                  BIGSERIAL   PRIMARY KEY,
    approval_line_id    BIGINT      NOT NULL REFERENCES approval_lines(id) ON DELETE CASCADE,
    step_order          INT         NOT NULL,
    approver_id         BIGINT      NOT NULL REFERENCES users(id),
    role_label          VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'WAITING'
                                    CHECK (status IN ('WAITING', 'IN_PROGRESS', 'APPROVED', 'REJECTED', 'SKIPPED')),
    created_at          TIMESTAMP   NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP   NOT NULL DEFAULT now(),
    UNIQUE (approval_line_id, step_order)
);

CREATE TABLE approval_histories (
    id                  BIGSERIAL   PRIMARY KEY,
    approval_step_id    BIGINT      NOT NULL REFERENCES approval_steps(id) ON DELETE CASCADE,
    approver_id         BIGINT      NOT NULL REFERENCES users(id),
    approver_name       VARCHAR(100) NOT NULL,
    action              VARCHAR(20) NOT NULL CHECK (action IN ('APPROVED', 'REJECTED', 'WITHDRAWN', 'ESCALATED', 'DELEGATED')),
    comment             TEXT,
    processed_at        TIMESTAMP   NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
    id              BIGSERIAL   PRIMARY KEY,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       BIGINT      NOT NULL,
    action          VARCHAR(50) NOT NULL,
    actor_id        BIGINT,
    actor_name      VARCHAR(100) NOT NULL,
    before_state    TEXT,
    after_state     TEXT        NOT NULL,
    ip_address      VARCHAR(45),
    user_agent      VARCHAR(500),
    occurred_at     TIMESTAMP   NOT NULL DEFAULT now()
);
```

### 6.3 V2__create_indexes.sql

```sql
-- approval_requests
CREATE INDEX idx_ar_status ON approval_requests(status);
CREATE INDEX idx_ar_drafter_id ON approval_requests(drafter_id);
CREATE INDEX idx_ar_department_id ON approval_requests(department_id);
CREATE INDEX idx_ar_submitted_at ON approval_requests(submitted_at);
CREATE INDEX idx_ar_status_submitted_at ON approval_requests(status, submitted_at)
    WHERE status IN ('PENDING', 'IN_PROGRESS', 'ESCALATED');

-- approval_steps
CREATE INDEX idx_as_approver_status ON approval_steps(approver_id, status)
    WHERE status = 'IN_PROGRESS';

-- audit_logs
CREATE INDEX idx_al_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_al_actor_id ON audit_logs(actor_id);
CREATE INDEX idx_al_occurred_at ON audit_logs(occurred_at DESC);
```
