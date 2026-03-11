# APPLICATION.md

Application 서버 고유 구현 가이드. 공통 컨벤션은 CODE_CONVENTION.md 참조.

---

## 1. 버저닝

### 1.1 방식

URL 기반 버저닝만 허용한다.

- /api/v1/**
- /api/v2/**

### 1.2 Controller 규칙

- Controller는 버전별로 분리하지 않는다.
- 단일 Controller에서 {version} PathVariable을 사용한다.
- ServiceFactory에 version을 전달한다.

```java
@RequestMapping("/api/{version}/client")
```

### 1.3 Service 규칙

- VersionedService 상속 필수
- getVersion() 구현 필수
- 정책 변경 시에만 새 버전 생성
- 기존 Service 복사 후 v2 생성 금지

### 1.4 ServiceFactory 규칙

- 모든 구현체 자동 주입
- getVersion() 값으로 구현체 선택
- 지원하지 않는 버전 → UNSUPPORTED_VERSION 예외

---

## 2. 디렉토리 구조

```
root
├─ core
│   ├─ config
│   ├─ security
│   └─ filter
│
├─ common
│   ├─ dto
│   ├─ util
│   ├─ exception
│   └─ response
│
├─ biz
│   ├─ vo
│   │   ├─ mmhr        → scm_mmhr
│   │   ├─ mmauth      → scm_mmauth
│   │   └─ mmpublic    → scm_mmpublic
│   ├─ auth
│   ├─ calendar
│   └─ event
│
└─ Application.java
```

### 2.1 VO 위치 규칙 (필수)

모든 VO는 반드시 `biz/vo/{schema}` 에 생성한다.

---

## 3. 에러 코드 (3XXXX)

### 3.1 역할 분류

| 범위 | 영역 |
|------|------|
| 31XXX | 계정/인증 |
| 32XXX | 보안 |
| 33XXX | 데이터 |

### 3.2 공통 코드

9XXXX → hs-common CommonErrorCode 사용

### 3.3 신규 코드 추가

1. Application 전용 → ResponseCode.java 추가
2. 공통 코드 → hs-common CommonErrorCode.java 추가
3. 역할 확장 시 본 문서 업데이트

---

END OF FILE
