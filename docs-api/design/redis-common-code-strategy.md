# Redis 기반 공통코드 캐시 전략 설계서

## 목차
1. [DDL 설계](#1-ddl-설계)
2. [System 공통코드 조회 API](#2-system-공통코드-조회-api)
3. [Application 내부용 공통코드 API](#3-application-내부용-공통코드-api)
4. [Redis 키/TTL/저장 포맷](#4-redis-키ttl저장-포맷)
5. [Cache Stampede 방지 전략](#5-cache-stampede-방지-전략)
6. [이벤트 스키마 + Outbox 패턴](#6-이벤트-스키마--outbox-패턴)
7. [Profile별 컴포넌트 구성](#7-profile별-컴포넌트-구성)
8. [장애 시나리오별 동작](#8-장애-시나리오별-동작)

---

## 1. DDL 설계

### 1.1 tb_code_mngmn (코드 그룹 관리)

```sql
CREATE TABLE scm_efpublic.tb_code_mngmn (
    client_no           BIGINT       NOT NULL,
    code_init           VARCHAR(50)  NOT NULL,
    code_init_name      VARCHAR(100) NOT NULL,
    code_init_descp     VARCHAR(500),
    use_yn              CHAR(1)      NOT NULL DEFAULT 'Y',
    disp_yn             CHAR(1)      NOT NULL DEFAULT 'Y',
    sort_order          INT          NOT NULL DEFAULT 0,
    reg_date            TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_user_no         BIGINT       NOT NULL DEFAULT 100000,
    mod_date            TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    mod_user_no         BIGINT       NOT NULL DEFAULT 100000,
    CONSTRAINT tb_code_mngmn_pk PRIMARY KEY (client_no, code_init)
);

CREATE INDEX tb_code_mngmn_idx01 ON scm_efpublic.tb_code_mngmn (client_no, use_yn);
```

**설계 근거:**
- PK가 `(client_no, code_init)`인 이유: clientNo 단위 멀티테넌시 지원
- 인덱스 idx01: clientNo 기준 전체 그룹 조회 최적화

### 1.2 tb_code (코드 상세)

```sql
CREATE TABLE scm_efpublic.tb_code (
    client_no           BIGINT       NOT NULL,
    code_init           VARCHAR(50)  NOT NULL,
    code_id             VARCHAR(50)  NOT NULL,
    code_name           VARCHAR(100) NOT NULL,
    code_descp          VARCHAR(500),
    code_nknm           VARCHAR(100),
    use_yn              CHAR(1)      NOT NULL DEFAULT 'Y',
    disp_yn             CHAR(1)      NOT NULL DEFAULT 'Y',
    sort_order          INT          NOT NULL DEFAULT 0,
    ext1~ext5           VARCHAR(200),
    reg_date            TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_user_no         BIGINT       NOT NULL DEFAULT 100000,
    mod_date            TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    mod_user_no         BIGINT       NOT NULL DEFAULT 100000,
    CONSTRAINT tb_code_pk PRIMARY KEY (client_no, code_init, code_id),
    CONSTRAINT tb_code_fk01 FOREIGN KEY (client_no, code_init)
        REFERENCES scm_efpublic.tb_code_mngmn(client_no, code_init)
);

CREATE INDEX tb_code_idx01 ON scm_efpublic.tb_code (client_no, code_init, use_yn, sort_order);
CREATE INDEX tb_code_idx02 ON scm_efpublic.tb_code (client_no, use_yn);
```

**설계 근거:**
- PK `(client_no, code_init, code_id)`: 가장 빈번한 조회 패턴과 일치
- idx01: 그룹 내 코드 목록 조회 시 use_yn 필터 + sort_order 정렬 최적화
- ext1~ext5: 확장 필드로 유연성 확보

### 1.3 tb_outbox_event (Outbox 이벤트)

```sql
CREATE TABLE scm_efpublic.tb_outbox_event (
    event_id            VARCHAR(36)  NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    aggregate_type      VARCHAR(100) NOT NULL,
    aggregate_id        VARCHAR(200) NOT NULL,
    payload             TEXT         NOT NULL,
    occurred_at         TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at        TIMESTAMP,
    published_yn        CHAR(1)      NOT NULL DEFAULT 'N',
    retry_count         INT          NOT NULL DEFAULT 0,
    last_error          VARCHAR(1000),
    trace_id            VARCHAR(50),
    CONSTRAINT tb_outbox_event_pk PRIMARY KEY (event_id)
);

CREATE INDEX tb_outbox_event_idx01 ON scm_efpublic.tb_outbox_event (published_yn, occurred_at)
    WHERE published_yn = 'N';
```

**설계 근거:**
- Partial Index (published_yn = 'N'): 미발행 이벤트만 빠르게 조회
- event_id UUID: 멱등성 보장을 위한 전역 유니크 식별자

---

## 2. System 공통코드 조회 API

### 2.1 엔드포인트

```
POST /api/{version}/code/query
```

### 2.2 Request Headers

| 헤더 | 필수 | 설명 |
|------|------|------|
| Authorization | O | Bearer {JWT 토큰} |
| X-Client-No | O | 고객사 번호 (Gateway에서 JWT claim 추출) |
| Content-Type | O | application/json |

### 2.3 Request Body

```json
{
    "clientNo": 20001,
    "codeInitList": ["USER_STATUS", "ROLE_TYPE", "GENDER"]
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| clientNo | Long | O | 고객사 번호 |
| codeInitList | List<String> | O | 조회할 코드 그룹 목록 (최대 20개) |

### 2.4 Response Body

```json
{
    "code": 1,
    "message": "OK",
    "data": {
        "clientNo": 20001,
        "items": {
            "USER_STATUS": [
                {
                    "codeId": "A",
                    "codeName": "활성",
                    "useYn": "Y",
                    "dispYn": "Y",
                    "sortOrder": 1,
                    "codeNknm": null,
                    "codeDescp": null,
                    "ext1": null,
                    "ext2": null,
                    "ext3": null,
                    "ext4": null,
                    "ext5": null
                },
                {
                    "codeId": "I",
                    "codeName": "비활성",
                    "useYn": "Y",
                    "dispYn": "Y",
                    "sortOrder": 2
                }
            ],
            "ROLE_TYPE": [
                {
                    "codeId": "ADMIN",
                    "codeName": "관리자",
                    "useYn": "Y",
                    "dispYn": "Y",
                    "sortOrder": 1
                }
            ],
            "GENDER": []
        },
        "cached": true,
        "lastUpdatedAt": "2026-01-18T00:00:00Z"
    }
}
```

### 2.5 에러 응답

| 에러명 | 메시지 | 발생 상황 |
|--------|--------|-----------|
| INVALID_CLIENT_NO | 유효하지 않은 고객사 번호입니다. | clientNo가 null이거나 0 이하 |
| EMPTY_CODE_INIT_LIST | 조회할 코드 그룹이 없습니다. | codeInitList가 비어있음 |
| TOO_MANY_CODE_INIT | 한 번에 최대 20개 그룹까지 조회 가능합니다. | codeInitList.size() > 20 |
| CACHE_UNAVAILABLE | 캐시 서비스를 일시적으로 사용할 수 없습니다. | Redis + Application 모두 실패 |

### 2.6 cached 필드 규칙

- `true`: 전체 그룹이 Redis에서 hit
- `false`: 1개 이상의 그룹이 miss되어 Application으로 조회 후 캐시에 저장됨
- 부분 hit/miss도 `false`로 표시 (miss 그룹 존재 시)

---

## 3. Application 내부용 공통코드 API

### 3.1 엔드포인트

```
POST /internal/api/{version}/code/query
```

**보안 설정:**
- Gateway 라우팅에서 `/internal/**` 경로 차단 (외부 노출 금지)
- `X-Gateway-Secret` 헤더 검증 필수

### 3.2 Request Body

```json
{
    "clientNo": 20001,
    "codeInitList": ["USER_STATUS", "ROLE_TYPE"]
}
```

### 3.3 Response Body

```json
{
    "code": 1,
    "message": "OK",
    "data": {
        "clientNo": 20001,
        "items": {
            "USER_STATUS": [
                {
                    "codeId": "A",
                    "codeName": "활성",
                    "useYn": "Y",
                    "dispYn": "Y",
                    "sortOrder": 1
                }
            ]
        },
        "fromDb": true,
        "queriedAt": "2026-01-18T00:00:00Z"
    }
}
```

### 3.4 에러 응답

| 에러명 | 메시지 |
|--------|--------|
| INTERNAL_ACCESS_DENIED | 내부 API 접근이 거부되었습니다. |
| COMMON_CODE_NOT_FOUND | 요청한 공통코드 그룹이 존재하지 않습니다. |

---

## 4. Redis 키/TTL/저장 포맷

### 4.1 캐시 키 규칙

```
system:common-code:{clientNo}:{code_init}
```

**예시:**
```
system:common-code:20001:USER_STATUS
system:common-code:20001:ROLE_TYPE
system:common-code:20001:GENDER
```

**설계 근거:**
- prefix `system:`: System 서버 전용 네임스페이스
- 그룹(code_init) 단위 캐싱: 변경 시 해당 그룹만 무효화 (최소 영향 범위)

### 4.2 TTL

```
TTL = 86400초 (24시간)
```

**설계 근거:**
- TTL은 MQ 장애/무효화 지연 시 "최후의 안전망"
- 정상 반영은 이벤트 기반 무효화로 즉시 처리
- 24시간: 야간 배치/점검 주기 고려

### 4.3 저장 포맷 (JSON)

```json
{
    "clientNo": 20001,
    "codeInit": "USER_STATUS",
    "codes": [
        {
            "codeId": "A",
            "codeName": "활성",
            "useYn": "Y",
            "dispYn": "Y",
            "sortOrder": 1,
            "codeNknm": null,
            "codeDescp": null,
            "ext1": null,
            "ext2": null,
            "ext3": null,
            "ext4": null,
            "ext5": null
        }
    ],
    "cachedAt": "2026-01-18T00:00:00Z"
}
```

### 4.4 Redis 명령 예시

**저장:**
```
SET system:common-code:20001:USER_STATUS '{"clientNo":20001,...}' EX 86400
```

**조회 (Pipeline):**
```
MGET system:common-code:20001:USER_STATUS system:common-code:20001:ROLE_TYPE
```

**삭제 (무효화):**
```
DEL system:common-code:20001:USER_STATUS
```

---

## 5. Cache Stampede 방지 전략

### 5.1 선택한 방식: Redis 분산락 + 더블체크

**이유:**
- Single-Flight 대비 구현 단순
- 분산 환경(Scale-out)에서 일관성 보장
- Redis 장애 시 락 해제 자동화 가능 (TTL 기반)

### 5.2 상세 동작

```
1. Redis MGET으로 요청된 code_init 목록 조회
2. hit된 그룹은 즉시 응답 후보에 추가
3. miss된 그룹 각각에 대해:
   3.1. 락 획득 시도: SET lock:common-code:{clientNo}:{code_init} {uuid} NX EX 5
   3.2. 락 획득 성공:
        - Application 내부 API 호출
        - Redis에 결과 저장 (SET ... EX 86400)
        - 락 해제: DEL lock:... (Lua Script로 본인 확인 후 삭제)
   3.3. 락 획득 실패:
        - 100ms 대기
        - Redis 재조회 (더블체크)
        - 여전히 miss면 최대 3회 재시도
        - 최종 실패 시 Application 직접 호출 (캐시 저장 생략)
4. 모든 그룹 데이터 취합 후 응답
```

### 5.3 락 키 규칙

```
lock:common-code:{clientNo}:{code_init}
```

**락 TTL:** 5초 (타임아웃 + 여유)

### 5.4 락 해제 Lua Script

```lua
-- 본인이 획득한 락만 해제 (UUID 확인)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**설계 근거:**
- 타임아웃으로 인해 다른 요청이 락을 획득한 경우, 원래 요청자가 실수로 해제하지 않도록 방지
- 락 누수 방지: TTL 5초로 자동 해제

### 5.5 안전성 보장

| 시나리오 | 대응 |
|----------|------|
| 락 획득 요청자 타임아웃 | TTL 5초로 자동 해제 |
| Application 호출 실패 | 락 해제 후 다음 요청에서 재시도 |
| Redis 장애 | Circuit Breaker 발동, Application fallback |

---

## 6. 이벤트 스키마 + Outbox 패턴

### 6.1 이벤트 스키마 (CommonCodeChanged)

```json
{
    "event_id": "01HXYZ123456789ABCDEFGHIJK",
    "event_type": "CommonCodeChanged",
    "occurred_at": "2026-01-18T06:12:00Z",
    "client_no": 20001,
    "code_init": "USER_STATUS",
    "change_type": "UPDATE",
    "trace_id": "trace-abc-123",
    "payload": {
        "before": null,
        "after": {
            "codeId": "A",
            "codeName": "활성화"
        }
    }
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| event_id | String | O | UUID/ULID (멱등성 키) |
| event_type | String | O | 이벤트 타입 |
| occurred_at | String | O | 발생 시각 (ISO 8601 UTC) |
| client_no | Long | O | 고객사 번호 |
| code_init | String | O | 변경된 코드 그룹 |
| change_type | String | O | CREATE/UPDATE/DELETE |
| trace_id | String | - | 분산 추적 ID |
| payload | Object | - | 변경 상세 정보 |

### 6.2 Outbox 패턴 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application                              │
│                                                                  │
│  1. 비즈니스 로직 실행 (공통코드 CUD)                             │
│  2. tb_outbox_event에 이벤트 INSERT                              │
│  3. DB 트랜잭션 COMMIT                                           │
│                                                                  │
│  4. Outbox Publisher (별도 스레드/스케줄러)                       │
│     - SELECT * FROM tb_outbox_event WHERE published_yn = 'N'     │
│     - MQ에 이벤트 발행                                           │
│     - UPDATE tb_outbox_event SET published_yn = 'Y'              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ MQ (RabbitMQ/Kafka)
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    System (worker-mq)                            │
│                                                                  │
│  5. MQ 리스너가 이벤트 수신                                      │
│  6. 멱등성 체크 (event_id 중복 확인)                             │
│  7. Redis 캐시 삭제: DEL system:common-code:{clientNo}:{codeInit}│
│  8. ACK 전송                                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Outbox Publisher 구현

```java
@Scheduled(fixedDelay = 100) // 100ms 간격
public void publishPendingEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished(100);

    for (OutboxEvent event : events) {
        try {
            mqPublisher.publish(event.getEventType(), event.getPayload());
            outboxRepository.markAsPublished(event.getEventId());
        } catch (Exception e) {
            outboxRepository.incrementRetryCount(event.getEventId(), e.getMessage());
            if (event.getRetryCount() >= 3) {
                // DLQ 또는 알람 처리
            }
        }
    }
}
```

### 6.4 멱등성 보장 (System)

```java
public void handleCommonCodeChanged(CommonCodeChangedEvent event) {
    String eventId = event.getEventId();

    // 이미 처리된 이벤트면 스킵
    if (processedEventStore.exists(eventId)) {
        log.debug("Duplicate event skipped: {}", eventId);
        return;
    }

    // Redis 캐시 삭제
    String key = String.format("system:common-code:%d:%s",
            event.getClientNo(), event.getCodeInit());
    redisTemplate.delete(key);

    // 처리 완료 기록 (TTL 24시간)
    processedEventStore.markProcessed(eventId, Duration.ofDays(1));
}
```

### 6.5 설계 안전성

| 위험 | 대응 |
|------|------|
| MQ 발행 전 Application 크래시 | Outbox 테이블에 이벤트 저장됨, 재시작 후 Publisher가 재처리 |
| MQ 발행 후 ACK 전 크래시 | System에서 중복 수신, 멱등성 체크로 무시 |
| System 처리 실패 | NACK 후 MQ 재전송, 최대 3회 재시도 후 DLQ |

---

## 7. Profile별 컴포넌트 구성

### 7.1 프로파일 정의

| Profile | 포트 | 역할 |
|---------|------|------|
| api | 8082 | 공통코드 조회 API 제공 |
| worker-mq | 8083 | MQ 리스너, 캐시 무효화 |
| worker-batch | 8084 | Warm-up 배치 |

### 7.2 패키지 트리

```
kr.fingate.ef.system
├── SystemApplication.java
│
├── api/                           # [profile=api]
│   └── commoncode/
│       ├── controller/
│       │   └── CommonCodeController.java
│       ├── dto/
│       │   ├── CommonCodeQueryRequest.java
│       │   ├── CommonCodeQueryResponse.java
│       │   └── CodeItemDto.java
│       └── service/
│           ├── CommonCodeQueryService.java
│           └── CommonCodeCacheService.java
│
├── mq/                            # [profile=worker-mq]
│   ├── listener/
│   │   └── CommonCodeEventListener.java
│   ├── handler/
│   │   └── CommonCodeInvalidationHandler.java
│   └── config/
│       └── MqConsumerConfig.java
│
├── batch/                         # [profile=worker-batch]
│   ├── job/
│   │   └── CommonCodeWarmupJob.java
│   ├── config/
│   │   └── WarmupConfig.java
│   └── scheduler/
│       └── WarmupScheduler.java
│
├── client/                        # [공통]
│   └── application/
│       ├── ApplicationApiClient.java
│       └── dto/
│           └── CommonCodeFromAppResponse.java
│
├── common/                        # [공통]
│   ├── redis/
│   │   ├── RedisCommonCodeRepository.java
│   │   ├── RedisLockService.java
│   │   └── config/
│   │       └── RedisConfig.java
│   ├── resilience/
│   │   ├── CircuitBreakerConfig.java
│   │   └── RateLimiterConfig.java
│   └── constant/
│       └── CacheKeys.java
│
└── config/
    └── profile/
        ├── ApiProfileConfig.java
        ├── MqWorkerProfileConfig.java
        └── BatchWorkerProfileConfig.java
```

### 7.3 Profile별 활성화 빈

**ApiProfileConfig.java:**
```java
@Configuration
@Profile("api")
@ComponentScan(basePackages = "kr.fingate.ef.system.api")
public class ApiProfileConfig {
    // API 관련 빈만 활성화
}
```

**MqWorkerProfileConfig.java:**
```java
@Configuration
@Profile("worker-mq")
@ComponentScan(basePackages = "kr.fingate.ef.system.mq")
public class MqWorkerProfileConfig {
    // MQ 리스너 관련 빈만 활성화
}
```

**BatchWorkerProfileConfig.java:**
```java
@Configuration
@Profile("worker-batch")
@EnableBatchProcessing
@ComponentScan(basePackages = "kr.fingate.ef.system.batch")
public class BatchWorkerProfileConfig {
    // Batch 관련 빈만 활성화
}
```

### 7.4 실행 명령

```bash
# API 서버
java -jar system.jar --spring.profiles.active=api,dev

# MQ Worker
java -jar system.jar --spring.profiles.active=worker-mq,dev

# Batch Worker
java -jar system.jar --spring.profiles.active=worker-batch,dev
```

---

## 8. 장애 시나리오별 동작

### 8.1 Redis 장애

**시나리오:** Redis 연결 실패 또는 타임아웃

**대응:**
```
1. Redis 타임아웃: 500ms (짧게 설정)
2. Circuit Breaker 발동 조건:
   - 최근 10회 요청 중 50% 이상 실패
   - 열림 상태 지속 시간: 30초
3. Circuit Breaker 열림 시:
   - Redis 조회 스킵
   - Application fallback 호출 (Circuit Breaker 별도 적용)
4. Application fallback 조건:
   - 타임아웃: 3초
   - Rate Limit: 초당 100회 (Application 보호)
5. 모두 실패 시:
   - "캐시 서비스를 일시적으로 사용할 수 없습니다." 에러 반환
```

**설계 안전성:**
- Redis 장애가 Application으로 전파되지 않도록 Rate Limit 적용
- Circuit Breaker로 빠른 실패 (Fast Fail) 구현

### 8.2 MQ 장애

**시나리오:** MQ 연결 실패 또는 메시지 유실

**대응:**
```
1. Outbox 테이블에 이벤트가 저장되어 있으므로 MQ 복구 후 재발행
2. MQ 장애 시 캐시 무효화 지연 발생 (일시적 stale data)
3. TTL 24시간이 최후의 안전망
4. 긴급 시 수동 캐시 갱신 API 제공:
   POST /api/{version}/common-code/cache/refresh
```

**모니터링:**
- tb_outbox_event의 published_yn = 'N' 건수 알람
- retry_count >= 3 건수 알람

### 8.3 System 서버 재기동

**시나리오:** System 서버 배포 또는 크래시

**대응:**
```
1. Redis 캐시는 유지됨 → 즉시 서비스 가능
2. 배포 직후 Warm-up 실행 (선택):
   - Kubernetes: initContainer 또는 post-start hook
   - 단독: startup 시 ApplicationRunner에서 실행
3. MQ Worker 재기동 시:
   - 미처리 메시지 재수신 (ACK 전 메시지)
   - 멱등성 체크로 중복 처리 방지
```

### 8.4 System 장애 시 우회 라우팅

**시나리오:** System 서버 전체 다운

**대응:**
```
1. Gateway 라우팅 정책 변경:
   - /api/{version}/common-code/** → Application 내부 API로 임시 라우팅
   - 단, 외부 노출 금지 (Gateway 내부 라우팅으로만 처리)

2. 우회 조건:
   - System Health Check 실패 3회 연속
   - 자동 전환 또는 수동 전환

3. 복구 후:
   - System Health Check 성공 확인
   - 라우팅 원복
   - Warm-up 실행
```

**Gateway 설정 예시 (Spring Cloud Gateway):**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: common-code-system
          uri: lb://system-server
          predicates:
            - Path=/api/**/common-code/**
          filters:
            - name: CircuitBreaker
              args:
                name: systemCircuitBreaker
                fallbackUri: forward:/fallback/common-code
```

### 8.5 장애 대응 요약표

| 장애 유형 | 감지 방법 | 자동 대응 | 수동 대응 |
|-----------|-----------|-----------|-----------|
| Redis 장애 | 연결 타임아웃/실패 | Circuit Breaker → Application fallback | Redis 복구 후 Warm-up |
| MQ 장애 | 발행 실패 | Outbox 재시도, TTL 안전망 | DLQ 모니터링, 수동 재발행 |
| System API 장애 | Health Check 실패 | Gateway 우회 라우팅 | 서버 재시작, 로그 분석 |
| System MQ Worker 장애 | Consumer 미응답 | 메시지 큐 누적 | Worker 재시작 |
| Application 장애 | 연결 실패 | 캐시된 데이터 제공 (stale) | Application 복구 |

---

## 부록: Warm-up 전략

### 필수 code_init 목록 관리

**설정 파일 방식 (application.yml):**
```yaml
warmup:
  common-code:
    enabled: true
    clients:
      - clientNo: 20001
        codeInitList:
          - USER_STATUS
          - ROLE_TYPE
          - GENDER
      - clientNo: 20002
        codeInitList:
          - USER_STATUS
```

### Warm-up 실행 타이밍

1. **배포 직후:** ApplicationRunner에서 1회 실행
2. **Redis 재기동 감지 후:** Redis Pub/Sub로 감지 시 실행
3. **정기 실행:** 매일 새벽 3시 (cron)

### Warm-up Job 구현

```java
@Slf4j
@Configuration
@Profile("worker-batch")
public class CommonCodeWarmupJob {

    @Scheduled(cron = "0 0 3 * * *") // 매일 새벽 3시
    public void scheduledWarmup() {
        executeWarmup();
    }

    public void executeWarmup() {
        warmupConfig.getClients().forEach(client -> {
            try {
                var response = applicationApiClient.queryCommonCodes(
                    client.getClientNo(),
                    client.getCodeInitList()
                );
                redisCommonCodeRepository.saveAll(client.getClientNo(), response.getItems());
                log.info("Warmup completed for clientNo={}", client.getClientNo());
            } catch (Exception e) {
                log.error("Warmup failed for clientNo={}", client.getClientNo(), e);
            }
        });
    }
}
```
