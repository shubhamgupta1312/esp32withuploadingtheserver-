#include <SPI.h>
#include <MFRC522.h>
#include <Servo_ESP32.h>
#include <WiFi.h>
#include <ESPAsyncWebServer.h>

String getFormattedUID(MFRC522::Uid uid);



#define SS_PIN    5    // ESP32 pin GPIO5
#define RST_PIN   27   // ESP32 pin GPIO27
#define BUZZER_PIN 22   // Buzzer pin
#define BLUE_LED  12   // Blue LED GPIO pin
#define RED_LED   16   // Red LED GPIO pin
#define GREEN_LED 21   // Green LED GPIO pin
#define SERVO_PIN  15   // Servo motor GPIO pin

MFRC522 rfid(SS_PIN, RST_PIN);
Servo_ESP32 gateServo;

const char* ssid = "Khushi";
const char* password = "47273ECECD9E";

AsyncWebServer server(80);

bool blueLEDOn = true;
bool greenLEDOn = false;
bool redLEDOn = false;
bool gateOpen = false; // Track gate state
unsigned long gateOpenTime = 0;
const unsigned long gateOpenDuration = 10000; // 10 seconds

// Function definition
String getFormattedUID(MFRC522::Uid uid) {
  String formattedUID = "";
  for (int i = 0; i < uid.size; i++) {
    formattedUID += (uid.uidByte[i] < 0x10 ? " 0" : " ");
    formattedUID += String(uid.uidByte[i], HEX);
  }
  return formattedUID;
}

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  gateServo.attach(SERVO_PIN);

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BLUE_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  digitalWrite(BLUE_LED, HIGH);
  blueLEDOn = true;

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");

  // Print the ESP32's IP address to the Serial Monitor
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
  String html = "<html><head>";
  html += "<style>";
  html += ".button { width: 100px; height: 100px; font-size: 20px; transition: background-color 0.3s; }";
  html += ".button:hover { background-color: darkgrey; }";
  html += ".button:active { background-color: darkred; }";
  html += ".message { font-size: 24px; font-weight: bold; }"; // New style for the message
  html += "</style>";
  html += "</head><body>";
  html += "<h1>RFID Gate Control</h1>";
  html += "<span class='message' id='statusMessage'></span><br>"; // New span for the message
  html += "Gate is: <span id='gateStatus'>Unknown</span><br>";
  html += "Time remaining: <span id='timer'>0 seconds</span><br>";
  html += "<button class='button' style='background-color: green;' onclick='openGate()'>Open</button>";
  html += "<button class='button' style='background-color: red;' onclick='closeGate()'>Close</button>";
  html += "<button class='button' style='background-color: blue;' onclick='resetSystem()'>Reset</button>";
  html += "<script>";
  html += "function updateStatus() {";
  html += "fetch('/status')";
  html += ".then(response => response.json())";
  html += ".then(data => {";
  html += "document.getElementById('gateStatus').textContent = data.status;";
  html += "document.getElementById('timer').textContent = 'Time remaining: ' + data.remainingTime + ' seconds';";
  html += "if (data.message) document.getElementById('statusMessage').innerHTML = data.message;"; // New line for updating the message
  html += "});";
  html += "}";
  html += "function openGate() {";
  html += "fetch('/open');";
  html += "}";
  html += "function closeGate() {";
  html += "fetch('/close');";
  html += "}";
  html += "function resetSystem() {";
  html += "fetch('/reset');";
  html += "}";
  html += "updateStatus();";
  html += "setInterval(updateStatus, 2000);"; // Update every 2 seconds
  html += "</script>";
  html += "</body></html>";
  request->send(200, "text/html", html);
});

// ...

server.on("/status", HTTP_GET, [](AsyncWebServerRequest *request){
  String status = gateOpen ? "Open" : "Closed";
  unsigned long currentTime = millis();
  unsigned long remainingTime = gateOpen ? (currentTime - gateOpenTime > gateOpenDuration ? 0 : gateOpenDuration - (currentTime - gateOpenTime)) : 0;

  String message = "";
if (isAuthorized(rfid.uid)) {
  message = "<span class='message' style='color: green; font-weight: bold;'>Welcome! Access granted. UID: " + getFormattedUID(rfid.uid) + "</span>";
} else {
  message = "<span class='message' style='color: red; font-weight: bold;'>Intruder alert! Unauthorized access. UID: " + getFormattedUID(rfid.uid) + "</span>";
}

  String json = "{\"status\":\"" + status + "\",\"remainingTime\":" + String(remainingTime / 1000) + ",\"message\":\"" + message + "\"}";
  request->send(200, "application/json", json);
});

  server.on("/open", HTTP_GET, [](AsyncWebServerRequest *request){
    openGate();
    request->send(200, "text/plain", "Gate opened.");
  });

  server.on("/close", HTTP_GET, [](AsyncWebServerRequest *request){
    closeGate();
    request->send(200, "text/plain", "Gate closed.");
  });

  server.on("/reset", HTTP_GET, [](AsyncWebServerRequest *request){
    resetSystem();
    request->send(200, "text/plain", "System reset.");
  });

  server.begin();

  Serial.println("Tap an RFID/NFC tag on the RFID-RC522 reader");
}

void loop() {
  if (rfid.PICC_IsNewCardPresent()) {
    if (rfid.PICC_ReadCardSerial()) {
      MFRC522::PICC_Type piccType = rfid.PICC_GetType(rfid.uid.sak);
      Serial.print("RFID/NFC Tag Type: ");
      Serial.println(rfid.PICC_GetTypeName(piccType));

      Serial.print("UID:");
      for (int i = 0; i < rfid.uid.size; i++) {
        Serial.print(rfid.uid.uidByte[i] < 0x10 ? " 0" : " ");
        Serial.print(rfid.uid.uidByte[i], HEX);
      }
      Serial.println();

      if (isAuthorized(rfid.uid)) {
        beepHigh(375);
        digitalWrite(GREEN_LED, HIGH);
        greenLEDOn = true;
        if (redLEDOn) {
          digitalWrite(RED_LED, LOW);
          redLEDOn = false;
        }
        if (blueLEDOn) {
          digitalWrite(BLUE_LED, LOW);
          blueLEDOn = false;
        }
        openGate();
        gateOpen = true;
        gateOpenTime = millis();
        delay(10000);
        closeGate();
        delay(5000);
      } else {
        beepHigh(1000);
        digitalWrite(RED_LED, HIGH);
        redLEDOn = true;
        if (greenLEDOn) {
          digitalWrite(GREEN_LED, LOW);
          greenLEDOn = false;
        }
        if (blueLEDOn) {
          digitalWrite(BLUE_LED, LOW);
          blueLEDOn = false;
        }
        closeGate();
        gateOpen = false;
        delay(5000);
      }
      resetSystem();
      rfid.PICC_HaltA();
      rfid.PCD_StopCrypto1();
    }
  }
}

bool isAuthorized(MFRC522::Uid uid) {
  return (uid.uidByte[0] == 0x03 &&
          uid.uidByte[1] == 0x59 &&
          uid.uidByte[2] == 0xD8 &&
          uid.uidByte[3] == 0x1B);
}

void beepHigh(int duration) {
  tone(BUZZER_PIN, 2000, duration);
  delay(1000);
  noTone(BUZZER_PIN);
}

void openGate() {
  gateServo.attach(SERVO_PIN);
  gateServo.write(180);
  delay(1000);
  gateServo.detach();
  digitalWrite(RED_LED, LOW);
  digitalWrite(GREEN_LED, HIGH);
  digitalWrite(BLUE_LED, LOW);
  gateOpen = true;
  gateOpenTime = millis();
}

void closeGate() {
  gateServo.attach(SERVO_PIN);
  gateServo.write(20);
  delay(1000);
  gateServo.detach();
  digitalWrite(RED_LED, HIGH);
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(BLUE_LED, LOW);
  gateOpen = false;
}

void resetSystem() {
  if (redLEDOn) {
    digitalWrite(RED_LED, LOW);
    redLEDOn = false;
  }
  if (gateOpen) {
    closeGate();
    gateOpen = false;
  }
  gateServo.attach(SERVO_PIN);
  gateServo.write(0);
  if (!gateOpen) {
    digitalWrite(RED_LED, LOW);
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }
  if (redLEDOn) {
    digitalWrite(RED_LED, LOW);
    redLEDOn = false;
  }
  if (greenLEDOn) {
    digitalWrite(GREEN_LED, LOW);
    greenLEDOn = false;
  }
}
