# GATEWAY.md

Gateway 서버 고유 구현 가이드. 공통 컨벤션은 CODE_CONVENTION.md 참조.

---

## 1. 구현 원칙

### 1.1 Stateless 원칙

- 세션 저장 금지
- 상태 캐싱 금지
- 사용자 상태 유지 금지

### 1.2 Reactive 원칙

- Mono / Flux 기반 구현
- Blocking 코드 사용 금지
- Thread.sleep 금지
- RestTemplate 사용 금지

### 1.3 정책 기반 제어

- 모든 정책은 Filter 기반
- 코드 하드코딩 최소화
- 설정 기반 제어 우선

### 1.4 구조적 차단

Gateway 우회 요청은 구조적으로 차단되어야 한다.

---

## 2. 라우팅 정책

### 2.1 외부 URL 구조

- /app/v1/**
- /web/v2/**

### 2.2 내부 API Server URL

- /api/v1/**
- /api/v2/**

### 2.3 버전 독립 정책

- Web Client 버전과 API 버전은 독립 관리
- Web v2에서도 API v1 / v2 혼용 가능

### 2.4 RouteLocator 규칙

- 모든 Route는 ApiRouteLocator에서 관리
- 설정 기반 제어 우선
- 코드 변경 최소화

---

## 3. 디렉토리 구조

```
root
├─ core
│  ├─ config        (Route / Filter 설정)
│  ├─ security      (JWT 인증)
│  ├─ filter        (Global / Route Filter)
│  └─ policy        (Version / Deprecation / Block 정책)
│
├─ common
│  ├─ util
│  ├─ constant
│  └─ logging
│
├─ route
│  └─ ApiRouteLocator
│
└─ GatewayApplication.java
```

규칙:
- 모든 정책은 Filter 기반으로 구현
- Route 설정은 config 또는 route 영역에만 작성
- JWT 관련 코드는 security 영역에만 작성

---

## 4. 에러 코드 (1XXXX)

### 4.1 비즈니스 레벨 정의

| 범위 | 영역 |
|------|------|
| 11XXX | JWT 인증 |
| 12XXX | API 버전 정책 |
| 13XXX | 라우팅 |
| 19XXX | 서버 오류 |

### 4.2 예시

| 코드 | 설명 |
|------|------|
| 11001 | 토큰 없음 |
| 11003 | 토큰 만료 |
| 12001 | 버전 차단 |
| 19999 | 내부 오류 |

### 4.3 신규 코드 추가

1. 미사용 비즈니스1레벨 번호 할당
2. 본 문서에 정의 추가
3. ResponseCode.java에 코드 추가

---

END OF FILE
