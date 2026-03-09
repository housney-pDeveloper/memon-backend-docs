# SYSTEM_SEQUENCE_DIAGRAMS.md

이 문서는 EF 시스템 간 주요 통신 시나리오를
시퀀스 다이어그램으로 정의한다.

표기:
- Client / Gateway / Application / System / MQ / DB

---

## 1. 로그인/보호 API 호출 흐름 (Client → Gateway → Application)

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant G as Gateway
    participant A as Application

    C->>G: /{clientType}/{version}/... (Authorization: Bearer JWT)
    G->>G: JWT 검증(서명/만료/issuer/audience)
    alt 인증 실패
        G-->>C: {code:110xx, message, data:null}
    else 인증 성공
        G->>G: 정책(버전/차단/Deprecated) 확인
        G->>A: /api/{version}/... + X-Gateway-Verified + X-Gateway-Secret + X-Request-Id
        A->>A: GatewayAuthFilter 헤더 검증
        alt 헤더 누락/비밀키 불일치
            A-->>G: HTTP 403 (fail)
            G-->>C: {code:..., message, data:null}
        else 정상
            A-->>G: {code:1, message:"OK", data:{...}}
            G-->>C: {code:1, message:"OK", data:{...}}
        end
    end
```

## 2. 공통코드 조회 (Client → Gateway → System(api profile))
```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant G as Gateway
    participant S as System(API)

    C->>G: /{clientType}/{version}/commonCode/...
    G->>G: JWT 검증 + 정책 확인
    G->>S: /api/{version}/commonCode/... + Gateway Headers
    S->>S: GatewayAuthFilter 검증
    S-->>G: {code:1, message:"OK", data:[...]}
    G-->>C: {code:1, message:"OK", data:[...]}
```

## 3. 알림 구독(SSE) + 이벤트 푸시 (Client → Gateway → System, MQ → System)

SSE는 장시간 연결이므로 “인증/세션 유지”는 Gateway 정책에 의해 통제된다.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant G as Gateway
    participant S as System(API)
    participant MQ as MQ
    participant SW as System(Worker-MQ)

    C->>G: SSE Subscribe (/.../notifications/subscribe)
    G->>G: JWT 검증 + 정책 확인
    G->>S: SSE 연결 전달 + Gateway Headers
    S-->>C: SSE Stream Opened

    MQ->>SW: Event Message(event_id, trace_id, payload)
    SW->>SW: 멱등 체크(event_id)
    alt 중복 이벤트
        SW-->>MQ: ack (skip)
    else 신규 이벤트
        SW->>SW: 처리(최대 3회 재시도 + backoff)
        SW->>S: Push 요청(내부 인터페이스/스토어)
        S-->>C: SSE Push(data)
        SW-->>MQ: ack
    end
```

## 4. Batch 실행 (System(worker-batch) → Application/DB)

```mermaid
sequenceDiagram
    autonumber
    participant SB as System(Worker-Batch)
    participant A as Application
    participant DB as Database

    SB->>SB: Scheduler Trigger
    SB->>A: (필요 시) Application API 호출 (내부 인증 정책 적용)
    A->>DB: 트랜잭션 처리
    DB-->>A: result
    A-->>SB: 결과 반환
    SB->>SB: 결과 로깅/리포팅
```

## 5. 오류 응답 규칙(공통)
```mermaid
sequenceDiagram
    autonumber
    participant X as Any Server
    participant F as Frontend

    X-->>F: {code != 1, message, data:null}
    F->>F: message 표시
```

END OF FILE