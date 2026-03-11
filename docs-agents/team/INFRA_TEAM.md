# INFRA_TEAM

인프라 구성 작업을 수행할 때 사용하는 Team이다.

---

## 1. 개요

INFRA_TEAM은 Docker, CI/CD, 클라우드 서비스 등
인프라를 구성하고 관리하는 Team이다.

---

## 2. Team 구성

| 순서 | 역할 | 책임 |
|------|------|------|
| 1 | AI_INFRA_ENGINEER | 인프라 구성 |
| 2 | AI_SECURITY_ENGINEER | 보안 설정 검토 |

---

## 3. 트리거 조건

다음 요청에서 이 Team이 선택된다:

- Docker 구성 요청
- CI/CD 구성 요청
- AWS 구성 요청
- 배포 설정 요청
- 인프라 변경 요청

---

## 4. 실행 워크플로우

```
┌─────────────────────┐
│ AI_INFRA_ENGINEER   │ 인프라 구성
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ AI_SECURITY_ENGINEER│ 보안 설정 검토
└─────────────────────┘
```

---

## 5. 작업 범위

### 5.1 컨테이너화

- Dockerfile 작성
- Docker Compose 구성
- 멀티스테이지 빌드
- 이미지 최적화

### 5.2 CI/CD

- GitLab CI 파이프라인
- 빌드 스크립트
- 배포 자동화
- 환경별 설정

### 5.3 메시지 큐

- RabbitMQ 설정
- Exchange/Queue 구성
- 클러스터 설정

### 5.4 캐시

- Redis 설정
- 클러스터 구성
- 캐시 정책

### 5.5 클라우드

- AWS 서비스 구성
- 인프라 프로비저닝
- 네트워크 설정

---

## 6. 보안 검토 항목

AI_SECURITY_ENGINEER는 다음을 검토한다:

- Secret 관리 (환경변수, Vault 등)
- 네트워크 보안 설정
- 컨테이너 보안 설정
- IAM 정책
- 포트 노출 범위
- 로깅 설정

---

## 7. 핸드오프 규칙

### 7.1 INFRA_ENGINEER → SECURITY_ENGINEER

전달 항목:
- 인프라 구성 파일
- 환경 변수 목록
- 네트워크 설정
- 접근 권한 설정

---

## 8. 실행 예시

### 요청

"Application 서버 Docker 구성을 만들어줘"

### 실행

```
[STEP 1] AI_INFRA_ENGINEER
- Dockerfile 작성
  - 멀티스테이지 빌드
  - JRE 21 기반 이미지
- docker-compose.yml 작성
  - 서비스 정의
  - 네트워크 설정
  - 볼륨 설정
- 환경 변수 설정
  - .env.example 작성

[STEP 2] AI_SECURITY_ENGINEER
- Secret 관리 검토
  - DB 비밀번호 환경변수화
  - API 키 관리 방식
- 네트워크 검토
  - 포트 노출 범위
  - 내부 네트워크 분리
- 컨테이너 보안 검토
  - non-root 사용자 사용
  - 불필요한 패키지 제거
```

---

## 9. 제약 사항

- 애플리케이션 코드 수정 금지
- Secret 하드코딩 금지
- 불필요한 포트 노출 금지

---

## 10. 관련 문서

- [AI_INFRA_ENGINEER](../agent/AI_INFRA_ENGINEER.md)
- [AI_SECURITY_ENGINEER](../agent/AI_SECURITY_ENGINEER.md)
- 01_docs/docs-docker/**

---

## Team 수행 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.

### 수행 순서 및 docs-claude 매핑

| 순서 | 역할 | 필수 docs-claude | 병렬 |
|------|------|-----------------|------|
| 1 | AI_INFRA_ENGINEER | 01_architecture, 05_infra | N |
| 2 | AI_SECURITY_ENGINEER | 01_architecture, 02_security | N |

### 핸드오프 흐름

```
prompt.md → task_prompt.md
→ result_AI_INFRA_ENGINEER.md → result_AI_SECURITY_ENGINEER.md
→ result_VERIFICATION.md
```

---

END OF FILE
