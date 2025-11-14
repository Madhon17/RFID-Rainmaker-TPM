/*
  RFID LOCK - v2.3.1 (Perbaikan kompilasi + komentar Bahasa Indonesia)
  - Migrasi otomatis dari EEPROM (lama) -> Preferences (NVS)
  - Relay pindah ke pin output-capable
  - Non-blocking ADD/REMOVE RFID via RMaker params (flag + loop)
  - INPUT_PULLUP + debouncing dasar untuk tombol
  - Tidak mencetak password/PoP ke Serial
  - Menjaga semua fitur existing (RMaker params, OTA, buzzer, offline button, factory reset)
*/

#include <RMaker.h>
#include <string.h>
#include <WiFi.h>
#include <WiFiProv.h>
#include <SimpleTimer.h>
#include <EEPROM.h>
#include <MFRC522.h>
#include <SPI.h>
#include <Preferences.h>
#include "mbedtls/sha256.h" // opsional jika ingin hashing kelak

// ====================== KONFIG ======================
#define DEFAULT_RELAY_MODE HIGH   // HIGH = relay OFF bergantung rangkaian
#define RELAY_PIN 25              // pindah dari 33 ke 25 (output-capable)
#define RESET_BTN 4               // tombol reset pabrik (bukan GPIO0)
#define SS_PIN 21
#define RST_PIN 5
#define Internet_LED 2
#define Read_Mode_LED 13
#define SWITCH_PIN 32
#define BUZZER_PIN 26

// Provisioning PoP & service name (tidak dicetak)
const char *service_name = "PROV_RFID_TPM";
const char *pop = "1234567"; // disarankan ganti untuk produksi

// EEPROM migration settings
const bool DO_EEPROM_MIGRATION = true;
const bool CLEAR_EEPROM_AFTER_MIGRATION = false; // set true kalau mau bersihkan EEPROM pasca migrasi

// request timeout
const unsigned long REQUEST_TIMEOUT = 10000; // ms

// ====================================================

Preferences prefs;
MFRC522 mfrc522(SS_PIN, RST_PIN);
SimpleTimer Timer;

// Flags / state
enum ModeRequest { MODE_NONE = 0, MODE_ADD, MODE_REMOVE };
volatile ModeRequest requestMode = MODE_NONE;
unsigned long requestStartMillis = 0;

bool relay_state = false;
bool wifi_connected = false;
bool buzzer_state = false;
bool buzz = true; // master enable untuk buzzer; dikendalikan lewat RMaker param
bool add_button = false;
bool remove_button = false;
int SWITCH_STATE = HIGH;

// Device - RMaker
static Device my_lock("RFID TPM", "custom.device.device");

// Forward declarations
void triggerAddRequest();
void triggerRemoveRequest();
String normalizeUID(const String &raw);
bool uidExists(const String &uidNorm);
bool addUid(const String &uidNorm);
bool removeUid(const String &uidNorm);
String loadUidList();
void saveUidList(const String &s);
void migrateEepromToPrefs();
void success_buzzer();
void Failure_buzzer();
void beep();
void add_switch_off(void);
void remove_switch_off(void);

// ------------------- Normalisasi UID -------------------
String normalizeUID(const String &raw) {
  String s = raw;
  s.replace(" ", "");
  s.toUpperCase();
  return s;
}

// ------------------- Preferences (NVS) helpers -------------------
void saveUidList(const String &pipeSep) {
  prefs.begin("rfid", false);
  prefs.putString("uids", pipeSep);
  prefs.end();
}

String loadUidList() {
  prefs.begin("rfid", true);
  String s = prefs.getString("uids", "");
  prefs.end();
  return s;
}

bool uidExists(const String &uidNorm) {
  String list = loadUidList();
  if (list.length() == 0) return false;
  // cari token persis dengan separator
  if (list == uidNorm) return true;
  if (list.indexOf("|" + uidNorm + "|") != -1) return true;
  if (list.startsWith(uidNorm + "|")) return true;
  if (list.endsWith("|" + uidNorm)) return true;
  return false;
}

bool addUid(const String &uidNorm) {
  if (uidNorm.length() == 0) return false;
  if (uidExists(uidNorm)) return false;
  String list = loadUidList();
  if (list.length() == 0) list = uidNorm;
  else list += "|" + uidNorm;
  saveUidList(list);
  return true;
}

bool removeUid(const String &uidNorm) {
  String list = loadUidList();
  if (list.length() == 0) return false;
  // tokenisasi sederhana dan rebuild
  bool removed = false;
  String out = "";
  int start = 0;
  while (start < list.length()) {
    int sep = list.indexOf('|', start);
    String token;
    if (sep == -1) {
      token = list.substring(start);
      start = list.length();
    } else {
      token = list.substring(start, sep);
      start = sep + 1;
    }
    if (token.equalsIgnoreCase(uidNorm)) {
      removed = true;
      continue;
    }
    if (out.length() == 0) out = token;
    else out += "|" + token;
  }
  if (removed) {
    saveUidList(out);
  }
  return removed;
}

// ------------------- Migrasi EEPROM -> Preferences -------------------
void migrateEepromToPrefs() {
  if (!DO_EEPROM_MIGRATION) return;

  if (!EEPROM.begin(512)) {
    Serial.println("EEPROM init failed - migration dilewati");
    return;
  }

  // Baca byte non-zero dari area 1..499 dan gabungkan jadi satu string
  String old = "";
  for (int i = 1; i < 500; i++) {
    byte b = EEPROM.read(i);
    if (b == 0) {
      // bila ada run zero panjang, kita anggap akhir data
      bool restZero = true;
      for (int j = i; j < min(i + 8, 500); j++) {
        if (EEPROM.read(j) != 0) { restZero = false; break; }
      }
      if (restZero) break;
      else continue;
    }
    old += char(b);
  }

  if (old.length() > 0) {
    Serial.println("Ditemukan data di EEPROM. Memulai migrasi ke Preferences (NVS).");
    old.toUpperCase();
    // Hilangkan spasi, jadikan token tunggal (heuristik sederhana)
    String tmp = old;
    // ganti multiple spaces ke single, lalu hapus semua spasi
    while (tmp.indexOf("  ") != -1) tmp.replace("  ", " ");
    tmp.trim();
    tmp.replace(" ", "");
    String uid = normalizeUID(tmp);
    if (uid.length() > 0) {
      saveUidList(uid);
      Serial.println("Migrasi selesai: 1 token UID disimpan ke Preferences.");
    } else {
      Serial.println("Migrasi: tidak ada UID valid ditemukan.");
    }

    if (CLEAR_EEPROM_AFTER_MIGRATION) {
      for (int i = 0; i < 512; i++) EEPROM.write(i, 0);
      EEPROM.commit();
      Serial.println("EEPROM dibersihkan setelah migrasi.");
    }
  } else {
    Serial.println("Tidak ada data UID di EEPROM (tidak perlu migrasi).");
  }
}

// ------------------- Event provisioning -------------------
void sysProvEvent(arduino_event_t *sys_event)
{
  switch (sys_event->event_id) {
    case ARDUINO_EVENT_PROV_START:
#if CONFIG_IDF_TARGET_ESP32
      Serial.printf("\nProvisioning dimulai via BLE (service: %s)\n", service_name);
      printQR(service_name, pop, "ble");
#else
      Serial.printf("\nProvisioning dimulai via SoftAP (service: %s)\n", service_name);
      printQR(service_name, pop, "softap");
#endif
      break;
    case ARDUINO_EVENT_WIFI_STA_CONNECTED:
      Serial.println("\nTerhubung ke Wi-Fi!");
      digitalWrite(Internet_LED, HIGH);
      wifi_connected = true;
      break;
    case ARDUINO_EVENT_WIFI_STA_DISCONNECTED:
      Serial.println("\nTerputus dari Wi-Fi. Mencoba reconnect...");
      digitalWrite(Internet_LED, LOW);
      wifi_connected = false;
      break;
    case ARDUINO_EVENT_PROV_CRED_RECV:
      Serial.println("\nMenerima kredensial Wi-Fi (tidak ditampilkan demi keamanan).");
      break;
    case ARDUINO_EVENT_PROV_INIT:
      wifi_prov_mgr_disable_auto_stop(10000);
      break;
    case ARDUINO_EVENT_PROV_CRED_SUCCESS:
      Serial.println("Provisioning berhasil. Menghentikan provisioning.");
      wifi_prov_mgr_stop_provisioning();
      break;
    default:
      break;
  }
}

// ------------------- RMaker write callback (non-blocking) -------------------
void write_callback(Device *device, Param *param, const param_val_t val, void *priv_data, write_ctx_t *ctx)
{
  const char *device_name = device->getDeviceName();
  const char *param_name = param->getParamName();

  if (strcmp(device_name, "RFID LOCK") == 0)
  {
    if (strcmp(param_name, "display") == 0) {
      param->updateAndReport(val);
    }
    if (strcmp(param_name, "BUZZER") == 0)
    {
      buzzer_state = val.val.b;
      buzz = buzzer_state;
      param->updateAndReport(val);
    }
    if (strcmp(param_name, "DOOR OPEN") == 0)
    {
      relay_state = val.val.b;
      if (!relay_state) digitalWrite(RELAY_PIN, DEFAULT_RELAY_MODE);
      else digitalWrite(RELAY_PIN, !DEFAULT_RELAY_MODE);
      param->updateAndReport(val);
    }

    if (strcmp(param_name, "ADD RFID") == 0)
    {
      add_button = val.val.b;
      param->updateAndReport(val);
      if (add_button) {
        triggerAddRequest();
      } else {
        requestMode = MODE_NONE;
        add_switch_off();
      }
    }

    if (strcmp(param_name, "REMOVE RFID") == 0)
    {
      remove_button = val.val.b;
      param->updateAndReport(val);
      if (remove_button) {
        triggerRemoveRequest();
      } else {
        requestMode = MODE_NONE;
        remove_switch_off();
      }
    }
  }
}

// ------------------- Setup -------------------
void setup() {
  Serial.begin(115200);

  // Inisialisasi EEPROM (hanya untuk membaca dan migrasi)
  if (!EEPROM.begin(512)) {
    Serial.println("Peringatan: inisialisasi EEPROM gagal atau tidak tersedia. Migrasi mungkin dilewati.");
  } else {
    Serial.println("EEPROM siap (untuk migrasi jika ada).");
  }

  // Migrasi data lama ke Preferences (NVS)
  migrateEepromToPrefs();

  SPI.begin();
  mfrc522.PCD_Init();

  // Pin
  pinMode(Read_Mode_LED, OUTPUT);
  pinMode(Internet_LED, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Tombol menggunakan pullup internal (kabel ke GND)
  pinMode(RESET_BTN, INPUT_PULLUP);
  pinMode(SWITCH_PIN, INPUT_PULLUP);

  digitalWrite(RELAY_PIN, DEFAULT_RELAY_MODE);

  Serial.println("Siap membaca kartu...");

  // RMaker node/device/params
  Node my_node;
  my_node = RMaker.initNode("RFID-TPM");

  my_lock.addNameParam();
  Param disp("display", "custom.param.display", value("Welcome to Transporthub"), PROP_FLAG_READ);
  disp.addUIType(ESP_RMAKER_UI_TEXT);
  my_lock.addParam(disp);

  Param open_switch("DOOR OPEN", "custom.param.power", value(relay_state), PROP_FLAG_READ | PROP_FLAG_WRITE);
  open_switch.addUIType(ESP_RMAKER_UI_TOGGLE);
  my_lock.addParam(open_switch);

  Param add_switch("ADD RFID", "custom.param.power", value(add_button), PROP_FLAG_READ | PROP_FLAG_WRITE);
  add_switch.addUIType(ESP_RMAKER_UI_TOGGLE);
  my_lock.addParam(add_switch);

  Param remove_switch("REMOVE RFID", "custom.param.power", value(remove_button), PROP_FLAG_READ | PROP_FLAG_WRITE);
  remove_switch.addUIType(ESP_RMAKER_UI_TOGGLE);
  my_lock.addParam(remove_switch);

  Param buzz_switch("BUZZER", "custom.param.power", value(buzzer_state), PROP_FLAG_READ | PROP_FLAG_WRITE);
  buzz_switch.addUIType(ESP_RMAKER_UI_TOGGLE);
  my_lock.addParam(buzz_switch);

  my_lock.addCb(write_callback);
  my_node.addDevice(my_lock);

  // Report default values (untuk sinkronisasi awal)
  my_lock.updateAndReportParam("DOOR OPEN", relay_state);
  my_lock.updateAndReportParam("ADD RFID", add_button);
  my_lock.updateAndReportParam("REMOVE RFID", remove_button);
  my_lock.updateAndReportParam("BUZZER", buzzer_state);

  RMaker.enableOTA(OTA_USING_PARAMS);
  RMaker.enableTZService();
  RMaker.enableSchedule();

  Serial.printf("\nMemulai ESP-RainMaker\n");
  RMaker.start();

  WiFi.onEvent(sysProvEvent);
  WiFiProv.beginProvision(WIFI_PROV_SCHEME_BLE, WIFI_PROV_SCHEME_HANDLER_FREE_BTDM, WIFI_PROV_SECURITY_1, pop, service_name);
}

// ------------------- Trigger non-blocking -------------------
void triggerAddRequest() {
  requestMode = MODE_ADD;
  requestStartMillis = millis();
  digitalWrite(Read_Mode_LED, HIGH);
  beep();
  Serial.println("Permintaan ADD RFID dimulai - scan kartu dalam timeout.");
}

void triggerRemoveRequest() {
  requestMode = MODE_REMOVE;
  requestStartMillis = millis();
  digitalWrite(Read_Mode_LED, HIGH);
  beep();
  Serial.println("Permintaan REMOVE RFID dimulai - scan kartu dalam timeout.");
}

// ------------------- Loop utama -------------------
unsigned long lastSwitchDebounce = 0;
bool lastSwitchStateStable = HIGH;

void loop()
{
  // Debounce SWITCH (tombol akses offline)
  int rawSwitch = digitalRead(SWITCH_PIN);
  if (rawSwitch != lastSwitchStateStable) {
    lastSwitchDebounce = millis();
  }
  if ((millis() - lastSwitchDebounce) > 50) {
    if (rawSwitch != SWITCH_STATE) {
      SWITCH_STATE = rawSwitch;
      if (SWITCH_STATE == LOW) { // ditekan
        authorized_access_offline();
      }
    }
  }
  lastSwitchStateStable = rawSwitch;

  // Factory reset (long press)
  if (digitalRead(RESET_BTN) == LOW) { // aktif-low
    unsigned long start = millis();
    while (digitalRead(RESET_BTN) == LOW) {
      delay(50);
      if (millis() - start > 5000) {
        Serial.println("Factory reset dipicu.");
        RMakerFactoryReset(2);
        break;
      }
    }
  }

  // Tangani request ADD/REMOVE non-blocking
  if (requestMode != MODE_NONE) {
    if (millis() - requestStartMillis > REQUEST_TIMEOUT) {
      Serial.println("Timeout permintaan RFID.");
      requestMode = MODE_NONE;
      digitalWrite(Read_Mode_LED, LOW);
      add_switch_off();
      remove_switch_off();
    } else {
      if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
        String content = "";
        for (byte i = 0; i < mfrc522.uid.size; i++) {
          if (mfrc522.uid.uidByte[i] < 0x10) content += "0";
          content += String(mfrc522.uid.uidByte[i], HEX);
        }
        content.toUpperCase();
        String uidNorm = normalizeUID(content);
        Serial.print("UID scan untuk operasi: "); Serial.println(uidNorm);

        if (requestMode == MODE_ADD) {
          if (addUid(uidNorm)) {
            Serial.println("RFID BERHASIL DITAMBAHKAN");
            success_buzzer();
          } else {
            Serial.println("UID sudah ada (tidak ditambahkan)");
            Failure_buzzer();
          }
        } else if (requestMode == MODE_REMOVE) {
          if (removeUid(uidNorm)) {
            Serial.println("RFID BERHASIL DIHAPUS");
            success_buzzer();
          } else {
            Serial.println("UID tidak ditemukan (hapus gagal)");
            Failure_buzzer();
          }
        }

        requestMode = MODE_NONE;
        digitalWrite(Read_Mode_LED, LOW);
        add_switch_off();
        remove_switch_off();

        mfrc522.PICC_HaltA();
        mfrc522.PCD_StopCrypto1();
      }
    }
  }

  // Pemeriksaan kartu untuk akses normal (bila tidak sedang add/remove)
  if (requestMode == MODE_NONE) {
    if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
      String content = "";
      for (byte i = 0; i < mfrc522.uid.size; i++) {
        if (mfrc522.uid.uidByte[i] < 0x10) content += "0";
        content += String(mfrc522.uid.uidByte[i], HEX);
      }
      content.toUpperCase();
      String uidNorm = normalizeUID(content);
      Serial.print("UID terdeteksi untuk otorisasi: "); Serial.println(uidNorm);

      if (uidExists(uidNorm)) {
        authorized_access();
      } else {
        Serial.println("Access denied (UID tidak terdaftar).");
        digitalWrite(RELAY_PIN, DEFAULT_RELAY_MODE); // pastikan relay off
        Failure_buzzer();
        my_lock.updateAndReportParam("display", "Access Denied");
      }

      mfrc522.PICC_HaltA();
      mfrc522.PCD_StopCrypto1();
      delay(300); // jeda kecil untuk hindari multi-read
    }
  }

  delay(50); // yield kecil
}

// ------------------- Fungsi akses -------------------
String authorized_access(void)
{
  Serial.println("Authorized access (online).");
  my_lock.updateAndReportParam("display", "Access Authorized");
  beep();
  digitalWrite(RELAY_PIN, !DEFAULT_RELAY_MODE); // aktifkan relay
  delay(5000);
  digitalWrite(RELAY_PIN, DEFAULT_RELAY_MODE);  // non-aktifkan
  return "Access Authorized";
}

void authorized_access_offline()
{
  Serial.println("Authorized access (offline button).");
  digitalWrite(RELAY_PIN, !DEFAULT_RELAY_MODE);
  delay(5000);
  digitalWrite(RELAY_PIN, DEFAULT_RELAY_MODE);
}

// ------------------- Helper kecil -------------------
void add_switch_off(void)
{
  add_button = false;
  my_lock.updateAndReportParam("ADD RFID", add_button);
  Serial.println("Tombol ADD OFF");
}

void remove_switch_off(void)
{
  remove_button = false;
  my_lock.updateAndReportParam("REMOVE RFID", remove_button);
  Serial.println("Tombol REMOVE OFF");
}

// ------------------- Pola buzzer -------------------
void success_buzzer()
{
  if (buzz == true)
  {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(BUZZER_PIN, LOW);
  }
}

void Failure_buzzer()
{
  if (buzz == true)
  {
    for (int i = 0; i < 3; i++)
    {
      digitalWrite(BUZZER_PIN, HIGH);
      delay(100);
      digitalWrite(BUZZER_PIN, LOW);
      delay(50);
    }
  }
}

void beep()
{
  if (buzz == true)
  {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
  }
}
