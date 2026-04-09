#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <EEPROM.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- WIFI CONFIGURATION ---
const char* ssid = "TP";
const char* password = "Trade.123";

// --- PIN DEFINITIONS ---
const int RELAY_PWR = D4; // Main transformer power relay
const int RELAY_CW  = D5; // Turn Right (Clockwise)
const int RELAY_CCW = D6; // Turn Left (Counter-Clockwise)
const int LIMIT_CW  = D3; // Hardware limit switch (Right)
const int LIMIT_CCW = D7; // Hardware limit switch (Left)

ESP8266WebServer server(80);
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// ===================================================================
// --- DIRECT I2C COMMUNICATION WITH AS5600 ENCODER (Address 0x36) ---
// ===================================================================
#define AS5600_ADDR 0x36
#define AS5600_STATUS_REG 0x0B
#define AS5600_ANGLE_REG  0x0E 

bool checkMagnet() {
  Wire.beginTransmission(AS5600_ADDR);
  Wire.write(AS5600_STATUS_REG);
  if (Wire.endTransmission(false) != 0) return false;
  
  Wire.requestFrom(AS5600_ADDR, 1);
  if (Wire.available()) {
    uint8_t status = Wire.read();
    return (status & 0x20) != 0; // Bit 5 is "MD" (Magnet Detected)
  }
  return false;
}

bool readAS5600(float &angle) {
  Wire.beginTransmission(AS5600_ADDR);
  Wire.write(AS5600_ANGLE_REG);
  if (Wire.endTransmission(false) != 0) return false;
  
  Wire.requestFrom(AS5600_ADDR, 2);
  if (Wire.available() >= 2) {
    uint16_t highByte = Wire.read();
    uint16_t lowByte = Wire.read();
    uint16_t rawAngle = (highByte << 8) | lowByte; 
    angle = (rawAngle * 360.0) / 4096.0; 
    return true;
  }
  return false;
}
// ===================================================================

int targetAzimuth = 180;
int currentAzimuth = 0;
const int deadZone = 2; 
String motorStatus = "STOP";
bool isMoving = false; 

bool sensorWorking = false;
unsigned long lastSensorCheck = 0;
float rawSensorAngle = 0.0;

// Calibration data structure (Zero Offset/Tare)
struct CalibData {
  float zeroOffset;
  bool isValid;
} calib;

// ===================================================================
// --- HTML PAGE STORED IN PROGMEM (Saves RAM) ---
// ===================================================================
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html><html><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width, initial-scale=1'>
<title>Rotor DX Pro (AS5600)</title>
<style>
  body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; text-align: center; background: #121212; color: #eee; margin: 0; padding: 20px; }
  h1 { color: #3498db; margin-top: 0; text-transform: uppercase; letter-spacing: 2px; font-size: 28px; }
  
  .container { display: flex; flex-direction: column; max-width: 1100px; margin: 0 auto; gap: 25px; }
  .panel { background: #1e1e1e; padding: 25px; border-radius: 12px; box-shadow: 0 8px 16px rgba(0,0,0,0.6); width: 100%; box-sizing: border-box; }
  
  @media (min-width: 800px) {
    .container { flex-direction: row; align-items: stretch; }
    .panel { width: 50%; }
  }

  .btn { padding: 12px; margin: 5px; border-radius: 6px; border: none; background: #444; color: white; cursor: pointer; font-size: 15px; font-weight: bold; transition: 0.2s; }
  .btn:hover { background: #555; }
  .btn-wide { width: 45%; max-width: 160px; }
  .btn-full { width: 95%; max-width: 330px; margin-top: 10px; }
  .btn-small { width: 65px; background: #3498db; }
  .btn-small:hover { background: #2980b9; }
  .btn-red { background: #e74c3c; } .btn-red:hover { background: #c0392b; }
  .btn-green { background: #2ecc71; width: 95%; max-width: 330px; margin-top: 15px;} .btn-green:hover { background: #27ae60; }
  
  #error_box { display: none; background: #c0392b; padding: 15px; margin-bottom: 20px; font-weight: bold; border-radius: 8px; letter-spacing: 1px; }
  canvas { background: #111; border-radius: 50%; box-shadow: inset 0 0 20px rgba(0,0,0,1), 0 0 15px rgba(0,0,0,0.8); margin: 15px 0; }
  
  #dxSearch { width: 100%; padding: 12px; margin-bottom: 10px; border-radius: 6px; border: 1px solid #444; background: #2a2a2a; color: white; font-size: 16px; box-sizing: border-box; }
  select { width: 100%; padding: 10px; font-size: 16px; border-radius: 6px; background: #2a2a2a; color: white; border: 1px solid #444; margin-bottom: 20px; cursor: pointer; box-sizing: border-box; }
  
  .slider-container { width: 100%; margin: 20px 0; }
  input[type=range] { width: 100%; cursor: pointer; margin: 0; }
  .slider-labels { display: flex; justify-content: space-between; padding: 0 5px; margin-top: 8px; color: #888; font-size: 13px; font-weight: bold; }
  
  .motor-panel { display: flex; justify-content: center; gap: 30px; margin-bottom: 15px; }
  .ind { padding: 10px 20px; border-radius: 6px; font-weight: bold; background: #2a2a2a; color: #666; transition: 0.3s; border: 2px solid #222; font-size: 14px;}
  .ind.active-ccw { background: #e67e22; color: #fff; box-shadow: 0 0 20px #e67e22; border-color: #e67e22; }
  .ind.active-cw { background: #2ecc71; color: #fff; box-shadow: 0 0 20px #2ecc71; border-color: #2ecc71; }
  
  #geo_info { font-size: 19px; color: #f1c40f; font-weight: bold; margin: 15px 0; min-height: 28px; }
  .status-text { font-size: 22px; margin: 10px 0; }
  hr { border: 0; height: 1px; background: #444; margin: 25px 0; }
</style></head><body>

<h1>Rotor DX Pro <span style="font-size:12px; color:#888;">(AS5600)</span></h1>
<div id='error_box'>⚠ ENCODER ERROR / NO MAGNET!</div>

<div class="container">
  
  <div class="panel">
    <div class="motor-panel">
      <div id="ind_ccw" class="ind">⮜ LEFT</div>
      <div id="ind_cw" class="ind">RIGHT ⮞</div>
    </div>

    <canvas id="gauge" width="280" height="280"></canvas>
    
    <div class="status-text">Target: <span id='targ' style="color:#3498db; font-weight:bold;">--</span>&deg; &nbsp;|&nbsp; Pos: <span id='curr' style="color:#e74c3c; font-weight:bold;">--</span>&deg;</div>
    <div id="geo_info">Loading directions...</div>
  </div>

  <div class="panel">
    <input type="text" id="dxSearch" placeholder="🔍 Search country (e.g. Japan)..." onkeyup="filterDX()">
    <select id="dxSelect" size="5" onchange="setFromDX()"></select>
    
    <div class="slider-container">
      <input type='range' min='0' max='359' value='180' id='azSlider' oninput='setTarget(this.value)'>
      <div class="slider-labels">
        <span>0&deg;</span><span>90&deg;</span><span>180&deg;</span><span>270&deg;</span><span>360&deg;</span>
      </div>
    </div>
    
    <div style="margin-bottom: 15px;">
      <button class='btn btn-small' onclick='stepTarget(-10)'>-10&deg;</button>
      <button class='btn btn-small' onclick='stepTarget(-1)'>-1&deg;</button>
      <button class='btn btn-small' onclick='stepTarget(1)'>+1&deg;</button>
      <button class='btn btn-small' onclick='stepTarget(10)'>+10&deg;</button>
    </div>
    
    <div>
      <button class='btn btn-wide' onclick='setTarget(0)'>North</button>
      <button class='btn btn-wide' onclick='setTarget(180)'>South</button>
    </div>
    
    <button class='btn btn-green' onclick='flipPath()'>Flip 180&deg; (Long Path)</button>
    
    <hr>
    <h3 style="margin-top:0; color:#888;">Zero Calibration</h3>
    <p style="font-size: 13px; color:#aaa; margin-bottom: 15px;">Physically point the antenna to True North, then click the button below to reset the sensor to 0&deg;.</p>
    <button class='btn btn-full btn-red' onclick='confirmCalib()'>SET CURRENT POSITION AS 0&deg;</button>
  </div>

</div>

<script>
  let currentTg = 0;
  
  // DXCC Database (Calculated for Central Europe / Poland)
  const dxcc = [
    // EUROPE
    {a: 190, n: "Albania (ZA)"}, {a: 235, n: "Andorra (C3)"}, {a: 200, n: "Austria (OE)"}, 
    {a: 265, n: "Belgium (ON)"}, {a: 85, n: "Belarus (EW)"}, {a: 150, n: "Bosnia (E7)"}, 
    {a: 145, n: "Bulgaria (LZ)"}, {a: 140, n: "Croatia (9A)"}, {a: 135, n: "Cyprus (5B)"}, 
    {a: 165, n: "Montenegro (4O)"}, {a: 190, n: "Czechia (OK)"}, {a: 315, n: "Denmark (OZ)"}, 
    {a: 35, n: "Estonia (ES)"}, {a: 25, n: "Finland (OH)"}, {a: 240, n: "France (F)"}, 
    {a: 150, n: "Greece (SV)"}, {a: 220, n: "Spain (EA)"}, {a: 265, n: "Netherlands (PA)"}, 
    {a: 285, n: "Ireland (EI)"}, {a: 320, n: "Iceland (TF)"}, {a: 25, n: "Lithuania (LY)"}, 
    {a: 260, n: "Luxembourg (LX)"}, {a: 35, n: "Latvia (YL)"}, {a: 160, n: "North Macedonia (Z3)"}, 
    {a: 185, n: "Malta (9H)"}, {a: 270, n: "Germany (DL)"}, {a: 345, n: "Norway (LA)"}, 
    {a: 225, n: "Portugal (CT)"}, {a: 60, n: "Russia - Europe (UA1-UA6)"}, 
    {a: 130, n: "Romania (YO)"}, {a: 155, n: "Serbia (YT/YU)"}, {a: 150, n: "Slovakia (OM)"}, 
    {a: 170, n: "Slovenia (S5)"}, {a: 220, n: "Switzerland (HB9)"}, {a: 345, n: "Sweden (SM)"}, 
    {a: 135, n: "Turkey (TA)"}, {a: 105, n: "Ukraine (UR)"}, {a: 150, n: "Hungary (HA)"}, 
    {a: 275, n: "United Kingdom (G, GM, GW)"}, {a: 190, n: "Italy (I)"},

    // ASIA
    {a: 130, n: "Saudi Arabia (HZ)"}, {a: 125, n: "Armenia (EK)"}, {a: 60, n: "China (BY)"}, 
    {a: 65, n: "Philippines (DU)"}, {a: 115, n: "Georgia (4L)"}, {a: 105, n: "India (VU)"}, 
    {a: 85, n: "Indonesia (YB)"}, {a: 115, n: "Iran (EP)"}, {a: 135, n: "Israel (4X)"}, 
    {a: 45, n: "Japan (JA)"}, {a: 125, n: "Jordan (JY)"}, {a: 95, n: "Kazakhstan (UN)"}, 
    {a: 55, n: "South Korea (HL)"}, {a: 135, n: "Kuwait (9K)"}, {a: 140, n: "Lebanon (OD)"}, 
    {a: 85, n: "Malaysia (9M)"}, {a: 120, n: "Oman (A4)"}, {a: 45, n: "Russia - Asia (UA9-UA0)"}, 
    {a: 100, n: "Thailand (HS)"}, {a: 65, n: "Taiwan (BV)"}, {a: 80, n: "Vietnam (3W)"}, 
    {a: 125, n: "UAE (A6)"},

    // NORTH & CENTRAL AMERICA
    {a: 340, n: "Alaska (KL7)"}, {a: 285, n: "Dominican Rep. (HI)"}, {a: 330, n: "Greenland (OX)"}, 
    {a: 295, n: "Guatemala (TG)"}, {a: 280, n: "Jamaica (6Y)"}, {a: 305, n: "Canada - East (VE1-VE3)"}, 
    {a: 330, n: "Canada - West (VE7)"}, {a: 290, n: "Cuba (CO)"}, {a: 295, n: "Mexico (XE)"}, 
    {a: 285, n: "Puerto Rico (KP4)"}, {a: 295, n: "USA - Florida (W4)"}, {a: 310, n: "USA - Central (W5/W9)"}, 
    {a: 300, n: "USA - East Coast (W1-W3)"}, {a: 320, n: "USA - West Coast (W6)"},

    // SOUTH AMERICA
    {a: 245, n: "Argentina (LU)"}, {a: 235, n: "Brazil - East (PY)"}, {a: 245, n: "Brazil - West (PT)"}, 
    {a: 255, n: "Chile (CE)"}, {a: 270, n: "Ecuador (HC)"}, {a: 275, n: "Colombia (HK)"}, 
    {a: 265, n: "Peru (OA)"}, {a: 245, n: "Uruguay (CX)"}, {a: 275, n: "Venezuela (YV)"},

    // AFRICA
    {a: 200, n: "Algeria (7X)"}, {a: 160, n: "Egypt (SU)"}, {a: 180, n: "Kenya (5Z)"}, 
    {a: 140, n: "Madagascar (5R)"}, {a: 210, n: "Morocco (CN)"}, {a: 165, n: "South Africa (ZS)"}, 
    {a: 225, n: "Senegal (6W)"}, {a: 195, n: "Tunisia (3V)"}, {a: 225, n: "Canary Islands (EA8)"}, 
    {a: 175, n: "Zambia (9J)"}, {a: 170, n: "Zimbabwe (Z2)"}, {a: 235, n: "Cape Verde (D4)"},

    // OCEANIA
    {a: 70, n: "Australia - East (VK4/VK8)"}, {a: 85, n: "Australia - West (VK6)"}, 
    {a: 350, n: "Hawaii (KH6)"}, {a: 55, n: "Mariana Islands (KH0)"}, {a: 85, n: "New Zealand (ZL)"}
  ];

  function populateList(filterText = "") {
    const sel = document.getElementById("dxSelect");
    sel.innerHTML = '<option value="" selected disabled>Select destination...</option>';
    let f = filterText.toLowerCase();
    
    dxcc.filter(d => d.n.toLowerCase().includes(f) || d.a.toString() === f)
        .forEach(d => {
          let opt = document.createElement("option");
          opt.value = d.a;
          opt.innerHTML = d.n + " (" + d.a + "&deg;)";
          sel.appendChild(opt);
    });
  }

  function filterDX() { populateList(document.getElementById('dxSearch').value); }

  window.onload = function() {
    dxcc.sort((a, b) => a.n.localeCompare(b.n)); 
    populateList();
  };
  
  function f(url){ fetch(url); }
  
  function setTarget(val) {
    currentTg = parseInt(val);
    document.getElementById('targ').innerHTML = currentTg;
    document.getElementById('azSlider').value = currentTg;
    f("/set?az=" + currentTg);
  }

  function stepTarget(delta) {
    let newVal = currentTg + delta;
    if(newVal >= 360) newVal -= 360;
    if(newVal < 0) newVal += 360;
    setTarget(newVal);
  }

  function setFromDX() {
    let val = document.getElementById('dxSelect').value;
    if(val !== "") setTarget(val);
  }

  function flipPath() {
    let newVal = currentTg + 180;
    if(newVal >= 360) newVal -= 360;
    setTarget(newVal);
  }
  
  function confirmCalib() {
    let isConfirmed = confirm("⚠️ WARNING!\n\nAre you sure you want to reset the zero position?\nMake sure the antenna is physically pointing to True North (0°).");
    if(isConfirmed) {
      f("/calib");
    }
  }
  
  function getCountries(az) {
      if (az >= 345 || az < 15) return "Scandinavia, Baltic Sea";
      if (az >= 15 && az < 45)  return "North Russia, Baltic States";
      if (az >= 45 && az < 75)  return "Japan, Far East";
      if (az >= 75 && az < 105) return "Australia, SE Asia";
      if (az >= 105 && az < 135) return "India, Middle East";
      if (az >= 135 && az < 165) return "Balkans, East Africa";
      if (az >= 165 && az < 195) return "South Africa, Italy, North Africa";
      if (az >= 195 && az < 225) return "Spain, West Africa";
      if (az >= 225 && az < 255) return "Brazil, South America";
      if (az >= 255 && az < 285) return "Argentina, Caribbean";
      if (az >= 285 && az < 315) return "USA (East - West)";
      if (az >= 315 && az < 345) return "Canada, Iceland, Greenland";
      return "Searching...";
  }
  
  function drawGauge(az, tg) {
    const c = document.getElementById("gauge");
    const ctx = c.getContext("2d");
    const cx = c.width/2; 
    const cy = c.height/2; 
    const r = cx - 25; 
    
    ctx.clearRect(0, 0, c.width, c.height);
    
    ctx.beginPath(); ctx.arc(cx, cy, r, 0, 2*Math.PI); 
    ctx.strokeStyle="#555"; ctx.lineWidth=3; ctx.stroke();
    
    ctx.textAlign="center"; ctx.textBaseline="middle";
    for(let i=0; i<360; i+=15) {
      let rad = (i - 90) * Math.PI / 180;
      let isMajor = (i % 45 === 0);
      let len = isMajor ? 12 : 6;
      
      ctx.beginPath();
      ctx.moveTo(cx + Math.cos(rad) * r, cy + Math.sin(rad) * r);
      ctx.lineTo(cx + Math.cos(rad) * (r - len), cy + Math.sin(rad) * (r - len));
      ctx.strokeStyle = isMajor ? "#aaa" : "#555";
      ctx.lineWidth = isMajor ? 2 : 1;
      ctx.stroke();
      
      if(isMajor) {
        let txtR = r - 26;
        ctx.fillStyle = "#ccc";
        ctx.font = i % 90 === 0 ? "bold 15px 'Segoe UI'" : "13px 'Segoe UI'";
        let label = i;
        if(i===0) label = "N";
        if(i===90) label = "E";
        if(i===180) label = "S";
        if(i===270) label = "W";
        ctx.fillText(label, cx + Math.cos(rad)*txtR, cy + Math.sin(rad)*txtR);
      }
    }
    
    function drawNeedle(angle, color, width, lenMult) {
      let rad = (angle - 90) * Math.PI / 180;
      ctx.beginPath(); 
      ctx.moveTo(cx, cy);
      ctx.lineTo(cx + Math.cos(rad) * (r * lenMult), cy + Math.sin(rad) * (r * lenMult));
      ctx.strokeStyle = color; ctx.lineWidth = width; ctx.lineCap = "round"; ctx.stroke();
    }
    
    drawNeedle(tg, "#3498db", 4, 0.95); 
    drawNeedle(az, "#e74c3c", 7, 0.85); 
    
    ctx.beginPath(); ctx.arc(cx, cy, 7, 0, 2*Math.PI);
    ctx.fillStyle="#fff"; ctx.fill();
  }
  
  setInterval(()=>{
    fetch('/data').then(r=>r.json()).then(d=>{
      document.getElementById('curr').innerHTML = d.az;
      
      if (currentTg !== d.tg) {
        currentTg = d.tg;
        document.getElementById('targ').innerHTML = currentTg;
        document.getElementById('azSlider').value = currentTg;
      }

      drawGauge(d.az, currentTg);
      document.getElementById('error_box').style.display = d.sensor ? 'none' : 'block';

      let ccwActive = (d.motor === 'CCW');
      let cwActive = (d.motor === 'CW');
      
      document.getElementById('ind_ccw').className = ccwActive ? 'ind active-ccw' : 'ind';
      document.getElementById('ind_cw').className = cwActive ? 'ind active-cw' : 'ind';
      
      document.getElementById('geo_info').innerHTML = getCountries(d.az);
    }).catch(e => console.log("Connection lost..."));
  }, 500); 
  
</script>
</body></html>
)rawliteral";
// ===================================================================

void setup() {
  Serial.begin(115200);
  
  // Configure relays
  pinMode(RELAY_PWR, OUTPUT);
  pinMode(RELAY_CW, OUTPUT); 
  pinMode(RELAY_CCW, OUTPUT);
  
  // Configure limit switches
  pinMode(LIMIT_CW, INPUT_PULLUP); 
  pinMode(LIMIT_CCW, INPUT_PULLUP);
  
  // Ensure all relays are OFF on boot
  digitalWrite(RELAY_PWR, LOW); 
  digitalWrite(RELAY_CW, LOW); 
  digitalWrite(RELAY_CCW, LOW);

  Wire.begin();
  
  delay(200);
  sensorWorking = checkMagnet(); 
  if (sensorWorking) sensorWorking = readAS5600(rawSensorAngle); 
  
  if (!sensorWorking) {
    Serial.println("No communication with AS5600 encoder or missing magnet!");
  }

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  
  EEPROM.begin(512);
  EEPROM.get(0, calib);
  if(!calib.isValid || isnan(calib.zeroOffset)) {
    calib = {0.0, false};
  }

  WiFi.begin(ssid, password);
  
  server.on("/", [](){
    String html = String(FPSTR(index_html));
    html.replace("%TARGET%", String(targetAzimuth));
    server.send(200, "text/html", html); 
  });
  
  server.on("/set", [](){ 
    if(server.hasArg("az")) {
      targetAzimuth = server.arg("az").toInt(); 
      isMoving = true; 
    }
    server.send(200); 
  });
  
  server.on("/data", [](){ 
    String json = "{\"az\":" + String(currentAzimuth) + ", \"tg\":" + String(targetAzimuth) + ", \"sensor\":" + (sensorWorking ? "true" : "false") + ", \"motor\":\"" + motorStatus + "\"}";
    server.send(200, "application/json", json); 
  });
  
  server.on("/calib", [](){
    if (sensorWorking) {
      calib.zeroOffset = rawSensorAngle; 
      calib.isValid = true; 
      EEPROM.put(0, calib); 
      EEPROM.commit(); 
      Serial.println("[CALIBRATION] New zero offset saved: " + String(calib.zeroOffset));
    }
    server.send(200);
  });
  server.begin();
}

void loop() {
  server.handleClient();
  
  if (sensorWorking) {
    bool magnetPresent = checkMagnet();
    bool success = readAS5600(rawSensorAngle);
    
    if (!success || !magnetPresent) {
       sensorWorking = false; 
    } else {
      float heading = rawSensorAngle - calib.zeroOffset;
      if (heading < 0) heading += 360.0;
      if (heading >= 360.0) heading -= 360.0;
      
      currentAzimuth = (int)heading;
    }
  } else {
    if (millis() - lastSensorCheck > 3000) {
      if (checkMagnet() && readAS5600(rawSensorAngle)) {
        sensorWorking = true;
      }
      lastSensorCheck = millis();
    }
  }

  bool stopCW = (digitalRead(LIMIT_CW) == LOW);
  bool stopCCW = (digitalRead(LIMIT_CCW) == LOW);

  // --- MOTOR AND RELAY LOGIC ---
  if (!sensorWorking) {
    digitalWrite(RELAY_PWR, LOW);
    digitalWrite(RELAY_CW, LOW); 
    digitalWrite(RELAY_CCW, LOW);
    motorStatus = "STOP";
    isMoving = false; 
  } 
  else if (isMoving) {
    if (abs(currentAzimuth - targetAzimuth) > deadZone) {
      int diff = targetAzimuth - currentAzimuth;
      if (diff < -180) diff += 360;
      if (diff > 180) diff -= 360;

      if (diff > 0 && !stopCW) {
        digitalWrite(RELAY_PWR, HIGH); 
        digitalWrite(RELAY_CW, HIGH); 
        digitalWrite(RELAY_CCW, LOW);
        motorStatus = "CW";
      } else if (diff < 0 && !stopCCW) {
        digitalWrite(RELAY_PWR, HIGH); 
        digitalWrite(RELAY_CW, LOW); 
        digitalWrite(RELAY_CCW, HIGH);
        motorStatus = "CCW";
      } else {
        digitalWrite(RELAY_PWR, LOW);
        digitalWrite(RELAY_CW, LOW); 
        digitalWrite(RELAY_CCW, LOW);
        motorStatus = "STOP";
        isMoving = false; 
      }
    } else {
      digitalWrite(RELAY_PWR, LOW);
      digitalWrite(RELAY_CW, LOW); 
      digitalWrite(RELAY_CCW, LOW);
      motorStatus = "STOP";
      isMoving = false; 
    }
  } else {
    digitalWrite(RELAY_PWR, LOW);
    digitalWrite(RELAY_CW, LOW); 
    digitalWrite(RELAY_CCW, LOW);
    motorStatus = "STOP";
  }

  static unsigned long lastOLED = 0;
  if(millis() - lastOLED > 500) {
    display.clearDisplay();
    display.setTextSize(1); display.setTextColor(WHITE);
    
    if (WiFi.status() == WL_CONNECTED) {
      display.setCursor(0,0); display.print(WiFi.localIP());
    } else {
      display.setCursor(0,0); display.print("Connecting Wi-Fi...");
    }
    
    if (!sensorWorking) {
      display.setTextSize(2); display.setCursor(0, 25); display.print("AS5600 ERROR");
      display.setTextSize(1); display.setCursor(0, 45); display.print("Check magnet!");
    } else {
      display.setCursor(0,15); display.print("TARGET: "); display.print(targetAzimuth);
      display.setTextSize(2); display.setCursor(0,35);
      display.print("AZ: "); display.print(currentAzimuth);
      if(stopCW || stopCCW) { display.setTextSize(1); display.setCursor(80,45); display.print("LIMIT!"); }
    }
    display.display();
    lastOLED = millis();
  }
}
