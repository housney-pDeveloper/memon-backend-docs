# CLAUDE.md (Docs)

이 파일은 /application에서 제공하는 API를 문서화하여
/docs 디렉토리에 정적 HTML 문서로 생성하는 가이드를 정의한다.

Claude Code는 명령에 따라 코드를 생성할 때
반드시 이 파일을 읽은 후 작성한다.

---

## 1. 역할 정의

이 디렉토리는 API 문서화를 담당한다.
- Application 프로젝트의 API를 HTML 문서로 생성
- System 프로젝트의 API를 HTML 문서로 생성
- 프론트엔드 개발자를 위한 문서 제공

---

## 2. 절대 원칙

- 한글로 작성한다
- 모든 API는 Gateway를 통한 접근만 허용
- 문서의 URL은 외부 유입 URL 기준으로 작성 (Gateway 진입점 기준)
- 정해진 규격에 따라 작성

---

## 3. 권한 설정

- directory 생성 권한 부여
- html file 생성 권한 부여
- 별도의 질문없이 문서 작성 진행

---

## 4. docs-claude 참조 규칙

Claude Code는 작업 성격에 따라
docs-claude 문서를 선택적으로 읽어야 한다.

### 4-1. 문서 유형별 참조 기준

| 문서 | 참조 시점 |
|------|----------|
| DIRECTORY_STRUCTURE.md | 파일 생성 위치 결정 시 |
| CONTENT_POLICY.md | API 문서 내용 작성 시 |
| URL_POLICY.md | API URL 작성 시 |

---

## 5. 불확실할 경우

- 작업 중단
- 질문 후 진행

---

END OF FILE
