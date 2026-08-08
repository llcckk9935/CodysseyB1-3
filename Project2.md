[프로젝트2_AI_고객리뷰_감정분석_및_불만자동알림_최종과제.md](https://github.com/user-attachments/files/30863054/2_AI_._._._._.md)
# 프로젝트 2 — AI 고객 리뷰 감정 분석 및 불만 자동 알림

> **분야:** AI 도구 학습  
> **과정:** 노코드 자동화 기초: 워크플로우 설계  
> **자동화 도구:** Make  
> **구현 상태:** 자동 실행, 조건 분기, AI 분석, 오류 알림 및 Retry 복구 검증 완료

---

## 1. 프로젝트 개요

### 1.1 프로젝트명

**AI 고객 리뷰 감정 분석 및 불만 자동 알림**

### 1.2 자동화할 반복 업무

고객 리뷰를 사람이 하나씩 읽고 내용을 요약한 뒤, 긍정·중립·부정으로 분류하고 담당자의 대응이 필요한 리뷰를 찾아 알림을 보내는 업무를 자동화하였다.

### 1.3 프로젝트 목적

고객이 HTML 기반 리뷰 입력 페이지에서 고객명과 리뷰를 작성해 제출하면 Make Webhook이 데이터를 받아 자동화를 시작한다. Make AI Toolkit은 리뷰를 분석하여 다음 세 가지 결과를 생성한다.

1. 고객 리뷰 한 문장 요약
2. 감정 분류: `긍정`, `중립`, `부정`
3. 대응 필요 여부: `YES`, `NO`

원본 리뷰와 AI 분석 결과는 모두 Google Sheets에 저장한다. AI가 `action_required = YES`로 판단한 경우에만 Gmail을 통해 담당자에게 고객 대응 필요 알림을 발송한다.

### 1.4 구현 범위

- HTML 고객 리뷰 입력 페이지
- Webhook을 이용한 자동 실행
- 생성형 AI 기반 요약·감정 분석·대응 필요 여부 판단
- Text Parser를 이용한 AI 결과 분리
- Google Sheets 기록
- 대응 필요 여부에 따른 조건 분기
- 대응 필요 리뷰의 Gmail 알림
- Google Sheets 저장 오류 발생 시 실패 알림
- Incomplete Execution 저장 및 Retry
- 설정 정상화 후 재시도 성공과 `Resolved` 상태 확인

---

## 2. 자동화 도구 선정

### 2.1 선정 도구

프로젝트 2의 자동화 도구로 **Make**를 선정하였다.

### 2.2 선정 이유

프로젝트 1에서 Make와 Zapier를 직접 비교한 결과, Make는 전체 워크플로우와 조건 분기를 한 화면에서 확인하기 쉬웠다. 테스트 실행 과정이 비교적 매끄러웠고, 설정 실수가 발생했을 때 문제가 생긴 모듈을 찾기도 더 편리했다.

이번 프로젝트에는 Webhook, AI 분석, Text Parser, Google Sheets, Filter, Gmail 및 Error Handler를 하나의 흐름으로 연결해야 했다. 따라서 여러 단계와 정상·오류 경로를 시각적으로 확인할 수 있는 Make가 적합하다고 판단하였다.

### 2.3 사용 도구

| 도구 | 사용 목적 |
|---|---|
| HTML 기반 고객 리뷰 입력 페이지 | 고객명과 고객 리뷰 입력 및 제출 |
| Make Webhook | 리뷰 데이터를 받아 자동화 시작 |
| Make AI Toolkit | 리뷰 요약, 감정 분석, 대응 필요 여부 판단 |
| Text Parser | AI 분석 결과를 후속 단계에서 사용할 값으로 분리 |
| Google Sheets | 원본 리뷰와 분석 결과 저장 |
| Make Filter | `action_required = YES` 조건 확인 |
| Gmail | 고객 대응 필요 알림 및 자동화 오류 알림 발송 |
| Error Handler·Retry | 저장 오류 처리, 실패 실행 보관 및 재시도 |

---

## 3. 고객 리뷰 입력 페이지

고객 리뷰 입력 페이지는 HTML로 구성하였다. 고객은 페이지에서 다음 정보를 작성해 제출한다.

- 고객명
- 고객 리뷰

제출된 데이터는 Make Webhook으로 전달된다. 사용자는 Make에서 `Run once`를 누르거나 Google Sheets에 직접 내용을 입력할 필요가 없다. Scenario가 활성화된 최종 상태에서는 고객이 입력 페이지의 리뷰를 제출하는 행동 자체가 자동화의 시작 조건이 된다.

<img width="640" height="707" alt="스크린샷 2026-08-09 040739" src="https://github.com/user-attachments/assets/8e96202e-a0c3-4e2d-a2b2-87bfbd758037" />



---

## 4. Workflow 설계

### 4.1 전체 Workflow

```text
고객 리뷰 입력 페이지
        ↓
Webhook
        ↓
Make AI Toolkit
        ↓
Text Parser
        ↓
Google Sheets
        ↓
Filter: action_required = YES
        ↓
Gmail 고객 대응 필요 알림
```

`action_required = NO`인 리뷰는 Google Sheets에 저장된 후 Gmail 단계로 전달되지 않고 정상 종료된다.

### 4.2 오류 처리 Workflow

```text
Google Sheets 저장 오류
        ↓
Error Handler
        ↓
오류 알림 Gmail
        ↓
Retry
        ↓
Incomplete Execution 저장
        ↓
예약된 재시도 실행
        ↓
성공 시 Resolved
```

<img width="1070" height="900" alt="스크린샷 2026-08-09 045514" src="https://github.com/user-attachments/assets/fa56daca-04c1-4f04-842d-ff6493915323" />


---

## 5. 단계별 자동화 흐름

### 5.1 Webhook — Trigger

고객이 HTML 입력 페이지에서 고객명과 리뷰를 제출하면 해당 데이터가 Make Webhook으로 전달된다. Webhook이 데이터를 받는 사건이 전체 자동화를 시작하는 Trigger이다.

### 5.2 Make AI Toolkit — AI Action

Make AI Toolkit이 고객 리뷰를 분석하여 다음 값을 생성한다.

- 한 문장 요약
- 감정: 긍정·중립·부정
- 대응 필요 여부: YES·NO

### 5.3 Text Parser — 분석 결과 분리

Text Parser는 AI가 생성한 결과에서 요약, 감정, 대응 필요 여부를 분리한다. 분리된 값은 Google Sheets 저장과 이후 Filter 조건 판단에 사용된다.

<img width="449" height="692" alt="스크린샷 2026-08-09 040927" src="https://github.com/user-attachments/assets/e4f3d2d1-3ed5-4e35-8789-4b326c8272aa" />


### 5.4 Google Sheets — 전체 결과 저장

AI 분석이 끝난 모든 리뷰는 대응 필요 여부와 관계없이 Google Sheets에 저장된다.

| 저장 항목 | 내용 |
|---|---|
| 접수일시 | 리뷰가 접수된 일시 |
| 고객명 | 입력 페이지에서 제출한 고객명 |
| 고객리뷰 | 고객이 작성한 원본 리뷰 |
| AI요약 | AI가 생성한 한 문장 요약 |
| 감정 | 긍정·중립·부정 중 하나 |
| 대응필요 | YES 또는 NO |

https://docs.google.com/spreadsheets/d/1j0Z9PHBuGQbaXotKVOnfdHCCjth7CknSQ3cE_imxDL4/edit?gid=0#gid=0
<img width="1069" height="612" alt="image" src="https://github.com/user-attachments/assets/96d2df4c-6985-4a04-927c-ff50ded4e500" />


### 5.5 Filter — 조건 분기

Google Sheets 저장 후 Filter가 `action_required` 값을 확인한다.

| 조건 | 후속 처리 |
|---|---|
| `action_required = YES` | Gmail 알림 발송 |
| `action_required = NO` | Google Sheets 저장 후 종료 |

따라서 모든 리뷰는 기록하되, 담당자가 확인해야 하는 리뷰에 대해서만 추가 알림을 보내도록 구성하였다.

### 5.6 Gmail — 고객 대응 필요 알림

Filter를 통과한 `action_required = YES` 리뷰는 Gmail Action으로 전달된다. 담당자는 이메일을 통해 대응이 필요한 고객 리뷰가 접수되었음을 확인할 수 있다.

<img width="1082" height="710" alt="image" src="https://github.com/user-attachments/assets/36711d8a-0614-4965-9fba-36bda76e2b01" />


---

## 6. 자동 실행 설정

구현 초기에는 Make의 `Run once` 기능을 사용하여 각 모듈의 데이터 전달과 실행 결과를 테스트하였다. 최종 단계에서는 Scenario를 활성화하고 실행 일정을 **`Immediately as data arrives`**로 설정하였다.

최종 자동 실행 과정은 다음과 같다.

1. 고객이 리뷰 입력 페이지에서 내용을 제출한다.
2. Webhook이 데이터를 즉시 수신한다.
3. Make Scenario가 자동으로 실행된다.
4. AI 분석, 결과 분리, Sheets 저장 및 조건부 Gmail 발송이 순서대로 진행된다.

따라서 현재는 사용자가 `Run once`를 누르지 않아도 리뷰 제출만으로 자동화가 실행된다.

<img width="1070" height="900" alt="스크린샷 2026-08-09 045514" src="https://github.com/user-attachments/assets/73690cfd-a917-43b3-8069-a7cab60a84f2" />


---

## 7. 보너스 1 — 생성형 AI Action

Make AI Toolkit을 생성형 AI Action으로 사용하였다. AI는 고객 리뷰 원문을 받아 다음 세 가지 정보를 생성한다.

1. 리뷰의 핵심 내용을 한 문장으로 요약한다.
2. 리뷰 감정을 긍정·중립·부정으로 분류한다.
3. 담당자의 대응이 필요한지 YES·NO로 판단한다.

고객 리뷰는 표현 방식과 문장 길이가 일정하지 않은 비정형 텍스트이다. AI Action을 통해 이 내용을 요약·감정·대응 필요 여부라는 구조화된 정보로 변환하고, 후속 자동화에서 사용할 수 있도록 하였다.

AI 결과는 Text Parser를 거쳐 Google Sheets 열에 각각 저장되며, `action_required` 값은 Gmail 발송 여부를 결정하는 Filter 조건으로도 사용된다.

---

## 8. 보너스 2 — 실패 알림 및 자동 재시도

### 8.1 오류 처리 구조

Google Sheets 저장 단계에 Error Handler를 연결하였다. 정상 저장에 실패하면 오류 경로가 실행되고, 실패 사실을 이메일로 알린 뒤 해당 실행을 Incomplete Execution으로 저장하여 Retry할 수 있도록 구성하였다.

```text
Google Sheets 오류
        ↓
오류 알림 Gmail
        ↓
Retry
        ↓
Incomplete Execution
        ↓
자동 재시도
```

### 8.2 실패 알림

Google Sheets 저장 오류가 발생하면 담당자에게 다음 제목의 Gmail이 발송되도록 구현하였다.

```text
[자동화 오류] 고객 리뷰 처리 실패
```

이를 통해 담당자가 Make 실행 화면을 계속 확인하지 않아도 자동화 실패 사실을 알 수 있다.

**[스크린샷 삽입 7 - 자동화 오류 Gmail]**
<img width="1076" height="577" alt="image" src="https://github.com/user-attachments/assets/5cbb0c31-acd2-49b9-bccf-576ef4e444bc" />


### 8.3 오류 발생 테스트

Error Handler가 실제로 실행되는지 확인하기 위해 Google Sheets 설정을 의도적으로 잘못 지정하였다. 테스트 결과 다음 오류가 실제로 발생하였다.

```text
400 INVALID_ARGUMENT
Unable to parse range
```

오류 발생 후 확인한 결과는 다음과 같다.

1. Google Sheets 저장이 실패하였다.
2. Error Handler가 실행되었다.
3. 담당자에게 자동화 오류 Gmail이 발송되었다.
4. 실패한 실행이 Incomplete executions에 저장되었다.

<img width="1079" height="910" alt="스크린샷 2026-08-09 050014" src="https://github.com/user-attachments/assets/852a4c3b-567e-4e83-b120-0ccc8c81117c" />
<img width="775" height="831" alt="스크린샷 2026-08-09 050125" src="https://github.com/user-attachments/assets/ec5ca6af-9257-465a-9058-034ff1f6ba70" />




### 8.4 Retry 설정

실패한 실행은 Incomplete Execution으로 보관되었고, Retry가 `Scheduled` 상태로 예약되었다. 이 구조를 통해 오류가 발생한 데이터를 바로 버리지 않고, 원인을 수정한 후 다시 처리할 수 있도록 하였다.

<img width="1078" height="892" alt="스크린샷 2026-08-09 050242" src="https://github.com/user-attachments/assets/5dda83b3-fb83-4fd6-9aec-8134987e1501" />



### 8.5 실제 Retry 성공 검증

Retry는 설계만 한 것이 아니라 다음 과정으로 실제 복구 성공까지 확인하였다.

1. 잘못 지정한 Google Sheets 설정으로 저장 오류를 발생시켰다.
2. Error Handler와 실패 알림 Gmail 실행을 확인하였다.
3. 실패 실행이 Incomplete executions에 저장된 것을 확인하였다.
4. Retry가 `Scheduled` 상태로 예약된 것을 확인하였다.
5. Google Sheets 설정 오류를 정상화하였다.
6. 예약된 Retry를 실행하였다.
7. Google Sheets 저장이 성공한 것을 확인하였다.
8. Incomplete Execution 상태가 `Resolved`로 변경된 것을 확인하였다.
9. `Attempts = 1`을 확인하였다.

따라서 이번 테스트에서는 **오류 발생 → 실패 알림 → Incomplete Execution 저장 → Retry → 정상 복구 → Resolved**의 전체 과정을 실제로 검증하였다.

다만 실제 확인된 값은 `Attempts = 1`이다. 여러 차례 연속으로 실패하여 최대 재시도 횟수까지 모두 실행되는 상황을 검증한 것은 아니다.

<img width="1076" height="903" alt="스크린샷 2026-08-09 050258" src="https://github.com/user-attachments/assets/6fb11814-52ac-46f0-a798-6a411ee92549" />
<img width="1076" height="910" alt="스크린샷 2026-08-09 050330" src="https://github.com/user-attachments/assets/e51c739f-0cc0-4110-9b86-2e85a0fcee93" />


---

## 9. 테스트 결과

### 9.1 정상 처리 테스트

| 번호 | 리뷰 유형 | AI 감정 결과 | 대응 필요 | Google Sheets | Gmail | 실제 결과 |
|:---:|---|:---:|:---:|:---:|:---:|---|
| 1 | 배송 지연 + 제품 파손 + 환불 요청 | 부정 | YES | 저장 | 발송 | 정상 동작 |
| 2 | 배송 및 제품 만족 | 긍정 | NO | 저장 | 미발송 | 정상 동작 |
| 3 | 제품 문제 및 환불 요청 | 부정 | YES | 저장 | 발송 | 정상 동작 |
| 4 | 고객 문의 | 중립 | YES | 저장 | 발송 | 정상 동작 |
| 5 | 긍정 리뷰 | 긍정 | NO | 저장 | 미발송 | 정상 동작 |

테스트 결과, 모든 리뷰가 Google Sheets에 저장되었다. `대응필요 = YES`인 세 건에서는 Gmail이 발송되었고, `대응필요 = NO`인 두 건에서는 Gmail이 발송되지 않았다.

### 9.2 오류 및 복구 테스트

| 테스트 항목 | 기대 결과 | 실제 확인 결과 | 판정 |
|---|---|---|:---:|
| Google Sheets 범위 오류 발생 | 저장 실패 및 Error Handler 실행 | `400 INVALID_ARGUMENT`, `Unable to parse range` 발생 | 성공 |
| 실패 알림 | 담당자에게 오류 Gmail 발송 | `[자동화 오류] 고객 리뷰 처리 실패` 메일 수신 | 성공 |
| 실패 실행 보관 | Incomplete executions에 저장 | 실패 실행 저장 확인 | 성공 |
| Retry 예약 | Retry가 Scheduled 상태가 됨 | Scheduled 상태 확인 | 성공 |
| 설정 정상화 후 Retry | Google Sheets 저장 성공 | 재시도 후 저장 성공 | 성공 |
| 실행 상태 복구 | Incomplete Execution이 Resolved로 변경 | `Resolved`, `Attempts = 1` 확인 | 성공 |

---

## 10. Trigger / Action / 조건 분기 정리

### 10.1 Trigger

Trigger는 자동화를 시작시키는 사건이다.

이번 프로젝트의 Trigger는 다음과 같다.

> 고객이 HTML 리뷰 입력 페이지에서 고객명과 리뷰를 제출하여 Make Webhook이 데이터를 수신하는 것

### 10.2 Action

Action은 Trigger 이후 자동으로 수행되는 처리 동작이다.

이번 프로젝트에 포함된 주요 Action은 다음과 같다.

| Action | 역할 |
|---|---|
| Make AI Toolkit | 리뷰 요약, 감정 분석, 대응 필요 여부 생성 |
| Text Parser | AI 출력에서 필요한 값을 분리 |
| Google Sheets | 원본 리뷰와 AI 분석 결과 저장 |
| Gmail | 고객 대응 필요 알림 발송 |
| 오류 알림 Gmail | Google Sheets 저장 실패 사실 통보 |
| Retry | 실패한 실행을 다시 처리 |

### 10.3 조건 분기

조건 분기는 모든 데이터를 같은 방식으로 처리하지 않고, 지정한 조건에 따라 후속 실행 여부를 결정하는 기능이다.

이번 프로젝트의 조건은 다음과 같다.

```text
action_required = YES → Gmail 발송
action_required = NO  → Google Sheets 저장 후 종료
```

Make Filter는 `action_required = YES`인 리뷰만 Gmail 단계로 통과시킨다. `NO`인 리뷰도 Google Sheets에는 저장되지만 담당자 알림은 발송되지 않는다.

---

## 11. 구현 결과

본 프로젝트는 다음 기능을 실제로 구현하고 검증하였다.

- 고객 리뷰 제출을 Webhook으로 수신
- Webhook 수신 즉시 Scenario 자동 실행
- AI를 이용한 한 문장 요약 생성
- 긍정·중립·부정 감정 분류
- 대응 필요 여부 YES·NO 판단
- 모든 리뷰와 AI 분석 결과의 Google Sheets 저장
- 대응 필요 리뷰에 한정한 Gmail 발송
- Google Sheets 오류 발생 시 실패 알림 Gmail 발송
- 실패 실행의 Incomplete Execution 저장
- Retry Scheduled 상태 확인
- 설정 정상화 후 Retry 성공
- `Resolved` 및 `Attempts = 1` 확인

이를 통해 공통 요구사항인 Trigger 1개 이상, Action 2개 이상, 조건 분기 1개 이상 및 실제 자동 실행 구조를 모두 충족하였다. 생성형 AI Action과 실패 알림·재시도도 실제 워크플로우에 포함하였다.

---

## 12. 한계 및 개선 방향

### 12.1 현재 확인된 한계

- AI의 감정 및 대응 필요 여부 판단은 입력 문장에 따라 달라질 수 있다.
- 이번 정상 동작 검증은 제시한 다섯 가지 리뷰 유형을 대상으로 진행했으며, 가능한 모든 고객 표현을 검증한 것은 아니다.
- Retry 복구는 `Attempts = 1`에서 성공한 결과를 확인하였다. 여러 차례 연속 오류가 발생하는 상황까지 검증한 것은 아니다.

### 12.2 향후 검토할 사항

- 애매한 표현, 감정이 섞인 리뷰 등 테스트 사례를 늘려 AI 분류 결과를 확인할 필요가 있다.
- 실제 운영 중 잘못 분류된 사례가 발견되면 AI 판단 기준을 점검할 필요가 있다.
- Google Sheets 오류가 연속으로 발생하는 상황에서 Retry가 어떻게 동작하는지 추가 검증할 수 있다.
- 제출용 캡처에서는 Webhook 주소, 계정 이메일 및 기타 민감정보가 노출되지 않도록 마스킹해야 한다.

위 항목은 현재 구현된 기능이 아니라, 실제 운영 범위를 확대할 때 검토할 수 있는 사항이다.

---

## 13. 최종 결과

본 프로젝트에서는 고객 리뷰 확인과 분류 업무를 자동화하기 위해 **HTML 입력 페이지 → Webhook → Make AI Toolkit → Text Parser → Google Sheets → Filter → Gmail**로 이어지는 워크플로우를 구현하였다.

고객이 리뷰를 제출하면 Make Scenario가 자동으로 시작되고, AI가 리뷰를 요약한 뒤 감정과 대응 필요 여부를 판단한다. 모든 결과는 Google Sheets에 기록되며, `action_required = YES`인 경우에만 담당자에게 Gmail이 발송된다. 긴급 대응이 필요하지 않은 리뷰는 기록 후 정상 종료되므로 불필요한 알림도 줄일 수 있다.

또한 Google Sheets 오류를 실제로 발생시켜 Error Handler, 오류 알림, Incomplete Execution 저장, Retry 예약 및 정상 복구 과정을 검증하였다. 설정을 정상화한 뒤 재시도에서 Google Sheets 저장이 성공했고, 실행 상태가 `Resolved`, `Attempts = 1`로 변경된 것까지 확인하였다.

이번 프로젝트를 통해 Trigger와 Action의 연결, AI 분석 결과를 이용한 조건 분기, Webhook 기반 자동 실행, 실행 실패에 대비한 오류 처리 및 재시도의 흐름을 직접 구현하고 설명할 수 있게 되었다.

---
