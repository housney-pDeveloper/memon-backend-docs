# SECURITY.md

전 모듈 보안 정책을 통합 정의한다.

---

## 1. 인증 아키텍처 (Gateway)

Gateway는 OAuth2 Client이자 내부 JWT 발급자이다.

### 1.1 역할

- **OAuth2 Client**: Google/Kakao OAuth2 로그인 처리
- **내부 JWT 발급**: OAuth2 인증 성공 후 내부 Access Token 발급
- **Refresh Token 관리**: Redis 기반 Stateful Refresh Token 관리
- **JWT 검증**: 이후 API 요청에서 내부 JWT 서명/issuer/exp 검증

### 1.2 Application/System과의 관계

- Application/System은 **내부 JWT 검증 없이** Gateway 헤더만 신뢰
- Application/System은 JWT 생성/파싱을 하지 않는다
- Gateway가 OAuth2 성공 후 Application 내부 API(/internal/user/syncOAuthUser) 호출

---

## 2. OAuth2 로그인 흐름

```
Client → OAuth2 Provider 로그인 → authorization code 획득
    ↓
Gateway ← callback (code)
    ↓ Provider token endpoint 호출 → id_token 수신
    ↓ sub, email 추출
    ↓ Application 내부 API 호출 (/internal/user/syncOAuthUser)
    ↓   → Application이 사용자 조회/생성 후 userNo 반환
    ↓ 내부 JWT(access token) 발급 (subject=userNo)
    ↓ Refresh Token 생성 → Redis 저장
    ↓ Client에 access token + refresh token 반환
```

---

## 3. 내부 JWT 정책

| 항목 | 값 |
|------|------|
| Access Token | Stateless JWT, 15분 TTL |
| Refresh Token | Redis Stateful, 14일 TTL, 로테이션 적용 |
| JWT Issuer | memon-gateway |
| JWT Claims | subject=userNo, email, role |

---

## 4. 내부 신뢰 헤더 정책

JWT 인증 성공 시 다음 헤더를 추가한다.

- X-Gateway-Verified: true
- X-Gateway-Secret: [공유 비밀키]
- X-Request-Id: UUID
- X-User-No: [사용자번호]
- X-User-Email: [이메일]
- X-User-Role: [역할]

인증 실패 시:
- 헤더 추가 금지
- 즉시 차단

---

## 5. Refresh Token Redis 구조

```
키: refresh:{tokenId}
값: { userNo, issuedAt, expiresAt }
TTL: 14일

인덱스 키: user:{userNo}:devices → Set<tokenId>
```

---

## 6. 공유 비밀키 (Internal Secret)

```yaml
gateway:
  routing:
    internal-secret: ${GATEWAY_INTERNAL_SECRET}
```

- Request Header로만 전송
- Response Header에 노출 금지
- CORS exposedHeaders에 포함 금지
- AWS Parameter Store (KMS 암호화) 저장
- 로그에 비밀키 출력 금지
- Gateway와 Application/System 동일 비밀키 사용

---

## 7. Application 보안 정책

### 7.1 Gateway 인증 헤더 검증

필수 헤더:
- X-Gateway-Verified
- X-Gateway-Secret

검증 실패 시: HTTP 403 반환
구현 위치: GatewayAuthFilter

### 7.2 Internal API

- /internal/** 경로는 JwtAuthFilter 우회
- GatewayAuthFilter로만 보호

---

## 8. System 보안 정책

- JWT 인증은 Gateway 책임
- System에서 JWT 로직 작성 금지
- Gateway 인증 헤더(X-Gateway-Verified, X-Gateway-Secret) 검증 필수
- 검증 위치: GatewayAuthFilter

---

## 9. 개인정보 암호화 (AES-256-GCM)

### 9.1 대상 데이터

- 주민등록번호
- 개인 이메일
- 휴대전화번호
- 기타 개인식별정보

### 9.2 암호화 흐름

```
[평문] → AES-256-GCM 암호화 → [IV + 암호문 + AuthTag] → Base64 → [DB 저장]
[DB 조회] → Base64 디코딩 → IV 분리 → AES-256-GCM 복호화 → [평문]
```

### 9.3 구현 클래스

| 클래스 | 용도 | 사용처 |
|--------|------|--------|
| `CryptoUtil` | 서비스 레이어 암복호화 | Service, Controller |
| `EncryptionUtil` | MyBatis 인터셉터용 (Static) | PrivateInfoInterceptor |

### 9.4 어노테이션 기반 자동 암복호화

```java
// VO 필드에 어노테이션 선언
public class UserVO {
    @Encrypt
    @Decrypt
    private String identityNo;  // 주민등록번호

    @Encrypt
    @Decrypt
    @Masking(MaskingType.EMAIL)
    private String email;       // 이메일
}

// DAO 메서드에 @PrivateSql 선언
@PrivateSql(encrypt = true, decrypt = true, masking = true)
UserVO selectUser(Long userNo);
```

### 9.5 GCM 모드 특징

- **암호화 + 무결성 검증**: 데이터 변조 감지
- **랜덤 IV**: 동일 평문도 매번 다른 암호문
- **레거시 CBC 호환**: 기존 데이터 자동 복호화 지원 (GCM 실패 시 CBC 폴백)

### 9.6 키 관리

```yaml
aws:
  aes:
    key: ${AES_KEY}  # AWS Parameter Store에서 주입
```

| 키 종류 | 저장 위치 | 갱신 주기 |
|---------|----------|----------|
| AES 키 | AWS Parameter Store | 수동 (필요 시), KMS 암호화 |

### 9.7 에러 코드

| 코드 | 이름 | 설명 |
|------|------|------|
| 34003 | ENCRYPTION_FAILED | AES 암호화/복호화 실패 |

---

## 10. 보안 절대 금지 사항

- 개인키 외부 노출
- 하드코딩된 암호화 키
- 자체 비밀번호 기반 로그인 구현 (OAuth2 전용)
- Application/System에서 JWT 생성/파싱/검증
- Gateway 인증 헤더 검증 누락
- 로그에 비밀키/개인정보 출력

---

END OF FILE
