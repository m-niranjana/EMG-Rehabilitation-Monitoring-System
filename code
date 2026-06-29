#include <Wire.h>
#include <U8g2lib.h>
#include <RTClib.h>
#include <SPI.h>
#include <SD.h>
#include <Preferences.h>
#include <time.h>

// ─── Blynk / WiFi ─────────────────────────────────────────────────────────────
#define BLYNK_TEMPLATE_ID   "your template id"
#define BLYNK_TEMPLATE_NAME "EMG Rehab Monitor"
#define BLYNK_AUTH_TOKEN    "your token"
#include <WiFi.h>
#include <BlynkSimpleEsp32.h>

// WiFi credentials
const char* WIFI_SSID = "your phone hotspot name";
const char* WIFI_PASS = "hotspot passwrod";

// ─── Blynk virtual pin map ────────────────────────────────────────────────────
//  V0  – Avg Activation    (%)
//  V1  – Best MVC          (RMS units)
//  V2  – Rep Count
//  V3  – Fatigue           (%)
//  V4  – Hold Time         (seconds)
//  V5  – Duration          (seconds)
//  V6  – Session date      (string DD/MM/YYYY)

#define EMG_PIN    34
#define START_BTN  27

U8G2_SH1106_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);
RTC_DS3231 rtc;
Preferences prefs;

// ─── State machine ────────────────────────────────────────────────────────────
enum State {
  WAIT_START,
  RELAX_CAL,
  WAIT_MVC,
  MVC_CAL,
  WAIT_SESSION,
  MONITORING,
  SESSION_END
};

State state = WAIT_START;

// ─── EMG metrics ──────────────────────────────────────────────────────────────
float smoothRMS      = 0;
float restRMS        = 0;
float mvcRMS         = 0;
float bestMVC        = 0;
float activation     = 0;
float fatigueIndex   = 0;
float finalActivation    = 0;
float finalMVC           = 0;
float finalFatigue       = 0;
int   finalRepCount      = 0;
int   finalStrength      = 0;
long  finalDuration      = 0;
bool  ignoreFirstPress   = false;

// ─── Rep counting ─────────────────────────────────────────────────────────────
int   repCount       = 0;
bool  repActive      = false;
float repStartThresh = 0;
float repEndThresh   = 0;

// ─── Fatigue tracking ─────────────────────────────────────────────────────────
float peakMvcPercent = 0;

// ─── Hold time ────────────────────────────────────────────────────────────────
bool          holdActive    = false;
unsigned long holdStartTime = 0;
float         holdTime      = 0;
float         finalHoldTime = 0;

// ─── Session average activation ───────────────────────────────────────────────
float activationSum      = 0;
long  activationSamples  = 0;
float finalAvgActivation = 0;

// ─── Session / display ────────────────────────────────────────────────────────
DateTime      sessionStart;
DateTime      finalDateTime;
unsigned long screenTimer  = 0;
unsigned long displayTimer = 0;
int           currentScreen = 0;
unsigned long endTimer      = 0;

// ─── Blynk session push ───────────────────────────────────────────────────────
bool blynkSent = false;

// ─── SD card ──────────────────────────────────────────────────────────────────
#define SD_SCK_PIN  18
#define SD_MISO_PIN 19
#define SD_MOSI_PIN 23
#define SD_CS_PIN   5
const char* LOG_FILE  = "/emg_log.csv";
bool sdAvailable      = false;

// ─── Recovery progress ────────────────────────────────────────────────────────
float baselineBestMVC = 0;
bool  baselineLoaded  = false;
float finalRecovery   = 0;

// ─── Long-press baseline reset ────────────────────────────────────────────────
const unsigned long BASELINE_RESET_HOLD_MS = 10000;

// ─── Forward declarations ─────────────────────────────────────────────────────
void showMessage(const String& line1, const String& line2);
void syncTimeFromWiFi();

// ─────────────────────────────────────────────────────────────────────────────
//  getRMS()
// ─────────────────────────────────────────────────────────────────────────────
float getRMS() {
  const int N = 25;
  int samples[N];
  long sum = 0;

  for (int i = 0; i < N; i++) {
    samples[i] = analogRead(EMG_PIN);
    sum += samples[i];
    delayMicroseconds(1000);
  }

  float mean  = sum / (float)N;
  float sumSq = 0;

  for (int i = 0; i < N; i++) {
    float diff = samples[i] - mean;
    sumSq += diff * diff;
  }

  return sqrt(sumSq / N);
}

// ─────────────────────────────────────────────────────────────────────────────
//  updateEMG()
// ─────────────────────────────────────────────────────────────────────────────
void updateEMG() {
  float rms = getRMS();
  smoothRMS = 0.7f * smoothRMS + 0.3f * rms;
}

// ─────────────────────────────────────────────────────────────────────────────
//  buttonPressed()
// ─────────────────────────────────────────────────────────────────────────────
bool buttonPressed() {
  static bool lastState = HIGH;
  bool currentState = digitalRead(START_BTN);

  if (lastState == HIGH && currentState == LOW) {
    delay(50);
    if (digitalRead(START_BTN) == LOW) {
      lastState = LOW;
      return true;
    }
  }

  lastState = currentState;
  return false;
}

// ─────────────────────────────────────────────────────────────────────────────
//  formatTime() / formatDate()
// ─────────────────────────────────────────────────────────────────────────────
void formatTime(DateTime t, char* buf) {
  sprintf(buf, "%02d:%02d:%02d", t.hour(), t.minute(), t.second());
}

void formatDate(DateTime t, char* buf) {
  sprintf(buf, "%02d/%02d/%04d", t.day(), t.month(), t.year());
}

// ─────────────────────────────────────────────────────────────────────────────
//  showMessage()
// ─────────────────────────────────────────────────────────────────────────────
void showMessage(const String& line1, const String& line2) {
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_ncenB08_tr);
  u8g2.drawStr(0, 20, line1.c_str());
  u8g2.drawStr(0, 45, line2.c_str());
  u8g2.sendBuffer();
}

// ─────────────────────────────────────────────────────────────────────────────
//  syncTimeFromWiFi()
//  Gets current IST time from internet and sets it into the DS3231 RTC.
//  Called once during setup() after WiFi connects.
// ─────────────────────────────────────────────────────────────────────────────
void syncTimeFromWiFi() {
  if (WiFi.status() != WL_CONNECTED) {
    showMessage("No WiFi", "Using RTC time");
    delay(1500);
    return;
  }

  showMessage("Syncing Time", "Please wait...");

  // IST = UTC + 5 hours 30 minutes = 19800 seconds offset
  configTime(19800, 0, "pool.ntp.org", "time.nist.gov");

  struct tm timeinfo;
  int retry = 0;

  // Try up to 20 times (10 seconds total)
  while (!getLocalTime(&timeinfo) && retry < 20) {
    delay(500);
    retry++;
  }

  if (getLocalTime(&timeinfo)) {
    // Successfully got time — write it into the RTC
    rtc.adjust(DateTime(
      timeinfo.tm_year + 1900,   // year  (tm_year counts from 1900)
      timeinfo.tm_mon  + 1,      // month (tm_mon is 0-indexed)
      timeinfo.tm_mday,          // day
      timeinfo.tm_hour,          // hour  (already IST due to configTime offset)
      timeinfo.tm_min,           // minute
      timeinfo.tm_sec            // second
    ));
    showMessage("Time Synced!", "RTC Updated");
    Serial.println("RTC synced from NTP (IST)");
  } else {
    showMessage("Sync Failed", "Check WiFi");
    Serial.println("NTP sync failed — RTC unchanged");
  }

  delay(1500);
}

// ─────────────────────────────────────────────────────────────────────────────
//  setup()
// ─────────────────────────────────────────────────────────────────────────────
void setup() {
  Serial.begin(115200);
  pinMode(START_BTN, INPUT_PULLUP);

  Wire.begin(21, 22);
  u8g2.begin();

  if (!rtc.begin()) {
    Serial.println("RTC not found");
    while (1);
  }

  // ── Permanent baseline storage ────────────────────────────────────────────
  prefs.begin("emgRehab", false);
  baselineLoaded = prefs.isKey("baseMVC");
  if (baselineLoaded) baselineBestMVC = prefs.getFloat("baseMVC", 0);

  // ── SD card init ──────────────────────────────────────────────────────────
  SPI.begin(SD_SCK_PIN, SD_MISO_PIN, SD_MOSI_PIN, SD_CS_PIN);
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("SD Card init failed");
    sdAvailable = false;
    showMessage("SD Card FAIL", "Logging disabled");
    delay(1200);
  } else {
    sdAvailable = true;
    if (!SD.exists(LOG_FILE)) {
      File f = SD.open(LOG_FILE, FILE_WRITE);
      if (f) {
        f.println("Date,Time,AvgActivation,BestMVC,StrengthScore,Fatigue,HoldTime,Reps,Duration,Recovery");
        f.close();
      }
    }
    showMessage("SD Card OK", baselineLoaded ? "Baseline loaded" : "First MVC = baseline");
    delay(1200);
  }

  analogReadResolution(12);
  analogSetPinAttenuation(EMG_PIN, ADC_11db);

  // ── WiFi + Blynk ──────────────────────────────────────────────────────────
  showMessage("WiFi...", WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  unsigned long wifiStart = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - wifiStart < 8000) {
    delay(300);
  }

  if (WiFi.status() == WL_CONNECTED) {
    showMessage("WiFi OK", "Connecting Blynk");
    Blynk.config(BLYNK_AUTH_TOKEN);
    Blynk.connect(4000);
    showMessage(Blynk.connected() ? "Blynk OK" : "Blynk OFFLINE", "");
    delay(800);
    syncTimeFromWiFi();   // ← sync RTC time from internet (IST)
  } else {
    showMessage("WiFi FAILED", "Offline mode");
    delay(1500);
  }
}

// ─────────────────────────────────────────────────────────────────────────────
//  calibrateRelax()
// ─────────────────────────────────────────────────────────────────────────────
void calibrateRelax() {
  float sum  = 0;
  int   count = 0;

  for (int sec = 8; sec > 0; sec--) {
    unsigned long start = millis();
    while (millis() - start < 1000) {
      updateEMG();
      sum += smoothRMS;
      count++;
    }
    showMessage("RELAX MUSCLE", String(sec) + " sec");
  }

  restRMS = sum / count;
  showMessage("REST SAVED", String(restRMS, 1));
  delay(2000);
}

// ─────────────────────────────────────────────────────────────────────────────
//  calibrateMVC()
// ─────────────────────────────────────────────────────────────────────────────
void calibrateMVC() {
  float sum  = 0;
  int   count = 0;

  for (int sec = 8; sec > 0; sec--) {
    unsigned long start = millis();
    while (millis() - start < 1000) {
      updateEMG();
      sum += smoothRMS;
      count++;
    }
    showMessage("MAX CONTRACT", String(sec) + " sec");
  }

  mvcRMS = sum / count;

  if (mvcRMS < restRMS * 1.5f) {
    showMessage("MVC TOO LOW", "TRY AGAIN");
    delay(3000);
    state = WAIT_MVC;
    return;
  }

  float calRange = mvcRMS - restRMS;
  if (calRange < 1.0f) calRange = 1.0f;
  repStartThresh = restRMS + 0.4f * calRange;
  repEndThresh   = restRMS + 0.2f * calRange;

  showMessage("MVC SAVED", String(mvcRMS, 1));
  delay(2000);
}

// ─────────────────────────────────────────────────────────────────────────────
//  updateMetrics()
// ─────────────────────────────────────────────────────────────────────────────
void updateMetrics() {
  updateEMG();

  float range = mvcRMS - restRMS;
  if (range < 1.0f) range = 1.0f;

  activation = ((smoothRMS - restRMS) / range) * 100.0f;
  activation = constrain(activation, 0, 100);

  if (activation > peakMvcPercent) {
    peakMvcPercent = activation;
  }

  if (peakMvcPercent > 0) {
    fatigueIndex = ((peakMvcPercent - activation) / peakMvcPercent) * 100.0f;
  } else {
    fatigueIndex = 0;
  }
  fatigueIndex = constrain(fatigueIndex, 0, 100);

  if (smoothRMS > bestMVC) {
    bestMVC = smoothRMS;
  }

  if (!holdActive && activation > 50) {
    holdActive    = true;
    holdStartTime = millis();
  }
  if (holdActive && activation < 50) {
    holdTime  += (millis() - holdStartTime) / 1000.0f;
    holdActive = false;
  }

  activationSum += activation;
  activationSamples++;

  if (!repActive && smoothRMS > repStartThresh) {
    repActive = true;
  }
  if (repActive && smoothRMS < repEndThresh) {
    repCount++;
    repActive = false;
  }
}

// ─────────────────────────────────────────────────────────────────────────────
//  showMonitoringScreen()
// ─────────────────────────────────────────────────────────────────────────────
void showMonitoringScreen() {

  if (millis() - screenTimer > 3000) {
    currentScreen++;
    if (currentScreen > 1)
      currentScreen = 0;
    screenTimer = millis();
  }

  DateTime now = rtc.now();

  long duration =
      now.unixtime() -
      sessionStart.unixtime();

  int mins = duration / 60;
  int secs = duration % 60;

  char buf[20];
  char clockBuf[10];
  formatTime(now, clockBuf);

  u8g2.clearBuffer();

  u8g2.setFont(u8g2_font_5x7_tr);
  u8g2.drawStr(82, 7, clockBuf);

  u8g2.setFont(u8g2_font_ncenB08_tr);

  switch(currentScreen) {

    case 0:
      u8g2.drawStr(0, 25, "ACT");
      u8g2.setCursor(70, 25);
      u8g2.print((int)activation);
      u8g2.print("%");

      u8g2.drawStr(0, 50, "REPS");
      u8g2.setCursor(70, 50);
      u8g2.print(repCount);
      break;

    case 1:
      u8g2.drawStr(0,18,"STR");
      u8g2.setCursor(70,18);
      u8g2.print((int)round((activation/100.0)*10.0));
      u8g2.print("/10");

      u8g2.drawStr(0,38,"FAT");
      u8g2.setCursor(70,38);
      u8g2.print((int)fatigueIndex);
      u8g2.print("%");

      sprintf(buf,"%02d:%02d",mins,secs);
      u8g2.drawStr(0,58,"TIME");
      u8g2.drawStr(70,58,buf);
      break;
  }

  u8g2.sendBuffer();
}

// ─────────────────────────────────────────────────────────────────────────────
//  sendBlynkSummary()
// ─────────────────────────────────────────────────────────────────────────────
void sendBlynkSummary() {
  if (!Blynk.connected()) return;

  char dateBuf[12];
  formatDate(finalDateTime, dateBuf);

  Blynk.virtualWrite(V0, (int)finalAvgActivation);
  Blynk.virtualWrite(V1, (int)bestMVC);
  Blynk.virtualWrite(V2, finalRepCount);
  Blynk.virtualWrite(V3, (int)finalFatigue);
  Blynk.virtualWrite(V4, (int)finalHoldTime);
  Blynk.virtualWrite(V5, (int)finalDuration);
  Blynk.virtualWrite(V6, String(dateBuf));

  Blynk.logEvent("session_complete",
    String("Reps:") + finalRepCount +
    " AvgAct:" + (int)finalAvgActivation + "%" +
    " Fat:" + (int)finalFatigue + "%");
}

// ─────────────────────────────────────────────────────────────────────────────
//  eraseBaseline()
// ─────────────────────────────────────────────────────────────────────────────
void eraseBaseline() {
  prefs.remove("baseMVC");
  baselineBestMVC = 0;
  baselineLoaded  = false;
}

// ─────────────────────────────────────────────────────────────────────────────
//  calculateRecoveryProgress()
// ─────────────────────────────────────────────────────────────────────────────
void calculateRecoveryProgress() {
  if (!baselineLoaded) {
    baselineBestMVC = bestMVC;
    baselineLoaded  = true;
    prefs.putFloat("baseMVC", baselineBestMVC);
  }

  float baseline = baselineBestMVC;
  if (baseline < 1.0f) baseline = 1.0f;

  finalRecovery = (bestMVC / baseline) * 100.0f;
  if (finalRecovery < 0) finalRecovery = 0;
}

// ─────────────────────────────────────────────────────────────────────────────
//  logSessionToSD()
// ─────────────────────────────────────────────────────────────────────────────
void logSessionToSD() {
  if (!sdAvailable) return;

  File f = SD.open(LOG_FILE, FILE_APPEND);
  if (!f) return;

  char dateBuf[12], timeBuf[10];
  formatDate(finalDateTime, dateBuf);
  formatTime(finalDateTime, timeBuf);

  f.print(dateBuf);                 f.print(",");
  f.print(timeBuf);                 f.print(",");
  f.print((int)finalAvgActivation); f.print(",");
  f.print((int)bestMVC);            f.print(",");
  f.print(finalStrength);           f.print(",");
  f.print((int)finalFatigue);       f.print(",");
  f.print(finalHoldTime, 1);        f.print(",");
  f.print(finalRepCount);           f.print(",");
  f.print(finalDuration);           f.print(",");
  f.println((int)finalRecovery);

  f.close();
}

// ─────────────────────────────────────────────────────────────────────────────
//  showSessionEnd()
// ─────────────────────────────────────────────────────────────────────────────
void showSessionEnd() {

  if (millis() - endTimer >= 4000) {
    currentScreen = (currentScreen + 1) % 4;
    endTimer = millis();
  }

  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_ncenB08_tr);

  if (currentScreen == 0) {

    u8g2.drawStr(0, 12, "SESSION END");

    u8g2.setCursor(0, 30);
    u8g2.print("Activ : ");
    u8g2.print((int)finalActivation);
    u8g2.print("%");

    u8g2.setCursor(0, 46);
    u8g2.print("MVC   : ");
    u8g2.print((int)finalMVC);
    u8g2.print("%");

    u8g2.setCursor(0, 62);
    u8g2.print("Reps  : ");
    u8g2.print(finalRepCount);

  } else if (currentScreen == 1) {

    int  mins = finalDuration / 60;
    int  secs = finalDuration % 60;
    char buf[8];
    sprintf(buf, "%02d:%02d", mins, secs);

    u8g2.drawStr(0, 12, "SESSION END");

    u8g2.setCursor(0, 30);
    u8g2.print("Str    : ");
    u8g2.print(finalStrength);
    u8g2.print("/10");

    u8g2.setCursor(0, 46);
    u8g2.print("Fatg   : ");
    u8g2.print((int)finalFatigue);
    u8g2.print("%");

    u8g2.setCursor(0, 62);
    u8g2.print("Time   : ");
    u8g2.print(buf);

  } else if (currentScreen == 2) {

    char dateBuf[12];
    formatDate(finalDateTime, dateBuf);

    u8g2.drawStr(0, 9, "SUMMARY");

    u8g2.setCursor(0, 21);
    u8g2.print("Date  : ");
    u8g2.print(dateBuf);

    u8g2.setCursor(0, 33);
    u8g2.print("AvgAct: ");
    u8g2.print((int)finalAvgActivation);
    u8g2.print("%");

    u8g2.setCursor(0, 45);
    u8g2.print("BestMVC:");
    u8g2.print((int)bestMVC);

    u8g2.setCursor(0, 57);
    u8g2.print("HoldT : ");
    u8g2.print(finalHoldTime, 1);
    u8g2.print("s");

  } else {

    u8g2.drawStr(0, 9, "RECOVERY");

    int barFill = constrain((int)finalRecovery, 0, 100);
    u8g2.drawFrame(0, 14, 102, 10);
    u8g2.drawBox(1, 15, barFill, 8);

    u8g2.setCursor(0, 36);
    u8g2.print("Recovery: ");
    u8g2.print(finalRecovery, 1);
    u8g2.print("%");

    u8g2.setCursor(0, 48);
    u8g2.print("Now MVC : ");
    u8g2.print((int)bestMVC);

    u8g2.setCursor(0, 60);
    u8g2.print("Base MVC: ");
    u8g2.print((int)baselineBestMVC);
  }

  u8g2.sendBuffer();
}

// ─────────────────────────────────────────────────────────────────────────────
//  loop()
// ─────────────────────────────────────────────────────────────────────────────
void loop() {
  switch (state) {

    case WAIT_START: {
      static unsigned long pressStart    = 0;
      static bool          isDown        = false;
      static bool          longPressDone = false;

      bool rawLow = (digitalRead(START_BTN) == LOW);

      if (rawLow && !isDown) {
        delay(50);
        if (digitalRead(START_BTN) == LOW) {
          isDown        = true;
          pressStart    = millis();
          longPressDone = false;
        }
      }

      if (isDown && rawLow && !longPressDone) {
        unsigned long held = millis() - pressStart;
        if (held >= BASELINE_RESET_HOLD_MS) {
          longPressDone = true;
          eraseBaseline();
          showMessage("BASELINE RESET", "Next = New Base");
          delay(1500);
        } else if (held >= 1000) {
          int secsLeft = 10 - (int)(held / 1000);
          showMessage("Hold to Reset", String(secsLeft) + "s left");
        } else {
          showMessage("EMG REHAB", "Press START");
        }
      }

      if (!rawLow && isDown) {
        unsigned long held = millis() - pressStart;
        isDown = false;
        if (!longPressDone && held < BASELINE_RESET_HOLD_MS) {
          if (ignoreFirstPress) {
            ignoreFirstPress = false;   // discard carry-over press from SESSION_END
          } else {
            state = RELAX_CAL;          // real press — start new session
          }
        }
        longPressDone = false;
      }

      if (!isDown && state == WAIT_START) {
        showMessage("EMG REHAB", "Press START");
      }

      break;
    }

    case RELAX_CAL:
      calibrateRelax();
      state = WAIT_MVC;
      break;

    case WAIT_MVC: {
      bool pressed = buttonPressed();
      showMessage("Ready for MVC", "Press START");
      if (pressed) state = MVC_CAL;
      break;
    }

    case MVC_CAL:
      calibrateMVC();
      state = WAIT_SESSION;
      break;

    case WAIT_SESSION: {
      bool pressed = buttonPressed();
      showMessage("MVC SAVED", "Press START");
      if (pressed) {
        sessionStart      = rtc.now();
        screenTimer       = millis();
        currentScreen     = 0;
        peakMvcPercent    = 0;
        repCount          = 0;
        holdActive        = false;
        holdTime          = 0;
        activationSum     = 0;
        activationSamples = 0;
        state             = MONITORING;
      }
      break;
    }

    case MONITORING:

      updateMetrics();

      Blynk.run();

      if (millis() - displayTimer >= 100) {
        displayTimer = millis();
        showMonitoringScreen();
      }

      if (buttonPressed()) {

        DateTime now = rtc.now();

        finalActivation    = activation;
        finalMVC           = activation;
        finalFatigue       = fatigueIndex;
        finalRepCount      = repCount;
        finalStrength      = round((activation / 100.0) * 10.0);
        finalHoldTime      = holdTime + (holdActive ? (millis() - holdStartTime) / 1000.0f : 0);
        finalAvgActivation = (activationSamples > 0)
                             ? activationSum / activationSamples
                             : 0;

        finalDuration =
          now.unixtime() -
          sessionStart.unixtime();

        finalDateTime = now;

        calculateRecoveryProgress();
        logSessionToSD();

        endTimer      = millis();
        currentScreen = 0;
        blynkSent     = false;
        state         = SESSION_END;
      }

      break;

    case SESSION_END:

      Blynk.run();

      if (!blynkSent) {
        sendBlynkSummary();
        blynkSent = true;
      }

      if (buttonPressed()) {
        smoothRMS         = 0;
        restRMS           = 0;
        mvcRMS            = 0;
        bestMVC           = 0;
        activation        = 0;
        fatigueIndex      = 0;
        peakMvcPercent    = 0;
        repCount          = 0;
        repActive         = false;
        repStartThresh    = 0;
        repEndThresh      = 0;
        holdActive        = false;
        holdTime          = 0;
        activationSum     = 0;
        activationSamples = 0;
        currentScreen     = 0;
        blynkSent         = false;
        ignoreFirstPress  = true;   // discard the press that exits SESSION_END
        state             = WAIT_START;
        break;
      }

      showSessionEnd();

      break;
  }
}
