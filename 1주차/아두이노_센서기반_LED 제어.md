# 실제로 제가 학교 프로젝트에 사용한 아두이노를 사용하여 진동, 물감지, 거리센서를 사용하여 만든 스마트 지팡이에 사용한 아두이노 코드입니다.

// === 핀 설정 ===
#define TRIG_PIN 9
#define ECHO_PIN 8
#define WATER_SENSOR A0

#define VIB1_PIN 5
#define VIB3_PIN 4
#define VIB2_PIN 6
#define BUTTON_PIN 7
#define BUZZER_PIN 10
#define SIGNAL_PIN 11

const int xPin = A1;
const int yPin = A2;
const int zPin = A3;

#define DIST_THRESHOLD 50
#define WATER_THRESHOLD 600

const float zeroG = 512.0;
const float sensitivity = 102.3;

const unsigned long requiredDuration = 4000;
unsigned long stableStartTime = 0;
bool isStable = false;
bool sosSent = false;

// === 버튼 ===
bool system_on = false;
bool last_button_state = HIGH;
bool current_button_state = HIGH;
unsigned long last_debounce_time = 0;
const unsigned long debounce_delay = 50;

// === 부저 타이밍 ===
bool buzzer_active = false;
unsigned long buzzer_start = 0;
int buzzer_stage = 0;

// === 버튼 입력 후 딜레이 ===
bool beep_delay_active = false;
unsigned long beep_delay_start = 0;
const unsigned long beep_delay_duration = 1000;

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(WATER_SENSOR, INPUT);
  pinMode(VIB1_PIN, OUTPUT);
  pinMode(VIB2_PIN, OUTPUT);
  pinMode(VIB3_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(SIGNAL_PIN, OUTPUT);

  Serial.begin(9600);
}

void loop() {
  unsigned long now = millis();

  // === 버튼 처리 ===
  int reading = digitalRead(BUTTON_PIN);
  if (!beep_delay_active && reading != last_button_state) {
    last_debounce_time = now;
  }

  if (!beep_delay_active && (now - last_debounce_time) > debounce_delay) {
    if (reading != current_button_state) {
      current_button_state = reading;
      if (current_button_state == LOW) {
        system_on = !system_on;
        if (system_on) {
          Serial.println("1");
        } else {
          Serial.println("0");
        }
        beep_delay_active = true;
        beep_delay_start = now;
      }
    }
  }

  if (beep_delay_active && (now - beep_delay_start >= beep_delay_duration)) {
    beep_delay_active = false;
  }

  last_button_state = reading;

  // === 부저 상태 제어 ===
  handleBuzzer(now);

  // === 시스템 동작 ===
  if (system_on) {
    long distance = measureDistance();
    int waterLevel = analogRead(WATER_SENSOR);

    bool obstacleDetected = (distance > 0 && distance <= DIST_THRESHOLD);
    digitalWrite(VIB1_PIN, obstacleDetected ? HIGH : LOW);
    digitalWrite(VIB3_PIN, obstacleDetected ? HIGH : LOW);
    digitalWrite(VIB2_PIN, (waterLevel >= WATER_THRESHOLD) ? HIGH : LOW);

    int rawX = analogRead(xPin);
    int rawY = analogRead(yPin);
    int rawZ = analogRead(zPin);

    float xg = (rawX - zeroG) / sensitivity;
    float yg = (rawY - zeroG) / sensitivity;
    float zg = (rawZ - zeroG) / sensitivity;

    bool xStable = (xg >= -1.80 && xg <= -1.50);
    bool yStable = (yg >= -1.80 && yg <= -1.50);
    bool zStable = (zg >= -2.40 && zg <= -2.0);
    bool currentlyStable = (xStable && yStable && zStable);

    static unsigned long lastStableBeep = 0;
    if (currentlyStable) {
      if (now - lastStableBeep > 1000) {
        startTone(1000, 200);
        lastStableBeep = now;
      }

      if (!isStable) {
        stableStartTime = now;
        isStable = true;
        sosSent = false;
      } else if (!sosSent && now - stableStartTime >= requiredDuration) {
        Serial.println("8");
        digitalWrite(SIGNAL_PIN, HIGH);
        sosSent = true;
      }
    } else {
      isStable = false;
      sosSent = false;
      digitalWrite(SIGNAL_PIN, LOW);
    }
  } else {
    digitalWrite(VIB1_PIN, LOW);
    digitalWrite(VIB2_PIN, LOW);
    digitalWrite(VIB3_PIN, LOW);
    digitalWrite(SIGNAL_PIN, LOW);
  }
}

// === 초음파 거리 측정 ===
long measureDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return -1;
  return duration * 0.034 / 2;
}

// === 부저 관련 ===
void startTone(int freq, int duration) {
  tone(BUZZER_PIN, freq);
  buzzer_active = true;
  buzzer_start = millis();
  buzzer_stage = duration;
}

void handleBuzzer(unsigned long now) {
  if (buzzer_active && (now - buzzer_start >= buzzer_stage)) {
    noTone(BUZZER_PIN);
    buzzer_active = false;
  }
}

void startBeepSingle() {
  tone(BUZZER_PIN, 500);
  buzzer_start = millis();
  buzzer_stage = 250;
  buzzer_active = true;
}

void startBeepDouble() {
  static bool secondBeep = false;
  static unsigned long nextChange = 0;

  if (!buzzer_active) {
    tone(BUZZER_PIN, 500);
    buzzer_start = millis();
    buzzer_stage = 250;
    buzzer_active = true;
    secondBeep = false;
    nextChange = millis() + 350;
  } else if (!secondBeep && millis() >= nextChange) {
    tone(BUZZER_PIN, 1000);
    buzzer_start = millis();
    buzzer_stage = 250;
    buzzer_active = true;
    secondBeep = true;
  }
}
