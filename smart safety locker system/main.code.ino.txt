/*
 ============================================================
  ANTI-THEFT LOCKER SYSTEM - ESP32
  Local WiFi HTTP Server (No Firebase)
  Telegram Notifications
 ============================================================
*/

#include <ArduinoJson.h>
#include <ESP32Servo.h>
#include <HTTPClient.h>
#include <Keypad.h>
#include <LiquidCrystal_I2C.h>
#include <Preferences.h>
#include <WebServer.h>
#include <WiFi.h>
#include <Wire.h>
#include <time.h>

// ===================== CONFIGURATION =====================
const char *WIFI_SSID     = "hello123";
const char *WIFI_PASSWORD = "poco1234";

// ── Security (default; overwritten from flash on boot) ────
String CORRECT_PASSWORD = "2580";

// ── Telegram (set via POST /config or web dashboard) ──────
String TELEGRAM_TOKEN   = "8510354203:AAGs0hyHpmpxIRAoA1gjj6-YbP8vMG_N_8c";
String TELEGRAM_CHAT_ID = "5524391658";

// ── Servo positions ───────────────────────────────────────
const int SERVO1_LOCKED    = 0;
const int SERVO1_UNLOCKED  = 90;
const int SERVO2_CLOSED    = 143;
const int SERVO2_OPEN      = 45;
const int TRAPDOOR_HOLD_MS = 2000;

// ── Timing ────────────────────────────────────────────────
const int VIBRATION_COOLDOWN = 10000;
const int STARTUP_GRACE_MS   = 8000;
const int TELEGRAM_TIMEOUT   = 3000;

// ── Pin definitions ───────────────────────────────────────
#define SERVO1_PIN    18
#define SERVO2_PIN    19
#define BUZZER_PIN    23
#define VIBRATION_PIN  5

// ===================== KEYPAD SETUP ========================
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {{'1','2','3','A'},
                          {'4','5','6','B'},
                          {'7','8','9','C'},
                          {'*','0','#','D'}};
byte rowPins[ROWS] = {13, 12, 14, 27};
byte colPins[COLS] = {26, 25, 33, 32};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// ===================== OBJECTS =============================
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo servo1;
Servo servo2;
WebServer server(80);
Preferences prefs;

// ===================== STATE VARIABLES =====================
int  failedAttempts    = 0;
bool lockerOpen        = false;
bool alertTriggered    = false;
bool vibAlertTriggered = false;
bool ntpSynced         = false;
bool vibSensorEnabled  = false;   // set true when physical SW-420 is connected & working
unsigned long vibLastTrigger  = 0;
unsigned long systemStartTime = 0;
unsigned long lastNtpRetry    = 0;
String enteredPassword = "";
String localIP         = "";

// ── Telegram single-slot queue (non-blocking) ─────────────
String              telegramQueue = "";
SemaphoreHandle_t   tgramMutex    = NULL;

// ===================== LOG RING BUFFER =====================
#define LOG_MAX 30
struct LogEntry {
  String type;
  String message;
  String timestamp;
  unsigned long ms;
};
LogEntry logBuffer[LOG_MAX];
int logHead  = 0;
int logCount = 0;

// Forward declare so addLog can call getCurrentTime
String getCurrentTime();

void addLog(const String &type, const String &message) {
  logBuffer[logHead] = { type, message, getCurrentTime(), millis() };
  logHead = (logHead + 1) % LOG_MAX;
  if (logCount < LOG_MAX) logCount++;
  Serial.printf("[LOG][%s] %s\n", type.c_str(), message.c_str());
}

// ===================== FORWARD DECLARES ====================
void showIdleScreen();
void flipTrapdoor();
void resetAlert();
void handleWebCommand(String text);
void processKey(char key);

// ===================== HELPERS =============================
void drainKeypad() {
  char k = keypad.getKey();
  if (k) processKey(k);
}

void waitWithKeypad(unsigned long ms) {
  unsigned long start = millis();
  while (millis() - start < ms) {
    drainKeypad();
    server.handleClient();
    delay(1);
  }
}

void queueTelegram(const String &msg) {
  if (tgramMutex && xSemaphoreTake(tgramMutex, pdMS_TO_TICKS(50)) == pdTRUE) {
    if (telegramQueue.isEmpty()) telegramQueue = msg;
    xSemaphoreGive(tgramMutex);
  }
}

// ===================== CORS HELPER =========================
void addCORSHeaders() {
  server.sendHeader("Access-Control-Allow-Origin",  "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
  server.sendHeader("Access-Control-Allow-Headers", "Content-Type");
}

// ===================== LOAD / SAVE CONFIG ==================
void loadConfig() {
  prefs.begin("locker", true);
  String p = prefs.getString("password", "");
  if (p.length() >= 4) { CORRECT_PASSWORD = p; Serial.println("[CFG] PIN loaded from flash."); }
  String tok = prefs.getString("tgToken",  "");
  String cid = prefs.getString("tgChatId", "");
  if (tok.length() > 10) { TELEGRAM_TOKEN   = tok; Serial.println("[CFG] Telegram token loaded."); }
  if (cid.length() >  0) { TELEGRAM_CHAT_ID = cid; Serial.println("[CFG] Telegram chatId loaded."); }
  vibSensorEnabled = prefs.getBool("vibEnabled", false);  // default OFF (safe for broken/missing sensor)
  Serial.println("[CFG] Vib sensor HW: " + String(vibSensorEnabled ? "ENABLED" : "DISABLED"));
  prefs.end();
}

void saveConfig() {
  prefs.begin("locker", false);
  prefs.putString("password",  CORRECT_PASSWORD);
  prefs.putString("tgToken",   TELEGRAM_TOKEN);
  prefs.putString("tgChatId",  TELEGRAM_CHAT_ID);
  prefs.putBool("vibEnabled",  vibSensorEnabled);
  prefs.end();
  Serial.println("[CFG] Config saved to flash.");
}

// ===================== NTP =================================
void syncTimeQuick() {
  configTime(19800, 0, "time.google.com", "pool.ntp.org", "time.cloudflare.com");
  struct tm t;
  int retry = 0;
  while (!getLocalTime(&t) && retry++ < 3) { delay(1000); Serial.print("."); }
  if (getLocalTime(&t)) {
    char buf[30]; strftime(buf, sizeof(buf), "%d/%m/%Y %H:%M:%S", &t);
    Serial.println("\nNTP synced: " + String(buf));
    ntpSynced = true;
  } else {
    Serial.println("\nNTP pending — will retry in background.");
  }
}

void retryNtpBackground() {
  struct tm t;
  if (getLocalTime(&t)) {
    char buf[30]; strftime(buf, sizeof(buf), "%d/%m/%Y %H:%M:%S", &t);
    Serial.println("[NTP] Background sync OK: " + String(buf));
    ntpSynced = true;
  }
}

String getCurrentTime() {
  struct tm t;
  if (!getLocalTime(&t)) return "Syncing...";
  char buf[30];
  strftime(buf, sizeof(buf), "%d/%m/%Y %H:%M:%S", &t);
  return String(buf);
}

// ===================== TELEGRAM (FreeRTOS Core 0) ==========
// Runs on Core 0 — never blocks the main loop (Core 1).
void telegramTask(void *param) {
  for (;;) {
    String msg = "";
    String tok = "";
    String cid = "";

    // Snapshot queue + creds under mutex
    if (xSemaphoreTake(tgramMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
      msg = telegramQueue;
      tok = TELEGRAM_TOKEN;
      cid = TELEGRAM_CHAT_ID;
      xSemaphoreGive(tgramMutex);
    }

    if (!msg.isEmpty() && !tok.isEmpty() && WiFi.status() == WL_CONNECTED) {
      HTTPClient http;
      http.begin("https://api.telegram.org/bot" + tok + "/sendMessage");
      http.setTimeout(TELEGRAM_TIMEOUT);
      http.addHeader("Content-Type", "application/json");

      DynamicJsonDocument doc(512);
      doc["chat_id"]    = cid;
      doc["text"]       = msg;
      doc["parse_mode"] = "HTML";
      String body; serializeJson(doc, body);

      int code = http.POST(body);
      http.end();

      if (code >= 200 && code < 300) {
        Serial.printf("[TGRAM] OK (HTTP %d)\n", code);
        // Clear only if queue hasn't been updated while we were sending
        if (xSemaphoreTake(tgramMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
          if (telegramQueue == msg) telegramQueue = "";
          xSemaphoreGive(tgramMutex);
        }
      } else {
        Serial.printf("[TGRAM] FAILED (HTTP %d) — retry in 10 s\n", code);
        vTaskDelay(pdMS_TO_TICKS(10000)); // back-off before retry
      }
    }

    vTaskDelay(pdMS_TO_TICKS(300)); // poll queue every 300 ms
  }
}

// ===================== WEB SERVER HANDLERS =================

// GET /status
void handleStatus() {
  addCORSHeaders();
  DynamicJsonDocument doc(512);
  doc["isLocked"]                = !lockerOpen;
  doc["isSecretCompartmentOpen"] = (servo2.read() == SERVO2_OPEN);
  doc["failedAttempts"]          = failedAttempts;
  doc["buzzerOn"]                = (digitalRead(BUZZER_PIN) == HIGH);
  doc["isBreached"]              = alertTriggered;
  doc["vibrationDetected"]       = vibAlertTriggered;
  doc["vibSensorEnabled"]        = vibSensorEnabled;
  doc["ntpSynced"]               = ntpSynced;
  doc["lastSeen"]                = getCurrentTime();
  doc["uptimeMs"]                = (long)millis();
  doc["ip"]                      = localIP;
  String body; serializeJson(doc, body);
  server.send(200, "application/json", body);
}

// GET /logs
void handleLogs() {
  addCORSHeaders();
  DynamicJsonDocument doc(4096);
  JsonArray arr = doc.createNestedArray("logs");
  int start = (logCount < LOG_MAX) ? 0 : logHead;
  for (int i = 0; i < logCount; i++) {
    int idx = (start + i) % LOG_MAX;
    JsonObject e = arr.createNestedObject();
    e["type"]      = logBuffer[idx].type;
    e["message"]   = logBuffer[idx].message;
    e["timestamp"] = logBuffer[idx].timestamp;
    e["ms"]        = logBuffer[idx].ms;
  }
  String body; serializeJson(doc, body);
  server.send(200, "application/json", body);
}

// POST /command  { "cmd": "/lock" }
void handleCommandHTTP() {
  addCORSHeaders();
  if (!server.hasArg("plain")) { server.send(400, "application/json", "{\"error\":\"No body\"}"); return; }
  DynamicJsonDocument doc(256);
  if (deserializeJson(doc, server.arg("plain"))) { server.send(400, "application/json", "{\"error\":\"Bad JSON\"}"); return; }
  String cmd = doc["cmd"].as<String>();
  server.send(200, "application/json", "{\"ok\":true}");
  handleWebCommand(cmd);
}

// POST /config  { "password":"XXXX", "tgToken":"...", "tgChatId":"..." }
void handleConfigHTTP() {
  addCORSHeaders();
  if (!server.hasArg("plain")) { server.send(400, "application/json", "{\"error\":\"No body\"}"); return; }
  DynamicJsonDocument doc(512);
  if (deserializeJson(doc, server.arg("plain"))) { server.send(400, "application/json", "{\"error\":\"Bad JSON\"}"); return; }

  bool changed = false;
  if (doc.containsKey("password")) {
    String p = doc["password"].as<String>();
    if (p.length() >= 4) { CORRECT_PASSWORD = p; changed = true; }
  }
  if (doc.containsKey("tgToken")) {
    String t = doc["tgToken"].as<String>();
    if (t.length() > 10) { TELEGRAM_TOKEN = t; changed = true; }
  }
  if (doc.containsKey("tgChatId")) {
    String c = doc["tgChatId"].as<String>();
    if (c.length() > 0) { TELEGRAM_CHAT_ID = c; changed = true; }
  }
  if (doc.containsKey("vibEnabled")) {
    vibSensorEnabled = doc["vibEnabled"].as<bool>();
    changed = true;
    Serial.println("[CFG] Vib sensor HW set to: " + String(vibSensorEnabled ? "ENABLED" : "DISABLED"));
    addLog("info", vibSensorEnabled ? "HW vibration sensor ENABLED" : "HW vibration sensor DISABLED");
  }
  if (changed) saveConfig();
  server.send(200, "application/json", "{\"ok\":true}");
  addLog("info", "Config updated via web");
}

// OPTIONS preflight (CORS)
void handleOptions() {
  addCORSHeaders();
  server.send(204);
}

// ===================== SYSTEM SCREENS ======================
void showIdleScreen() {
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("  ANTI-THEFT    ");
  lcd.setCursor(0, 1); lcd.print("  Enter Pass: * ");
}

// ===================== TRAPDOOR ============================
void flipTrapdoor() {
  servo2.write(SERVO2_OPEN);
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(" Securing Items ");
  lcd.setCursor(0, 1); lcd.print(" Chamber Active ");
  waitWithKeypad(TRAPDOOR_HOLD_MS);
  servo2.write(SERVO2_CLOSED);
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(" Items Secured! ");
  lcd.setCursor(0, 1); lcd.print(" Chamber Sealed ");
  waitWithKeypad(1500);
}

// ===================== LOCKER OPEN =========================
void openLocker() {
  lockerOpen = true;
  servo1.write(SERVO1_UNLOCKED);
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("  ACCESS GRANTED");
  lcd.setCursor(0, 1); lcd.print("  Locker Opened!");
  Serial.println("[EVENT] Door opened via keypad");
  addLog("success", "Access granted — door opened via keypad");
  waitWithKeypad(1200);

  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(" Door is Open   ");
  lcd.setCursor(0, 1); lcd.print(" Press A to Lock");

  // Wait for A (physical) or remote /lock (web)
  unsigned long lastSrv = millis();
  bool remotelyLocked = false;
  while (true) {
    char k = keypad.getKey();
    if (k == 'A') break;
    if (millis() - lastSrv > 50) {
      lastSrv = millis();
      server.handleClient();
      if (!lockerOpen) { remotelyLocked = true; break; }
    }
    delay(1);
  }

  if (!remotelyLocked) {
    servo1.write(SERVO1_LOCKED);
    lockerOpen = false;
    failedAttempts = 0;
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print("  Locker Locked ");
    lcd.setCursor(0, 1); lcd.print("                ");
    waitWithKeypad(1200);
    showIdleScreen();
    Serial.println("[EVENT] Door locked by A key");
    addLog("info", "Door locked via keypad (A)");
  } else {
    failedAttempts = 0;
    Serial.println("[EVENT] Door locked remotely while open");
  }
}

// ===================== SECURITY ALERT ======================
void triggerSecurityAlert(String reason) {
  if (alertTriggered) return;
  alertTriggered = true;
  digitalWrite(BUZZER_PIN, HIGH);
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(" !! ALERT !!    ");
  lcd.setCursor(0, 1); lcd.print(reason.substring(0, 16));
  Serial.println("[ALERT] " + reason + " — Trapdoor deploying. Buzzer latched ON.");

  String tgramMsg;
  if (reason.indexOf("Wrong Pass") >= 0 || reason.indexOf("Wrong PIN") >= 0) {
    tgramMsg = "🔐 <b>SECURITY BREACH — Wrong PIN</b>\n"
               "3 consecutive failed unlock attempts detected.\n"
               "⏰ Buzzer active | Trapdoor deployed\n"
               "• Enter correct PIN on keypad, OR\n"
               "• Reset via web dashboard.";
  } else if (reason.indexOf("Tamper") >= 0 || reason.indexOf("Vibration") >= 0) {
    tgramMsg = "⚡ <b>SECURITY BREACH — Physical Tamper</b>\n"
               "Vibration sensor (SW-420) triggered!\n"
               "⏰ Buzzer active | Trapdoor deployed\n"
               "• Enter correct PIN on keypad, OR\n"
               "• Reset via web dashboard.";
  } else {
    tgramMsg = "🚨 <b>SECURITY BREACH</b>\nReason: " + reason + "\n"
               "⏰ Buzzer active | Trapdoor deployed\nReset via web or correct PIN.";
  }
  queueTelegram(tgramMsg);
  addLog("critical", "ALERT: " + reason + " — buzzer latched, reset required");
  flipTrapdoor();
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(" !! ALERT !!    ");
  lcd.setCursor(0, 1); lcd.print(reason.substring(0, 16));
}

// ===================== RESET ALERT =========================
void resetAlert() {
  alertTriggered    = false;
  vibAlertTriggered = false;
  failedAttempts    = 0;
  vibLastTrigger    = millis();   // enforce cooldown — sensor may still be vibrating
  servo2.write(SERVO2_CLOSED);
  digitalWrite(BUZZER_PIN, LOW);
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(" System Reset   ");
  waitWithKeypad(1500);
  showIdleScreen();
  Serial.println("[EVENT] System RESET — all alerts cleared");
  queueTelegram("✅ <b>Alert Cleared</b>\nLocker reset. System secure.");
  addLog("info", "System reset — all alerts cleared");
}

// ===================== HANDLE WEB COMMAND ==================
void handleWebCommand(String text) {
  text.trim();
  String action;

  if (text == "/unlock") {
    servo1.write(SERVO1_UNLOCKED);
    lockerOpen = true;
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(" Remote UNLOCK  ");
    lcd.setCursor(0, 1); lcd.print(" Door Open!     ");
    action = "Door UNLOCKED via web";

  } else if (text == "/lock") {
    servo1.write(SERVO1_LOCKED);
    lockerOpen = false;
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(" Remote LOCK    ");
    lcd.setCursor(0, 1); lcd.print(" Door Locked!   ");
    waitWithKeypad(1500);
    showIdleScreen();
    action = "Door LOCKED via web";

  } else if (text == "/trapdoor_open") {
    servo2.write(SERVO2_OPEN);
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(" Trapdoor OPEN  ");
    lcd.setCursor(0, 1); lcd.print(" Manual Control ");
    action = "Trapdoor OPENED manually";

  } else if (text == "/trapdoor_close") {
    servo2.write(SERVO2_CLOSED);
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(" Trapdoor CLOSED");
    lcd.setCursor(0, 1); lcd.print(" Manual Control ");
    waitWithKeypad(1500);
    showIdleScreen();
    action = "Trapdoor CLOSED manually";

  } else if (text == "/trapdoor_flip") {
    flipTrapdoor();
    showIdleScreen();
    action = "Trapdoor flipped and SEALED";

  } else if (text == "/buzzer_on") {
    alertTriggered = true;
    digitalWrite(BUZZER_PIN, HIGH);
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(" BUZZER ON      ");
    lcd.setCursor(0, 1); lcd.print(" ALERT ACTIVE   ");
    queueTelegram("🌐 <b>REMOTE ALERT — Web Trigger</b>\n"
                  "Alarm activated manually via web dashboard.\n"
                  "⏰ Buzzer latched ON\n"
                  "• Stop via web dashboard Reset button, OR\n"
                  "• Enter correct PIN on keypad.");
    action = "Buzzer LATCHED ON — web alert active";

  } else if (text == "/buzzer_off") {
    digitalWrite(BUZZER_PIN, LOW);
    showIdleScreen();
    action = "Buzzer turned OFF";

  } else if (text == "/reset") {
    resetAlert();
    return;

  } else if (text == "/sim_vib") {
    // Virtual vibration trigger — identical to a real SW-420 event
    unsigned long now = millis();
    if (now - systemStartTime > STARTUP_GRACE_MS && !alertTriggered) {
      vibAlertTriggered = true;
      vibLastTrigger = now;
      triggerSecurityAlert("Tamper Detected!");  // flips trapdoor, latches buzzer, sends Telegram
      action = "VIBRATION SIMULATED — full security alert triggered";
    } else {
      action = alertTriggered ? "Sim vib ignored (alert already active)" : "Sim vib ignored (startup grace)";
    }

  } else if (text == "/status") {
    action = "Status refreshed at " + getCurrentTime();

  } else {
    action = "Unknown command: " + text;
  }

  Serial.println("[WEB] " + action);
  addLog("info", action);
}

// ===================== PROCESS KEYPAD KEY ==================
void processKey(char key) {
  Serial.print("[KEY] "); Serial.println(key);

  if (key == '*') {
    enteredPassword = "";
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print("Enter Password: ");
    lcd.setCursor(0, 1);
    return;
  }

  if (key == 'B') {
    if (alertTriggered || vibAlertTriggered) resetAlert();
    enteredPassword = "";
    return;
  }

  if (key == '#') {
    lcd.clear(); lcd.setCursor(0, 0);
    if (enteredPassword == CORRECT_PASSWORD) {
      failedAttempts    = 0;
      vibAlertTriggered = false;
      if (alertTriggered) resetAlert();
      openLocker();
    } else {
      failedAttempts++;
      lcd.print("Wrong Password! ");
      lcd.setCursor(0, 1);
      lcd.print("Attempt "); lcd.print(failedAttempts); lcd.print("/3      ");
      Serial.println("[KEY] Wrong PIN — attempt " + String(failedAttempts) + "/3");
      addLog("warning", "Wrong PIN — attempt " + String(failedAttempts) + "/3");
      waitWithKeypad(500);
      if (failedAttempts >= 3) {
        triggerSecurityAlert("3x Wrong Pass!");
        failedAttempts = 0;
      } else {
        lcd.clear();
        lcd.setCursor(0, 0); lcd.print("Enter Password: ");
        lcd.setCursor(0, 1);
      }
    }
    enteredPassword = "";
    return;
  }

  if (key == 'D') {
    if (enteredPassword.length() > 0)
      enteredPassword.remove(enteredPassword.length() - 1);
  } else if (key != 'A' && key != 'C') {
    enteredPassword += key;
  }

  lcd.setCursor(0, 1);
  String masked = "";
  for (unsigned int i = 0; i < enteredPassword.length(); i++) masked += '*';
  while (masked.length() < 16) masked += ' ';
  lcd.print(masked);
}

// ===================== SETUP ===============================
void setup() {
  Serial.begin(115200);
  Serial.println("\n[BOOT] Anti-Theft Locker System starting...");

  lcd.init(); lcd.backlight();
  lcd.setCursor(0, 0); lcd.print("  ANTI-THEFT    ");
  lcd.setCursor(0, 1); lcd.print("  LOCKER SYSTEM ");
  delay(1500);

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(VIBRATION_PIN, INPUT_PULLDOWN);
  digitalWrite(BUZZER_PIN, LOW);
  servo1.attach(SERVO1_PIN); servo1.write(SERVO1_LOCKED);
  servo2.attach(SERVO2_PIN); servo2.write(SERVO2_CLOSED);

  loadConfig();

  // Start Telegram task on Core 0 (main loop runs on Core 1)
  tgramMutex = xSemaphoreCreateMutex();
  xTaskCreatePinnedToCore(telegramTask, "TelegramTask", 8192, NULL, 1, NULL, 0);
  Serial.println("[BOOT] Telegram task started on Core 0.");

  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("Connecting WiFi ");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  int timeout = 0;
  while (WiFi.status() != WL_CONNECTED && timeout++ < 20) {
    delay(500); Serial.print(".");
  }

  if (WiFi.status() == WL_CONNECTED) {
    localIP = WiFi.localIP().toString();
    lcd.setCursor(0, 1); lcd.print("WiFi Connected! ");
    Serial.println("\n[WiFi] IP: " + localIP);

    // Register routes
    server.on("/status",  HTTP_GET,     handleStatus);
    server.on("/logs",    HTTP_GET,     handleLogs);
    server.on("/command", HTTP_POST,    handleCommandHTTP);
    server.on("/config",  HTTP_POST,    handleConfigHTTP);
    // CORS preflight
    server.on("/status",  HTTP_OPTIONS, handleOptions);
    server.on("/command", HTTP_OPTIONS, handleOptions);
    server.on("/config",  HTTP_OPTIONS, handleOptions);
    server.on("/logs",    HTTP_OPTIONS, handleOptions);
    server.begin();
    Serial.println("[WEB] Server started → http://" + localIP);

    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(" Syncing Time.. ");
    syncTimeQuick();

    addLog("info", "System initialized — ESP32 online at " + localIP);
    queueTelegram("🟢 <b>Locker Online</b>\nLocal API: http://" + localIP + "\nAccess via same WiFi network.");

  } else {
    lcd.setCursor(0, 1); lcd.print("WiFi FAILED!    ");
    Serial.println("\n[WiFi] Connection failed — running offline");
    addLog("warning", "WiFi failed — running offline");
  }

  delay(1000);
  systemStartTime = millis();

  // Show IP briefly on LCD
  if (localIP.length() > 0) {
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print("IP:" + localIP);
    lcd.setCursor(0, 1); lcd.print("Port 80 / Local ");
    delay(3000);
  }
  showIdleScreen();
  Serial.println("[BOOT] Ready. API → http://" + localIP);
}

// ===================== MAIN LOOP ===========================
void loop() {
  server.handleClient();
  drainKeypad();

  // Vibration sensor — only when hardware is enabled
  if (vibSensorEnabled && millis() - systemStartTime > STARTUP_GRACE_MS) {
    if (digitalRead(VIBRATION_PIN) == HIGH) {
      int highCount = 0;
      for (int i = 0; i < 5; i++) {
        waitWithKeypad(20);
        if (digitalRead(VIBRATION_PIN) == HIGH) highCount++;
      }
      if (highCount == 5) {
        unsigned long now = millis();
        if (!vibAlertTriggered || (now - vibLastTrigger > VIBRATION_COOLDOWN)) {
          vibAlertTriggered = true;
          vibLastTrigger = now;
          Serial.println("[SENSOR] High-impact vibration detected!");
          triggerSecurityAlert("Tamper Detected!");
        }
      }
    }
  }

  // NTP background retry (until synced)
  if (!ntpSynced && millis() - lastNtpRetry > 30000) {
    lastNtpRetry = millis();
    retryNtpBackground();
  }

  // Telegram is handled by telegramTask() on Core 0 — nothing to do here.
}