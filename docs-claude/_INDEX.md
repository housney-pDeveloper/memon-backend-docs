# docs-claude 역할별 문서 로딩 가이드

이 문서는 Claude Code가 작업 수행 시 역할(Role)별로 어떤 문서를 로딩해야 하는지 정의한다.

---

## 1. 역할별 필수 참조 문서

| 역할/작업 | 필수 로드 | 선택 로드 |
|-----------|----------|----------|
| Gateway 개발 | 01_architecture, 02_security, 04_backend/CODE_CONVENTION, 04_backend/GATEWAY | 07_process |
| Application 개발 | 01_architecture, 02_security, 03_data, 04_backend/CODE_CONVENTION, 04_backend/APPLICATION | 07_process |
| System 개발 | 01_architecture, 02_security, 03_data, 05_infra, 04_backend/CODE_CONVENTION, 04_backend/SYSTEM | 07_process |
| DB 설계/변경 | 03_data | 01_architecture |
| API 문서 작성 | 06_api-docs | 04_backend/APPLICATION |
| 보안 점검 | 02_security | 01_architecture |
| MQ 작업 | 05_infra, 04_backend/SYSTEM | 01_architecture |
| 전체 리뷰 | 01_architecture, 07_process | 전체 |

---

## 2. 문서 디렉토리 구조

```
docs-claude/
├── _INDEX.md                          ← 이 문서 (진입점)
├── 01_architecture/ARCHITECTURE.md    ← 전체 아키텍처, 서버 역할, 통신, 시퀀스 다이어그램
├── 02_security/SECURITY.md            ← 전 모듈 보안 정책 (OAuth2, JWT, 헤더검증, 암호화)
├── 03_data/DATABASE.md                ← DB 정책 (스키마, 테이블, 네이밍, DML, 매퍼)
├── 04_backend/
│   ├── CODE_CONVENTION.md             ← 백엔드 공통 컨벤션 (DI, URL, 클래스명, VO/DTO, Validation)
│   ├── GATEWAY.md                     ← Gateway 고유 구현 (라우팅, Reactive, 에러코드)
│   ├── APPLICATION.md                 ← Application 고유 구현 (버저닝, 에러코드, 디렉토리)
│   └── SYSTEM.md                      ← System 고유 구현 (프로파일, 이벤트처리, 에러코드)
├── 05_infra/INFRASTRUCTURE.md         ← MQ 리소스, 동기화, 미래 로드맵, WebSocket
├── 06_api-docs/API_DOCUMENTATION.md   ← API 문서 생성 규칙 (콘텐츠, URL, 디렉토리)
└── 07_process/PROCESS.md              ← Git, 에러코드 글로벌, 명령 라우팅
```

---

## 3. 로딩 원칙

1. 작업 시작 전 이 문서(_INDEX.md)를 먼저 읽는다
2. Agent 역할 기반 수행 시에는 AGENT_DOCS_MAPPING.md를 우선 참조한다
3. 역할에 맞는 필수 문서만 로드한다
4. 모든 문서를 한 번에 읽지 않는다
5. 다른 역할의 문서를 임의로 적용하지 않는다
6. 단순 필드 추가 등 소규모 작업 시 아키텍처 문서 재로딩 금지

---

## 4. 조건부 로딩 규칙

| 작업 유형 | 추가 로딩 문서 |
|-----------|--------------|
| Validation 수정 | 04_backend/CODE_CONVENTION (Validation 섹션) |
| Mapper/SQL 수정 | 03_data (매퍼 규칙 섹션) |
| 에러코드 추가 | 07_process (에러코드 글로벌 섹션) + 해당 모듈 문서 |
| 응답 구조 수정 | 04_backend/CODE_CONVENTION (응답 정책 섹션) |
| MQ 리소스 변경 | 05_infra (MQ 레지스트리 섹션) |
| 암호화 수정 | 02_security (암호화 섹션) |

---

END OF FILE
