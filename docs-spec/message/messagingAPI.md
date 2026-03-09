# CLAUDE.md — External Messaging API (SMS/LMS/MMS/카카오 알림톡 + 버튼) 적용 가이드
(WiseT RESTful API v2.6.1 기준 / 이 파일은 Claude Code가 구현 시 준수해야 할 규칙이다)

---

## 0. 목표
- 우리 시스템에서 외부 메시지 발송 연동 시 다음 4가지만 지원한다.
    1) SMS
    2) LMS
    3) MMS
    4) 카카오 알림톡 (버튼 포함)

- 친구톡 / 브랜드메시지 / 동보발송 등은 구현하지 않는다.
- 알림톡은 버튼 기능을 반드시 지원한다.
- 버튼은 문서의 “버튼 발송방법” 규칙을 따르되, 이번 범위에서는 알림톡 버튼만 구현한다.

---

# 1. 공통 규칙

## 1.1 Request 형식
- 모든 발송 요청은 JSON Array 형식으로 전송한다.
- 단건도 배열로 보낸다.
- 공통 식별자:
    - custMsgSn: 고객사 발송 요청 ID (Unique)
    - sn: 중계사 발송 요청 ID (응답)

## 1.2 문자 Byte 규칙
- 영문/숫자 = 1 byte
- 한글 = 2 byte
- SMS: 90 byte
- LMS/MMS: 2000 byte
- 서버에서 반드시 byte 계산 후 검증한다.

---

# 2. SMS / LMS / MMS

## 2.1 SMS

### 필수 필드
- custMsgSn: string (required)
- phoneNum: string (required)
- smsSndNum: string (required)
- message: string (required, <= 90 bytes)

### Validation
- message byte length <= 90
- message blank 불가
- phoneNum blank 불가
- smsSndNum blank 불가

---

## 2.2 LMS

### 필수 필드
- custMsgSn: string (required)
- phoneNum: string (required)
- smsSndNum: string (required)
- subject: string (required, <= 40 chars)
- message: string (required, <= 2000 bytes)

### Validation
- subject blank 불가
- subject length <= 40
- message byte length <= 2000
- phoneNum/smsSndNum blank 불가

---

## 2.3 MMS

### 필수 필드
- custMsgSn: string (required)
- phoneNum: string (required)
- smsSndNum: string (required)
- subject: string (required, <= 40 chars)
- message: string (required, <= 2000 bytes)
- mmsCode1: string (required)
- mmsCode2: string (optional)
- mmsCode3: string (optional)

### Validation
- mmsCode1 blank 불가
- subject length <= 40
- message byte length <= 2000
- phoneNum/smsSndNum blank 불가

---

# 3. 카카오 알림톡 (버튼 포함)

## 3.1 기본 필드

- custMsgSn: string (required)
- senderKey: string (required)
- phoneNum: string (required)
- templateCode: string (required)
- message: string (required)

---

# 4. 알림톡 버튼 기능

## 4.1 button 필드

- button: JSON Array (optional)
- 최대 5개까지 허용

### 버튼 기본 구조 예시

{
"name": "홈페이지 바로가기",
"type": "WL",
"url_mobile": "https://m.example.com",
"url_pc": "https://www.example.com"
}

---

## 4.2 버튼 공통 필드

- name: 버튼에 표시될 텍스트 (required)
- type: 버튼 타입 코드 (required)
- url_mobile: 모바일 URL (타입에 따라 필수)
- url_pc: PC URL (타입에 따라 필수)
- scheme_android: 안드로이드 앱 스킴 (선택)
- scheme_ios: iOS 앱 스킴 (선택)

---

## 4.3 지원 버튼 타입 (알림톡 범위)

아래 타입만 구현한다.

- WL : 웹링크
- AL : 앱링크
- DS : 배송조회
- BK : 봇키워드
- MD : 메시지전달

친구톡 전용 타입은 구현하지 않는다.

---

## 4.4 버튼 Validation 규칙

### 공통
- button 최대 5개
- name 필수
- type 필수
- type은 enum으로 제한

### WL (웹링크)
- url_mobile 필수
- url_pc 권장

### AL (앱링크)
- scheme_android 또는 url_mobile 중 최소 1개
- scheme_ios 또는 url_mobile 중 최소 1개

### DS / BK / MD
- 문서 정의에 맞는 필드 필수 검증
- 내부 enum으로 제한

---

# 5. 알림톡 실패 시 문자 전환 (우회문자)

알림톡 요청 시 아래 필드를 함께 보낼 수 있다.

- smsSndNum
- smsKind ('S'|'L'|'T')
- smsMessage
- lmsMessage
- subject
- mmsCode1
- mmsCode2
- mmsCode3

## 5.1 조건부 규칙

smsKind = 'S'
- smsMessage 필수 (<=90 bytes)
- smsSndNum 필수

smsKind = 'L'
- lmsMessage 필수 (<=2000 bytes)
- subject 필수
- smsSndNum 필수

smsKind = 'T'
- lmsMessage 필수
- subject 필수
- mmsCode1 필수
- smsSndNum 필수

잘못된 조합은 Validation Error 처리한다.

---

# 6. Response 모델

모든 발송 응답은 공통 구조로 파싱한다.

- sn
- custMsgSn
- code
- reqDtm
- smsCode
- smsMsg
- smsSndDtm
- smsRcptDtm
- carrierCode

주의:
- code는 접수 결과
- 최종 발송 상태는 별도 결과조회 API로 확인 가능
- 발송요청과 발송결과조회 로직을 분리 설계한다.

---

# 7. 코드 구조 지침

Claude Code는 아래 구조를 생성해야 한다.

## 7.1 Enum 정의

public enum MessageType {
SMS,
LMS,
MMS,
ALIMTALK
}

public enum AlimTalkButtonType {
WL,
AL,
DS,
BK,
MD
}

---

## 7.2 서비스 구조

- MessageSendService
    - sendSms(List<SmsRequest>)
    - sendLms(List<LmsRequest>)
    - sendMms(List<MmsRequest>)
    - sendAlimTalk(List<AlimTalkRequest>)

- Validation 로직 분리
- External DTO Mapper 분리
- WiseTClient (WebClient 기반) 구현

---

# 8. 금지 사항

- 친구톡 구현 금지
- 브랜드메시지 구현 금지
- 동보 발송 구현 금지
- 문서 외 버튼 타입 확장 금지

---

# 9. 구현 전 체크리스트
- [ ] 알림톡 버튼 최대 5개 제한 적용
- [ ] 버튼 타입 enum 제한 적용
- [ ] Byte length 검증 존재
- [ ] 우회문자 조건부 필드 검증 완료
- [ ] Request는 JSON Array로 전송
- [ ] Response sn/custMsgSn 파싱
- [ ] 개인정보 로그 마스킹 처리

---
END