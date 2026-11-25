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

## 3. 회로 구성

### LED

```text
Arduino D13
    ↓
저항
    ↓
LED (+)

LED (-)
    ↓
GND
```

### 가변저항

```text
한쪽 핀   → 5V
가운데 핀 → A0
반대쪽 핀 → GND
```

### NTC 온도센서

2핀 10kΩ NTC 서미스터와 10kΩ 저항을 사용하여 전압 분배 회로를 구성하였다.

```text
5V
 │
[10kΩ NTC]
 │
 ├──── A1
 │
[10kΩ 저항]
 │
GND
```

브레드보드에서는 서미스터의 한쪽 다리, A1 선, 10kΩ 저항의 한쪽 다리가 같은 연결 지점에 위치하도록 구성하였다.

---

## 4. Arduino 코드

Arduino는 가변저항 값과 온도값을 Serial로 p5.js에 전송하고, p5.js로부터 `ON`, `OFF` 명령을 받아 LED를 제어한다.

```cpp
#include <math.h>

const int LED_PIN  = 13;
const int POT_PIN  = A0;
const int TEMP_PIN = A1;

// 10kΩ NTC 기준
const float R_FIXED = 10000.0;
const float R0      = 10000.0;
const float BETA    = 3950.0;
const float T0      = 298.15;

unsigned long previous = 0;

void setup() {
  Serial.begin(9600);

  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
}

void loop() {

  // 센서값 전송
  if (millis() - previous >= 200) {
    previous = millis();

    // 가변저항
    int potValue = analogRead(POT_PIN);

    Serial.print("POT:");
    Serial.println(potValue);


    // 온도센서
    int raw = analogRead(TEMP_PIN);

    if (raw > 0 && raw < 1023) {

      float resistance =
        R_FIXED * raw / (1023.0 - raw);

      float temperatureK =
        1.0 /
        (
          (1.0 / T0) +
          log(resistance / R0) / BETA
        );

      float temperatureC =
        temperatureK - 273.15;

      Serial.print("TEMP:");
      Serial.println(temperatureC, 1);
    }
  }


  // p5.js 명령 수신
  if (Serial.available()) {

    String command =
      Serial.readStringUntil('\n');

    command.trim();

    if (command == "ON") {
      digitalWrite(LED_PIN, HIGH);
      Serial.println("LED: ON");
    }

    else if (command == "OFF") {
      digitalWrite(LED_PIN, LOW);
      Serial.println("LED: OFF");
    }
  }
}
```

---

## 5. p5.js 코드

p5.js에서는 Arduino로부터 다음과 같은 데이터를 받는다.

```text
POT:523
TEMP:27.4
LED: ON
LED: OFF
```

받은 값은 화면에 표시하고 ThingSpeak의 Field 1~3에 업로드한다.

> 실제 GitHub에 올릴 때는 ThingSpeak Write API Key를 그대로 공개하지 않도록 주의한다.

```javascript
let serial, connectButton;

let ledOn = false;
let potValue = 0;
let temperature = 0;

let statusText = 'Arduino를 연결해 주세요.';
let serialMessage = '아직 없음';


// ThingSpeak 설정
const APIKey = "YOUR_WRITE_API_KEY";

const urlBase =
  "https://api.thingspeak.com/update.json?api_key=" +
  APIKey;

const updateInterval = 20000;

let nextUpdateTime = 0;
let dataChanged = false;
let sending = false;


function setup() {
  createCanvas(600, 450);
  textFont('sans-serif');

  serial = new SerialHelper(
    9600,
    receiveLine,
    showStatus
  );

  connectButton =
    createButton('Arduino 연결 / 해제');

  connectButton.mousePressed(
    () => serial.toggle()
  );
}


function draw() {
  background(242, 246, 250);

  fill(25, 40, 60);
  noStroke();
  textAlign(CENTER);

  textSize(24);
  text(
    'Arduino Uno R3 · ThingSpeak',
    width / 2,
    36
  );

  textSize(14);
  text(
    statusText,
    width / 2,
    66
  );


  // LED 상태
  fill(
    ledOn
      ? '#22c55e'
      : '#64748b'
  );

  circle(150, 170, 100);

  fill(255);
  textSize(20);

  text(
    ledOn ? 'ON' : 'OFF',
    150,
    177
  );


  // 가변저항
  fill(25, 40, 60);
  textSize(16);
  text('가변저항 A0', 420, 105);

  fill('#dbe3ed');
  rect(310, 125, 220, 35, 10);

  fill('#3b82f6');
  rect(
    310,
    125,
    map(potValue, 0, 1023, 0, 220),
    35,
    10
  );

  fill(25, 40, 60);
  textSize(18);

  text(
    potValue + ' / 1023',
    420,
    190
  );


  // 온도
  textSize(17);
  text(
    'LED Temperature',
    420,
    245
  );

  textSize(28);

  text(
    temperature.toFixed(1) + ' °C',
    420,
    285
  );


  textSize(13);

  text(
    'from Arduino : ' + serialMessage,
    width / 2,
    380
  );


  updateIoTchannel();
}


// LED ON/OFF
async function mousePressed() {

  if (
    dist(
      mouseX,
      mouseY,
      150,
      170
    ) > 50
  ) return;

  const next = !ledOn;

  if (
    await serial.send(
      next ? 'ON' : 'OFF'
    )
  ) {

    ledOn = next;
    dataChanged = true;
    updateIoTchannel();
  }
}


// Arduino 데이터 수신
function receiveLine(line) {

  const oldPot = potValue;
  const oldLed = ledOn;
  const oldTemp = temperature;

  const message =
    line.trim().toUpperCase();

  serialMessage = message;


  if (
    message === 'LED: ON' ||
    message === 'ON'
  ) {
    ledOn = true;
  }

  else if (
    message === 'LED: OFF' ||
    message === 'OFF'
  ) {
    ledOn = false;
  }

  else if (
    message.startsWith('POT:')
  ) {
    let value =
      Number(message.substring(4));

    if (!isNaN(value)) {
      potValue =
        constrain(value, 0, 1023);
    }
  }

  else if (
    message.startsWith('TEMP:')
  ) {
    let value =
      Number(message.substring(5));

    if (!isNaN(value)) {
      temperature = value;
    }
  }


  if (
    potValue !== oldPot ||
    ledOn !== oldLed ||
    temperature !== oldTemp
  ) {

    dataChanged = true;
    updateIoTchannel();
  }
}


// ThingSpeak 전송
function updateIoTchannel() {

  if (!dataChanged) return;
  if (sending) return;
  if (millis() < nextUpdateTime) return;

  const ledValue =
    ledOn ? 1 : 0;

  const url =
    urlBase +
    '&field1=' + potValue +
    '&field2=' + ledValue +
    '&field3=' + temperature.toFixed(1);

  sending = true;
  dataChanged = false;

  loadJSON(
    url,
    updateDone,
    updateFailed
  );
}


function updateDone(data) {

  if (
    !data ||
    !(data.entry_id > 0)
  ) {
    updateFailed(
      'ThingSpeak 기록 거부'
    );
    return;
  }

  console.log(
    'ThingSpeak 기록 성공:',
    data.entry_id
  );

  nextUpdateTime =
    millis() +
    updateInterval;

  sending = false;
}


function updateFailed(error) {

  console.warn(
    'ThingSpeak 기록 실패',
    error
  );

  dataChanged = true;

  nextUpdateTime =
    millis() +
    updateInterval;

  sending = false;
}


function showStatus(message) {
  statusText = message;
}
```

---

## 6. ThingSpeak React 설정

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

---

## 7. MATLAB Analysis 이메일 알림

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

---

## 8. 이메일 알림 동작

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

---

## 9. 문제 해결 과정

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

## 10. 고찰

처음에는 LED의 온도를 ThingSpeak에 바로 전송하면 쉽게 알림을 만들 수 있을 것이라고 생각했지만, 실제로는 온도값을 얻기 위해 별도의 온도센서가 필요했다.

또한 2핀 서미스터는 값을 바로 읽는 것이 아니라 고정 저항과 함께 전압분배 회로를 만들어야 한다는 것을 알게 되었다.

ThingSpeak에서는 단순히 데이터를 저장하는 것뿐만 아니라 React를 이용해 특정 조건을 확인하고, MATLAB Analysis와 연결하여 이메일 같은 동작까지 자동으로 실행할 수 있다는 점이 인상적이었다.

특히 자동 알림을 구현하면서 센서값 측정, Serial 통신, p5.js, ThingSpeak, MATLAB Analysis가 각각 따로 동작하는 것이 아니라 하나의 흐름으로 연결된다는 것을 이해할 수 있었다.
