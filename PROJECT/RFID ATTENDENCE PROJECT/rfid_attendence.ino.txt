//Ardiuno RFID main code


/*
  ESP8266 RFID Simple Logger
  - Master card to toggle ADD MODE (register users via Serial)
  - FIFO overwrite when storage full (like CCTV)
  - NO IN/OUT tracking; every tap logs current time as a new row
  - Persist users + write pointer in EEPROM
  - Google Sheet update via HTTP GET (HTTPS with insecure client)
*/

#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <WiFiClientSecureBearSSL.h>
#include <SPI.h>
#include <MFRC522.h>
#include <EEPROM.h>
#include <WiFiUdp.h>
#include <NTPClient.h>
#include <time.h>

// ---------- CONFIG ----------
const char* WIFI_SSID     = "black 123";
const char* WIFI_PASSWORD = "9310615305";
String GOOGLE_SCRIPT_URL  = "https://script.google.com/macros/s/AKfycbyfoHHhKz7LCi01kGTmnJ7R0yX9SrOsgwVB2fNiZK95YOh9B70oLAZgy50rVa8JkZrx/exec";

#define RST_PIN     D3
#define SS_PIN      D4
#define BUZZER_PIN  D0

// EEPROM + storage
#define EEPROM_SIZE 4096
#define MAX_USERS   50
#define UID_LENGTH  8
#define NAME_LENGTH 20

// ---------- RFID ----------
MFRC522 mfrc522(SS_PIN, RST_PIN);

// ---------- Time (NTP) ----------
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", 19800); // IST +5:30

// ---------- Data Structures ----------
struct User {
  char uid[UID_LENGTH + 1];   // hex uid string (8 chars) + null
  char name[NAME_LENGTH];
};

struct DataStore {
  User users[MAX_USERS];
  uint16_t nextWriteIndex; // pointer for FIFO overwrite (0..MAX_USERS-1)
};

DataStore store;

// Master UID (uppercase hex), change to your master card UID
String MASTER_UID = "93B53436";
bool addMode = false;

// ---------- Helpers ----------
void beep(int duration) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(duration);
  digitalWrite(BUZZER_PIN, LOW);
}

void saveStoreToEEPROM() {
  EEPROM.put(0, store);
  EEPROM.commit();
}

void loadStoreFromEEPROM() {
  EEPROM.get(0, store);
  if (store.nextWriteIndex >= MAX_USERS) {
    store.nextWriteIndex = 0;
  }
  // अगर EEPROM खाली था, तो ensure करें कि UID strings null-terminated हों
  for (int i = 0; i < MAX_USERS; i++) {
    store.users[i].uid[UID_LENGTH] = '\0';
    store.users[i].name[NAME_LENGTH - 1] = '\0';
  }
}

void clearEEPROM() {
  for (int i = 0; i < MAX_USERS; i++) {
    memset(store.users[i].uid, 0, sizeof(store.users[i].uid));
    memset(store.users[i].name, 0, sizeof(store.users[i].name));
  }
  store.nextWriteIndex = 0;
  saveStoreToEEPROM();
  Serial.println("✅ EEPROM Cleared!");
}

String hexUIDFromMFRC() {
  if (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) return "";
  String uid = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    if (mfrc522.uid.uidByte[i] < 0x10) uid += "0";
    uid += String(mfrc522.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();
  mfrc522.PICC_HaltA();
  return uid;
}

int findUserIndexByUID(const String &uid) {
  for (int i = 0; i < MAX_USERS; i++) {
    if (String(store.users[i].uid) == uid) return i;
  }
  return -1;
}

String epochToDate(unsigned long epoch) {
  if (epoch == 0) return "";
  time_t t = (time_t)epoch;
  struct tm *tm = localtime(&t);
  if (!tm) return "";
  char buf[20];
  sprintf(buf, "%02d/%02d/%04d", tm->tm_mday, tm->tm_mon + 1, tm->tm_year + 1900);
  return String(buf);
}

String epochToTime(unsigned long epoch) {
  if (epoch == 0) return "";
  time_t t = (time_t)epoch;
  struct tm *tm = localtime(&t);
  if (!tm) return "";
  char buf[12];
  sprintf(buf, "%02d:%02d:%02d", tm->tm_hour, tm->tm_min, tm->tm_sec);
  return String(buf);
}

// URL-encode सिर्फ space के लिए (basic). चाहें तो और chars जोड़ सकते हैं.
String urlEncodeSpaces(const String &s) {
  String out = s;
  out.replace(" ", "%20");
  return out;
}

// Send to Google Sheet: हर बार नया row (date + time भेजें), outtime की जरूरत नहीं
void sendToGoogleSheet(const String &uid, const String &name, const String &date, const String &timeNow) {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("WiFi not connected - cannot send to sheet");
    return;
  }

  std::unique_ptr<BearSSL::WiFiClientSecure> client(new BearSSL::WiFiClientSecure);
  client->setInsecure();
  HTTPClient http;

  String url = GOOGLE_SCRIPT_URL;
  url += "?uid=" + uid;
  url += "&name=" + urlEncodeSpaces(name);
  url += "&date=" + date;
  url += "&intime=" + timeNow;   // Apps Script इसे "Time" की तरह इस्तेमाल करेगा

  Serial.println("Sending URL: " + url);
  if (http.begin(*client, url)) {
    int httpCode = http.GET();
    Serial.printf("HTTP Response: %d\n", httpCode);
    http.end();
  } else {
    Serial.println("HTTP begin failed");
  }
}

// ---------- Core functions ----------
void addOrUpdateUserEntry(const String &uid, const String &nameStr) {
  int idx = findUserIndexByUID(uid);
  if (idx != -1) {
    // user exists -> update name
    nameStr.toCharArray(store.users[idx].name, NAME_LENGTH);
    saveStoreToEEPROM();
    Serial.println("✅ User existed - name updated");
    return;
  }

  // find first empty slot
  for (int i = 0; i < MAX_USERS; i++) {
    if (strlen(store.users[i].uid) == 0) {
      uid.toCharArray(store.users[i].uid, UID_LENGTH + 1);
      nameStr.toCharArray(store.users[i].name, NAME_LENGTH);
      saveStoreToEEPROM();
      Serial.println("✅ User Added Successfully (empty slot)");
      return;
    }
  }

  // No empty slot -> overwrite using FIFO pointer
  int w = store.nextWriteIndex % MAX_USERS;
  Serial.printf("⚠ Storage full - overwriting at index %d (old UID=%s)\n", w, store.users[w].uid);
  uid.toCharArray(store.users[w].uid, UID_LENGTH + 1);
  nameStr.toCharArray(store.users[w].name, NAME_LENGTH);
  store.nextWriteIndex = (store.nextWriteIndex + 1) % MAX_USERS;
  saveStoreToEEPROM();
  Serial.println("✅ New User Added (after overwrite)");
}

// हर टैप = एक नया लॉग
void logScanOnce(const String &uid) {
  int idx = findUserIndexByUID(uid);
  if (idx == -1) {
    Serial.println("❌ Unknown Card! (not registered)");
    beep(500);
    return;
  }

  // हर स्कैन पर latest time
  timeClient.update();
  unsigned long epoch = timeClient.getEpochTime();
  if (epoch < 1600000000UL) {
    epoch = (unsigned long)time(nullptr);
  }

  String dateStr = epochToDate(epoch);
  String timeStr = epochToTime(epoch);

  Serial.printf("%s ✅ Logged at %s %s\n", store.users[idx].name, dateStr.c_str(), timeStr.c_str());

  sendToGoogleSheet(String(store.users[idx].uid),
                    String(store.users[idx].name),
                    dateStr,
                    timeStr);

  beep(120);
}

// ---------- Setup & Loop ----------
void setup() {
  Serial.begin(115200);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  SPI.begin();
  mfrc522.PCD_Init();

  EEPROM.begin(EEPROM_SIZE);
  loadStoreFromEEPROM();

  // अगर सब कुछ साफ़ करना हो तो नीचे वाली लाइन एक बार अनकमेंट करके अपलोड-रन करें, फिर वापस कमेंट कर दें।
  // clearEEPROM();

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  unsigned long wifiStart = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - wifiStart < 15000UL) {
    delay(500);
    Serial.print(".");
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected!");
  } else {
    Serial.println("\nWiFi not connected (continue offline)");
  }

  timeClient.begin();
  for (int i = 0; i < 8; i++) {
    timeClient.update();
    delay(200);
  }

  Serial.println("System Ready - Scan Master to toggle ADD MODE");
}

void loop() {
  timeClient.update();

  String uid = hexUIDFromMFRC();
  if (uid == "") return;

  Serial.println("Scanned UID: " + uid);

  if (uid == MASTER_UID) {
    addMode = !addMode;
    Serial.println(addMode ? "🔑 ADD MODE ON - Scan card to add (then type name in Serial)" : "🔒 ADD MODE OFF");
    beep(200);
    return;
  }

  if (addMode) {
    // Add mode: ask for name via Serial
    Serial.println("Enter Name for this UID (type in Serial Monitor, press ENTER): ");
    String empName = "";
    unsigned long start = millis();
    while (empName.length() == 0) {
      if (Serial.available()) {
        empName = Serial.readStringUntil('\n');
        empName.trim();
      }
      // timeout after 30 seconds
      if (millis() - start > 30000UL) {
        Serial.println("Timeout waiting for name. Add cancelled.");
        empName = "";
        break;
      }
      delay(10);
    }
    if (empName.length() > 0) {
      addOrUpdateUserEntry(uid, empName);
      Serial.println("Saved Name: " + empName);
    }
    addMode = false; // exit add mode automatically
    beep(150);
    return;
  }

  // Simple logging flow
  logScanOnce(uid);
}
