#include <LiquidCrystal_I2C.h>
#include <AltSoftSerial.h>
#include <TinyGPS++.h>
#include <SoftwareSerial.h>
#include <Wire.h>

// I2C LCD configuration
LiquidCrystal_I2C lcd(0x27, 16, 2); // Address 0x27, 16 columns and 2 rows

// Emergency phone number with country code
const String EMERGENCY_PHONE = "+917338287704";

// GSM Module RX pin to Arduino 3
// GSM Module TX pin to Arduino 2
#define rxPin 2
#define txPin 3
SoftwareSerial sim800(rxPin, txPin);

// GPS Module RX pin to Arduino 9
// GPS Module TX pin to Arduino 8
AltSoftSerial neogps;
TinyGPSPlus gps;

String sms_status, sender_number, received_date, msg;
String latitude, longitude;

#define BUZZER 12
#define BUTTON 11

#define xPin A1
#define yPin A2
#define zPin A3

byte updateflag;

int xaxis = 0, yaxis = 0, zaxis = 0;
int deltx = 0, delty = 0, deltz = 0;
int vibration = 2, devibrate = 75;
int magnitude = 0;
int sensitivity = 20;
double angle;
boolean impact_detected = false;

unsigned long time1;
unsigned long impact_time;
unsigned long alert_delay = 30000; // 30 seconds

void setup() {
  Serial.begin(9600); // Initialize serial communication
  sim800.begin(9600); // Initialize GSM module
  neogps.begin(9600); // Initialize GPS module
  
  pinMode(BUZZER, OUTPUT);     // Buzzer pin as output
  pinMode(BUTTON, INPUT_PULLUP); // Button pin as input
  
  lcd.init(); // Initialize LCD
  lcd.backlight(); // Turn on backlight
  lcd.clear(); // Clear LCD screen
  
  sms_status = "";
  sender_number = "";
  received_date = "";
  msg = "";
  
  sim800.println("AT"); // Check GSM Module
  delay(1000);
  sim800.println("ATE1"); // Echo ON
  delay(1000);
  sim800.println("AT+CPIN?"); // Check SIM ready
  delay(1000);
  sim800.println("AT+CMGF=1"); // SMS text mode
  delay(1000);
  sim800.println("AT+CNMI=1,1,0,0,0"); // SMS received format
  delay(1000);
  
  time1 = micros(); // Initialize time1
  xaxis = analogRead(xPin); // Read X-axis value
  yaxis = analogRead(yPin); // Read Y-axis value
  zaxis = analogRead(zPin); // Read Z-axis value
}

void loop() {
  // Call impact routine every 2mS
  if (micros() - time1 > 1999) {
    Impact();
  }
  
  if (updateflag > 0) {
    updateflag = 0;
    Serial.println("Impact detected!!");
    Serial.print("Magnitude: ");
    Serial.println(magnitude);

    getGps();
    digitalWrite(BUZZER, HIGH);
    impact_detected = true;
    impact_time = millis();
    
    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("Crash Detected");
    lcd.setCursor(0,1);
    lcd.print("Magnitude: " + String(magnitude));
  }
  
  if (impact_detected) {
    if (millis() - impact_time >= alert_delay) {
      digitalWrite(BUZZER, LOW);
      makeCall();
      delay(1000);
      sendAlert();
      impact_detected = false;
      impact_time = 0;
    }
  }
  
  if (digitalRead(BUTTON) == LOW) {
    delay(200);
    digitalWrite(BUZZER, LOW);
    impact_detected = false;
    impact_time = 0;
  }
  
  while (sim800.available()) {
    parseData(sim800.readString());
  }
  
  while (Serial.available()) {
    sim800.println(Serial.readString());
  }
}

void Impact() {
  time1 = micros(); // Reset time value
  int oldx = xaxis; // Store previous axis readings for comparison
  int oldy = yaxis;
  int oldz = zaxis;

  xaxis = analogRead(xPin); // Read X-axis value
  yaxis = analogRead(yPin); // Read Y-axis value
  zaxis = analogRead(zPin); // Read Z-axis value
  
  vibration--;
  if (vibration < 0) vibration = 0;
  
  if (vibration > 0) return;
  
  deltx = xaxis - oldx;
  delty = yaxis - oldy;
  deltz = zaxis - oldz;
  
  magnitude = sqrt(sq(deltx) + sq(delty) + sq(deltz));
  
  if (magnitude >= sensitivity) {
    updateflag = 1;
    vibration = devibrate;
  } else {
    magnitude = 0;
  }
}

void parseData(String buff) {
  Serial.println(buff);

  unsigned int len, index;
  
  index = buff.indexOf("\r");
  buff.remove(0, index+2);
  buff.trim();
  
  if (buff != "OK") {
    index = buff.indexOf(":");
    String cmd = buff.substring(0, index);
    cmd.trim();
    
    buff.remove(0, index+2);
    
    if (cmd == "+CMTI") {
      index = buff.indexOf(",");
      String temp = buff.substring(index+1, buff.length());
      temp = "AT+CMGR=" + temp + "\r";
      sim800.println(temp);
    } else if (cmd == "+CMGR") {
      if (buff.indexOf(EMERGENCY_PHONE) > 1) {
        buff.toLowerCase();
        if (buff.indexOf("get gps") > 1) {
          getGps();
          String sms_data = "GPS Location Data\r";
          sms_data += "http://maps.google.com/maps?q=loc:";
          sms_data += latitude + "," + longitude;
          sendSms(sms_data);
        }
      }
    }
  }
}

void getGps() {
  boolean newData = false;
  
  for (unsigned long start = millis(); millis() - start < 2000;) {
    while (neogps.available()) {
      if (gps.encode(neogps.read())) {
        newData = true;
        break;
      }
    }
  }
  
  if (newData) {
    latitude = String(gps.location.lat(), 6);
    longitude = String(gps.location.lng(), 6);
    newData = false;
  } else {
    Serial.println("No GPS data is available");
    latitude = "";
    longitude = "";
  }

  Serial.print("Latitude= ");
  Serial.println(latitude);
  Serial.print("Longitude= ");
  Serial.println(longitude);
}

void sendAlert() {
  String sms_data = "Accident Alert!!\r";
  sms_data += "http://maps.google.com/maps?q=loc:";
  sms_data += latitude + "," + longitude;
  sendSms(sms_data);
}

void makeCall() {
  Serial.println("Calling....");
  sim800.println("ATD" + EMERGENCY_PHONE + ";");
  delay(20000); // 20 sec delay
  sim800.println("ATH");
  delay(1000); // 1 sec delay
}

void sendSms(String text) {
  sim800.print("AT+CMGF=1\r");
  delay(1000);
  sim800.print("AT+CMGS=\"" + EMERGENCY_PHONE + "\"\r");
  delay(1000);
  sim800.print(text);
  delay(100);
  sim800.write(0x1A); // ASCII code for Ctrl-Z
  delay(1000);
  Serial.println("SMS Sent Successfully.");
}
