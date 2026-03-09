# BUILD_AND_PROMPT_POLICY.md

## 1. [build]

- application, gateway 각각 gradle build 수행
- gradlew build -x test
- 에러 발생 시 원인 분석 후 수정
- build 성공 시 run 수행
- 테스트는 수행하지 않음
- 해결 방안이 2가지 이상이면 질문

---

## 2. [prompt]

- ./prompt.md 읽고 수행
- 예시 코드는 참고만
- Code Convention은 각 프로젝트 CLAUDE.md 따름

---

## 3. [refactoring-claude]

- 최상위 CLAUDE.md 및 각 프로젝트 CLAUDE.md 읽고 리팩토링
- 개선점 존재 시 제안