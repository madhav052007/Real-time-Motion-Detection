# Real-time-Motion-Detection
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <Wire.h>

// ADXL345 I2C address
#define Addr 0x53

const uint8_t scl = 14; // D5
const uint8_t sda = 12; // D6

const char* ssid = "Enthusiasts";
const char* password = "madhav1234";

ESP8266WebServer server(80);

float xAccl, yAccl, zAccl;

void initADXL345() {
  Wire.beginTransmission(Addr);
  Wire.write(0x2D); // Power control
  Wire.write(0x08); // Measurement mode
  Wire.endTransmission();

  Wire.beginTransmission(Addr);
  Wire.write(0x31); // Data format
  Wire.write(0x08); // Full resolution, +/-2g
  Wire.endTransmission();

  Wire.beginTransmission(Addr);
  Wire.write(0x2C); // Data rate
  Wire.write(0x0A); // 100Hz
  Wire.endTransmission();

  delay(100);
}

void readADXL345() {
  Wire.beginTransmission(Addr);
  Wire.write(0x32); // Start from data register
  Wire.endTransmission(false);
  Wire.requestFrom(Addr, 6, true);

  if (Wire.available() == 6) {
    int16_t x = (Wire.read() | (Wire.read() << 8));
    int16_t y = (Wire.read() | (Wire.read() << 8));
    int16_t z = (Wire.read() | (Wire.read() << 8));

    xAccl = x * 0.0039; // Convert to g
    yAccl = y * 0.0039;
    zAccl = z * 0.0039;
  }
}

void handleRoot() {
  String html = R"rawliteral(
  <html>
  <head>
  <title>ADXL345 Real-Time Accelerometer Data</title>
  <meta name='viewport' content='width=device-width, initial-scale=1.0'>
  <style>
  body { font-family: Arial; text-align: center; background: #f0f0f0; margin-top: 50px; }
  h1 { color: #222; }
  .value { font-size: 1.8em; color: #0078D7; }
  </style>
  <script>
  function updateData() {
    fetch('/data')
      .then(r => r.json())
      .then(d => {
        document.getElementById('x').innerText = d.x.toFixed(3);
        document.getElementById('y').innerText = d.y.toFixed(3);
        document.getElementById('z').innerText = d.z.toFixed(3);
      });
  }
  setInterval(updateData, 500);
  </script>
  </head>
  <body>
  <h1>ADXL345 Real-Time Accelerometer Data</h1>
  X-Axis: <span class='value' id='x'>0</span> g</p>
  <p>Y-Axis: <span class='value' id='y'>0</span> g</p>
  <p>Z-Axis: <span class='value' id='z'>0</span> g</p>
  </body>
  </html>
  )rawliteral";

  server.send(200, "text/html", html);
}

void handleData() {
  readADXL345();
  String json = "{\"x\":" + String(xAccl, 3) + ",\"y\":" + String(yAccl, 3) + ",\"z\":" + String(zAccl, 3) + "}";
  server.send(200, "application/json", json);
}

void setup() {
  Wire.begin(sda, scl);
  Serial.begin(115200);
  delay(100);

  initADXL345();

  WiFi.begin(ssid, password);
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.begin();
  Serial.println("Server started!");
}

void loop() {
  server.handleClient();
}
