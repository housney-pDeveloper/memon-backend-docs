# 공통코드 Redis 캐시 정책

## 1. 개요

공통코드는 Redis 기반 분산 캐시를 통해 조회 성능을 최적화한다.
Outbox 패턴을 활용한 이벤트 기반 캐시 무효화를 적용하며,
Cache Stampede 방지를 위한 분산 락을 사용한다.

---

## 2. 아키텍처

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Gateway    │───▶│   System    │───▶│    Redis    │
│             │    │   (api)     │    │   Cluster   │
└─────────────┘    └──────┬──────┘    └─────────────┘
                          │ Cache Miss
                          ▼
                   ┌─────────────┐
                   │ Application │
                   │   Server    │
                   └──────┬──────┘
                          │ 변경 발생
                          ▼
                   ┌─────────────┐    ┌─────────────┐
                   │  Outbox     │───▶│  RabbitMQ   │
                   │  Table      │    │             │
                   └─────────────┘    └──────┬──────┘
                                             │
                                             ▼
                                      ┌─────────────┐
                                      │   System    │
                                      │ (worker-mq) │
                                      └──────┬──────┘
                                             │ 캐시 무효화
                                             ▼
                                      ┌─────────────┐
                                      │    Redis    │
                                      └─────────────┘
```

---

## 3. 캐시 키 설계

### 3.1 키 패턴

| 용도 | 키 패턴 | 예시 |
|------|---------|------|
| 공통코드 캐시 | `system:common-code:{clientNo}:{codeInit}` | `system:common-code:20001:USER_STATUS` |
| 분산 락 | `lock:common-code:{clientNo}:{codeInit}` | `lock:common-code:20001:USER_STATUS` |
| 처리 완료 이벤트 | `system:processed-event:{eventId}` | `system:processed-event:550e8400-...` |

### 3.2 키 생성 (RedisKey Enum)

```java
// ef-common/RedisKey.java
COMMON_CODE_CACHE("common:code:{groupCode}", "공통코드 캐시")
DISTRIBUTED_LOCK("lock:{lockName}", "분산 락")
```

### 3.3 System 서버 키 상수

```java
// CacheKeys.java
COMMON_CODE_PREFIX = "system:common-code"
COMMON_CODE_LOCK_PREFIX = "common-code"
PROCESSED_EVENT_PREFIX = "system:processed-event"
```

---

## 4. TTL 정책

| 리소스 | TTL | 근거 |
|--------|-----|------|
| 공통코드 캐시 | 24시간 | 이벤트 지연 시 안전망 |
| 분산 락 | 5초 | 락 누수 방지 |
| 처리 완료 이벤트 ID | 24시간 | 중복 처리 방지 (멱등성) |

```java
// RedisCommonCodeRepository.java
private static final Duration DEFAULT_TTL = Duration.ofHours(24);

// RedisLockService.java
private static final Duration LOCK_TTL = Duration.ofSeconds(5);
```

---

## 5. 캐시 데이터 구조

### 5.1 저장 형식 (JSON)

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
            "ext1": null
        },
        {
            "codeId": "I",
            "codeName": "비활성",
            "useYn": "Y",
            "dispYn": "Y",
            "sortOrder": 2
        }
    ],
    "cachedAt": "2026-02-27T00:00:00Z"
}
```

### 5.2 DTO 정의

```java
public class CachedCodeGroup {
    private Long clientNo;
    private String codeInit;
    private List<CodeItemDto> codes;
    private Instant cachedAt;
}

public class CodeItemDto {
    private String codeId;
    private String codeName;
    private String useYn;
    private String dispYn;
    private Integer sortOrder;
    private String codeNknm;
    private String codeDescp;
    private String ext1;
}
```

---

## 6. 조회 흐름

### 6.1 Cache Hit

```
Client → Gateway → System (api)
                       │
                       ▼
                    Redis MGET
                       │
                       ▼ (캐시 존재)
                    응답 반환
```

### 6.2 Cache Miss (단일 코드)

```
1. Redis MGET 조회 → 미스
2. 분산 락 획득 시도 (5초 TTL)
   ├─ 성공:
   │   ├─ Double-check: Redis 재조회
   │   ├─ 여전히 미스: Application API 호출
   │   ├─ Redis 저장 (24시간 TTL)
   │   └─ 락 해제
   └─ 실패:
       ├─ 100ms 대기 후 재시도 (최대 3회)
       └─ 3회 실패: Application 직접 호출 (캐시 저장 안 함)
3. 응답 반환
```

### 6.3 파라미터

```java
// CommonCodeRedisCacheService.java
private static final int MAX_CODE_INIT_COUNT = 20;          // 최대 조회 코드 수
private static final Duration LOCK_WAIT_TIME = Duration.ofMillis(300);
private static final Duration LOCK_RETRY_INTERVAL = Duration.ofMillis(100);
private static final int MAX_RETRY_COUNT = 3;
```

---

## 7. 캐시 무효화

### 7.1 Outbox 패턴

```
[Application 서버]
1. 공통코드 변경 트랜잭션 시작
2. tb_code 테이블 UPDATE/INSERT/DELETE
3. tb_outbox_event 테이블 INSERT (동일 트랜잭션)
4. 트랜잭션 커밋
5. Outbox Publisher가 이벤트 발행 (RabbitMQ)
6. published_yn = 'Y' 업데이트
```

### 7.2 이벤트 스키마

```java
public class CommonCodeChangedEvent {
    private String eventId;          // UUID
    private String eventType;        // "CommonCodeChanged"
    private Instant occurredAt;      // UTC
    private Long clientNo;
    private String codeInit;
    private ChangeType changeType;   // CREATE, UPDATE, DELETE
    private String traceId;
}
```

### 7.3 무효화 처리 (worker-mq)

```java
// CommonCodeInvalidationHandler.java
1. 멱등성 검사: processed-event:{eventId} 키 존재 확인
2. 캐시 삭제: DEL system:common-code:{clientNo}:{codeInit}
3. 처리 완료 기록: SET system:processed-event:{eventId} "1" EX 86400
4. ACK
```

---

## 8. 장애 대응

### 8.1 Circuit Breaker

```java
// ResilienceConfig.java

// Redis Circuit Breaker
CircuitBreakerConfig:
  SlidingWindowSize: 10
  FailureRateThreshold: 50%
  WaitDurationInOpenState: 30초
  PermittedCallsInHalfOpenState: 3

// Application Fallback Circuit Breaker
CircuitBreakerConfig:
  SlidingWindowSize: 10
  FailureRateThreshold: 50%
  WaitDurationInOpenState: 60초
```

### 8.2 Rate Limiter

```java
// Application API 호출 제한
RateLimiterConfig:
  LimitForPeriod: 100 req/sec
  LimitRefreshPeriod: 1초
  TimeoutDuration: 100ms
```

### 8.3 Fallback 전략

```
Redis 장애 시:
├─ Circuit Open → Application 직접 호출
├─ Application Circuit Open → 에러 메시지 반환
└─ "캐시 서비스를 일시적으로 사용할 수 없습니다."

락 획득 실패 시:
└─ 3회 재시도 후 Application 직접 호출 (캐시 미저장)

Application 타임아웃 시:
└─ 빈 리스트 반환, 에러 로깅
```

---

## 9. 캐시 워밍업

### 9.1 실행 시점

| 시점 | 트리거 |
|------|--------|
| 애플리케이션 시작 | ApplicationRunner |
| 매일 새벽 3시 | Cron: `0 0 3 * * *` |
| 수동 | K8s postStart Hook |

### 9.2 설정

```yaml
# application-worker-batch.yml
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

### 9.3 처리 흐름

```java
// CommonCodeWarmupJob.java
for (ClientConfig client : clients) {
    1. Application 내부 API 호출
    2. Redis 저장 (24시간 TTL)
    3. 로그 기록
}
// 개별 클라이언트 실패해도 전체 계속 진행
```

---

## 10. Profile별 역할

| Profile | 포트 | Redis 작업 |
|---------|------|-----------|
| api | 8082 | 읽기 (GET, MGET) |
| worker-mq | 8083 | 삭제 (DEL) |
| worker-batch | 8084 | 쓰기 (SET, MSET) |

---

## 11. 분산 락 구현

### 11.1 락 획득

```java
// RedisLockService.java
public String tryLock(String key) {
    String token = UUID.randomUUID().toString();
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent(key, token, LOCK_TTL);
    return Boolean.TRUE.equals(acquired) ? token : null;
}
```

### 11.2 락 해제 (Lua Script)

```lua
-- 소유권 검증 후 삭제 (원자적)
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

### 11.3 사용 예시

```java
String lockKey = "lock:common-code:" + clientNo + ":" + codeInit;
String token = lockService.tryLock(lockKey);

try {
    if (token != null) {
        // 임계 영역: DB 조회 후 캐시 저장
    }
} finally {
    if (token != null) {
        lockService.unlock(lockKey, token);
    }
}
```

---

## 12. 모니터링

### 12.1 메트릭

| 메트릭 | 설명 |
|--------|------|
| `cache.hit.rate` | 캐시 적중률 |
| `cache.miss.count` | 캐시 미스 횟수 |
| `lock.acquisition.time` | 락 획득 소요 시간 |
| `lock.timeout.count` | 락 타임아웃 횟수 |
| `application.fallback.count` | Application 폴백 횟수 |
| `circuit.state` | Circuit Breaker 상태 |

### 12.2 알림 임계치

| 조건 | 알림 |
|------|------|
| 캐시 적중률 < 80% | Warning |
| 락 타임아웃 > 10/min | Warning |
| Circuit Open | Critical |
| Redis 연결 실패 | Critical |

---

## 13. 금지 사항

- 캐시 키에 와일드카드(*) 사용 금지 (삭제 시 제외)
- TTL 없이 캐시 저장 금지
- 락 해제 없이 임계 영역 종료 금지 (finally 필수)
- Application 직접 호출 후 캐시 저장 금지 (락 미획득 시)
- 단일 요청에서 20개 초과 코드 그룹 조회 금지

---

## 14. 관련 파일

| 파일 | 설명 |
|------|------|
| `ef-common/.../RedisAutoConfiguration.java` | Redis 자동 설정 |
| `ef-common/.../RedisKey.java` | 타입 안전 키 생성 |
| `ef-common/.../RedisUtil.java` | Redis CRUD 유틸리티 |
| `system/.../RedisCommonCodeRepository.java` | 공통코드 저장소 |
| `system/.../RedisLockService.java` | 분산 락 서비스 |
| `system/.../CommonCodeRedisCacheService.java` | 캐시 조회 서비스 |
| `system/.../CommonCodeInvalidationHandler.java` | 캐시 무효화 핸들러 |
| `system/.../ResilienceConfig.java` | 장애 대응 설정 |
| `application/.../CommonCodeEventPublisher.java` | Outbox 이벤트 발행 |

---

END OF FILE
