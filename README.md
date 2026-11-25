# IFTTT로 알림 받기
# Arduino + p5.js + ThingSpeak LED Temperature Alert

## 1. 프로젝트 개요

Arduino와 p5.js를 이용하여 가변저항 값과 LED 상태, LED 주변 온도를 측정하고 ThingSpeak에 저장하는 프로젝트이다.

온도센서 값이 일정 온도 이상이 되면 ThingSpeak의 React와 MATLAB Analysis를 이용하여 이메일 경고가 전송되도록 구현하였다.

전체 동작 구조는 다음과 같다.

```text
가변저항 / LED / 온도센서
          ↓
       Arduino
          ↓ Serial
        p5.js
          ↓
      ThingSpeak
          ↓
 Field 3 온도 > 50℃
          ↓
        React
          ↓
   MATLAB Analysis
          ↓
   ThingSpeak Alerts API
          ↓
      이메일 알림
```

---

## 2. ThingSpeak Field 구성

ThingSpeak 채널은 다음과 같이 구성하였다.

| Field | 데이터 |
|---|---|
| Field 1 | 가변저항 값 |
| Field 2 | LED 상태 (0 / 1) |
| Field 3 | LED Temperature |

Arduino에서 받은 데이터는 p5.js를 통해 약 20초 간격으로 ThingSpeak에 전송된다.

---



## 3. ThingSpeak React 설정

LED 온도가 일정 값 이상이 되면 MATLAB Analysis가 실행되도록 React를 설정하였다.

설정은 다음과 같다.

```text
React Name:
LED Overheat

Condition Type:
Numeric

Test Frequency:
On Data Insertion

Field:
3 (LED Temperature)

Condition:
is greater than

Value:
50

Action:
MATLAB Analysis
```

Options는 다음과 같이 설정하였다.

```text
Run action only the first time the condition is met
```

온도가 계속 50℃ 이상일 때마다 이메일이 반복해서 오는 것을 방지하기 위해 처음 조건을 만족했을 때만 실행하도록 하였다.


<img width="641" height="539" alt="APPs_React" src="https://github.com/user-attachments/assets/9f40ef3f-352f-4c84-afaf-6279b63cbd4b" />



---

## 4. MATLAB Analysis 이메일 알림

ThingSpeak의 Alerts API를 사용하여 계정 이메일로 경고 메시지를 전송하였다.

```matlab
alertApiKey = "YOUR_ALERTS_API_KEY";

alertUrl = "https://api.thingspeak.com/alerts/send";

options = weboptions( ...
    "HeaderFields", ...
    ["ThingSpeak-Alerts-API-Key", alertApiKey] ...
);

subject = "LED 온도 경고";
body = "LED 온도가 50도를 초과했습니다.";

try
    webwrite( ...
        alertUrl, ...
        "body", body, ...
        "subject", subject, ...
        options ...
    );
catch err
    fprintf("메일 전송 실패: %s\n", err.message);
end
```

직접 `Save and Run`을 실행하여 이메일이 정상적으로 오는 것을 확인한 후 React와 연결하였다.

<img width="624" height="798" alt="ThingHTTP" src="https://github.com/user-attachments/assets/bdc7e185-af3e-481e-af49-d00c72df12f2" />


---

## 5. 이메일 알림 동작

최종 동작은 다음과 같다.

```text
LED 주변 온도 측정
        ↓
Arduino에서 TEMP 값 생성
        ↓
p5.js에서 수신
        ↓
ThingSpeak Field 3 저장
        ↓
온도 > 50℃
        ↓
React 실행
        ↓
MATLAB Analysis 실행
        ↓
이메일 경고 발송
```

예시:

```text
Field 3 = 35℃
→ 아무 동작 없음

Field 3 = 55℃
→ LED Overheat React 실행
→ 이메일 발송
```

<img width="1080" height="1575" alt="image" src="https://github.com/user-attachments/assets/366b9461-ab9c-40ce-8551-d7ec69fbe4c0" />




---

## 6. 문제 해결 과정

### 1. 온도값이 비정상적으로 출력되는 문제

처음에는 온도가 `-50℃`, `0℃`처럼 비정상적인 값으로 나타났다.

원인은 2핀 NTC 서미스터의 전압분배 방식과 계산식이 맞지 않았기 때문이다.

서미스터와 10kΩ 저항을 이용하여 전압분배 회로를 구성하고 다음 식으로 서미스터 저항값을 계산하였다.

```cpp
float resistance =
  R_FIXED * raw / (1023.0 - raw);
```

이후 Beta 식을 이용하여 온도를 계산하였다.

---

### 2. ThingSpeak Field 3이 비어 있던 문제

처음 p5.js 코드는 다음과 같이 Field 1과 Field 2만 전송하고 있었다.

```javascript
'&field1=' + potValue +
'&field2=' + ledValue
```

온도를 기록하기 위해 Field 3을 추가하였다.

```javascript
'&field3=' + temperature
```

---

### 3. 이메일이 자동으로 오지 않던 문제

MATLAB Analysis에서 직접 `Run`을 실행했을 때 이메일은 정상적으로 도착했지만 React에서는 자동으로 실행되지 않는 문제가 있었다.

React의 Action을 기존 ThingHTTP에서 MATLAB Analysis로 변경하고 Field 3의 조건을 연결하였다.

---

### 4. `429 Too Many Requests`

이메일을 반복해서 테스트하는 과정에서 다음 오류가 발생하였다.

```text
429 Too Many Requests
```

ThingSpeak Alerts API는 짧은 시간에 너무 많은 이메일 요청을 보내면 제한이 걸린다.

따라서 React 옵션을

```text
Run action only the first time the condition is met
```

으로 설정하여 불필요한 반복 발송을 막았다.

---

## 7. 고찰

처음에는 LED의 온도를 ThingSpeak에 바로 전송하면 쉽게 알림을 만들 수 있을 것이라고 생각했지만, 실제로는 온도값을 얻기 위해 별도의 온도센서가 필요했다.

또한 2핀 서미스터는 값을 바로 읽는 것이 아니라 고정 저항과 함께 전압분배 회로를 만들어야 한다는 것을 알게 되었다.

ThingSpeak에서는 단순히 데이터를 저장하는 것뿐만 아니라 React를 이용해 특정 조건을 확인하고, MATLAB Analysis와 연결하여 이메일 같은 동작까지 자동으로 실행할 수 있다는 점이 인상적이었다.

특히 자동 알림을 구현하면서 센서값 측정, Serial 통신, p5.js, ThingSpeak, MATLAB Analysis가 각각 따로 동작하는 것이 아니라 하나의 흐름으로 연결된다는 것을 이해할 수 있었다.
