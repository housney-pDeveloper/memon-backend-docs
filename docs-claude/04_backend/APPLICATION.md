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
- ServiceResolver.resolve(version)으로 구현체를 선택한다.

```java
@RequestMapping("/api/{version}/auth")
```

### 1.3 Service 규칙

- VersionedService 상속 필수
- getVersion()은 String 반환 (ApiVersion.V1 등)
- 정책 변경 시에만 새 버전 생성
- 기존 Service 복사 후 v2 생성 금지

### 1.4 ServiceResolver 규칙

- Resolver는 도메인별로 생성한다.
- Spring 주입된 모든 구현체를 getVersion() 값으로 Map 관리
- resolve(version) 메서드로 구현체 선택
- 지원하지 않는 버전 → UNSUPPORTED_VERSION 예외

### 1.5 ApiVersion 상수

- common/share/version/ApiVersion.java에 정의
- V1 = "v1", V2 = "v2"

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
│   ├─ response
│   └─ share
│       ├─ service    (VersionedService)
│       └─ version    (ApiVersion)
│
├─ biz
│   ├─ vo
│   │   ├─ mmhr        → scm_mmhr
│   │   ├─ mmauth      → scm_mmauth
│   │   └─ mmpublic    → scm_mmpublic
│   │
│   └─ {domain}
│       ├─ controller
│       │   └─ {Domain}Controller
│       ├─ resolver
│       │   ├─ {Domain}Service (인터페이스)
│       │   └─ {Domain}ServiceResolver
│       └─ v1
│           ├─ dao
│           │   └─ {Domain}DaoV1
│           ├─ dto
│           │   └─ {Action}{Type}DtoV1 (@Alias 필수)
│           └─ service
│               └─ {Domain}ServiceV1
│
└─ Application.java
```

### 2.1 VO 위치 규칙 (필수)

모든 VO는 반드시 `biz/vo/{schema}` 에 생성한다.

### 2.2 Mapper XML 위치 규칙

```
resources/mapper/{domain}/v1/
├─ {Domain}MapperV1.xml      (SELECT)
└─ {Domain}MapperModV1.xml   (INSERT/UPDATE/DELETE)
```

- Dao:Mapper = 1:2 관계 필수
- namespace는 해당 DaoV1의 full path
- parameterType/resultType은 @Alias 이름 사용 (full path 금지)

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
