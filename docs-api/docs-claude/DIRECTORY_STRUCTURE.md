# DIRECTORY_STRUCTURE.md

docs 디렉토리의 구조 규칙을 정의한다.

---

## 1. 디렉토리 구조

```
docs
├─ biz
│   └─ login
│   │   └─ loginController.html
│   └─ signup
│       └─ signupController.html
├─ system
└─ index.html
```

---

## 2. 구조 규칙

### 2-1. biz 디렉토리

- application의 biz 내부의 package별로 구분
- controller package 내의 controller.class당 html 파일을 하나 생성
- 파일명: `{controllerName}.html`

### 2-2. system 디렉토리

- system 프로젝트 내 API를 작성
- 구조는 biz와 동일

### 2-3. index.html

- 모든 html 문서를 통합한 문서
- 전체 API 목록 및 링크 제공

---

## 3. 파일 생성 예시

| 소스 위치 | 문서 위치 |
|----------|----------|
| application/biz/login/controller/LoginController.java | docs/biz/login/loginController.html |
| application/biz/signup/controller/SignupController.java | docs/biz/signup/signupController.html |
| system/api/code/controller/CodeController.java | docs/system/code/codeController.html |

---

END OF FILE
