# OvOn
An alarm and timer for an oven
#include <Adafruit_MAX31856.h>
//Adafruit_MAX31856 maxthermo = Adafruit_MAX31856(10, 11, 12, 13);
Adafruit_MAX31856 maxthermo = Adafruit_MAX31856(10);
const int buzzer = 9;     // buzzer pin
const int dataPin = 8;    // 74HC595 pin 8 DS
const int latchPin = A0;  // 74HC595 pin 9 STCP
const int clockPin = A1;  // 74HC595 pin 10 SHCP
const int digit0 = A2;    // 7-Segment pin D4
const int digit1 = A3;    // 7-Segment pin D3
const int digit2 = A4;    // 7-Segment pin D2
const int digit3 = A5;    // 7-Segment pin D1

// {0=timer, 1=temp, 2=set, 3=off, 4=down, 5=up}
int buttonPins[] = { 3, 4, 2, 5, 6, 7 };
boolean buttonStates[] = { false, false, false, false, false, false };
boolean prevButtonStates[] = { false, false, false, false, false, false };
//boolean isTemp;
/*     1
    6     2
       7
    5     3
       4

     d1   s1    s6    d2   d3   s2
     |    |     |     |    |    |
     
     |    |     |     |    |    |
     s5   s4    dec   s3   s7   d4
*/
byte table[] = {
  0x5F,                                                     // = 0
  0x44,                                                     // = 1
  0x9D,                                                     // = 2
  0xD5,                                                     // = 3
  0xC6,                                                     // = 4
  0xD3,                                                     // = 5
  0xDB,                                                     // = 6
  0x45,                                                     // = 7
  0xDF,                                                     // = 8
  0xC7,                                                     // = 9
  0x00                                                      // blank
                                                            // 0x80,  // -
};                                                          //Hex shown
byte digitDP = 32;                                          // 0x20 - adds this to digit to show decimal point
byte controlDigits[] = { digit0, digit1, digit2, digit3 };  // pins to turn off & on digits
byte displayDigits[] = { 0, 0, 0, 0, 0 };                   // ie: { 1, 0, 7, 13, 0} == d701 (all values from table array)
unsigned long onTime = 0;                                   // tracks time
bool switchView = false;                                    // switch between HexCounter (table array) and RawDisplay (raw bytes)
                                                            //   false = HexCounter
                                                            //   true = RawDisplay
unsigned int counter = 0;                                   // RawDisplay counter
int digitDelay = 50;                                        // delay between incrementing digits (ms)
int brightness = 90;                                        // valid range of 0-100, 100=brightest
unsigned int ShowSegCount = 250;                            // number of RawDisplay loops before switching again
// for thermocouple
unsigned long start;
int interval;
// for timer
unsigned long timer;
long minute;
boolean startTimer = false;

float preHeatTemp;   // the temp being changed by up down buttons
String temperature;  // input of thermocouple
float count = 200;   // number in setTemp()
float time = 2;
int state = 0;
float intTemp;

void setup() {
  Serial.begin(9600);
  //Thermocouple setup
  while (!Serial) delay(10);
  Serial.println("MAX31856 thermocouple test");
  if (!maxthermo.begin()) {
    Serial.println("Could not initialize thermocouple.");
    while (1) delay(10);
  }
  maxthermo.setThermocoupleType(MAX31856_TCTYPE_K);
  maxthermo.setConversionMode(MAX31856_ONESHOT_NOWAIT);

  // 4 digit display setup
  pinMode(latchPin, OUTPUT);
  pinMode(clockPin, OUTPUT);
  pinMode(dataPin, OUTPUT);
  for (int x = 0; x < 4; x++) {
    pinMode(controlDigits[x], OUTPUT);
    digitalWrite(controlDigits[x], LOW);  // Turns off the digit
  }

  pinMode(buzzer, OUTPUT);

  for (int i = 0; i < sizeof(buttonPins) / sizeof(int); i++) {
    pinMode(buttonPins[i], INPUT);
  }

  //isTemp = true;

  start = millis();
  interval = 500;

  timer = millis();
  minute = 60000;
}

void loop() {

  // termocouple code
  if (millis() - start >= interval) {
    maxthermo.triggerOneShot();
    //delay(500);
    start = millis();
  }
  intTemp = maxthermo.readThermocoupleTemperature() * 1.8 + 32;
  // check for conversion complete and read temperature
  if (maxthermo.conversionComplete()) {
    temperature = String(maxthermo.readThermocoupleTemperature() * 1.8 + 32);
  } else {
    temperature = "0";
  }
  // Serial.println(temperature);

  // 4 digit display code
  DisplaySegments();
  delayMicroseconds(1638 * ((100 - brightness) / 10));  // brightness

  for (int i = 0; i < sizeof(buttonPins) / sizeof(int); i++) {
    prevButtonStates[i] = buttonStates[i];
    buttonStates[i] = debouncedReading(buttonPins[i], prevButtonStates[i]);
  }

  if (buttonStates[0] == true && prevButtonStates[0] == false) {
    state = 1;
    for (int i = 0; i < 4; i++) {
      digitalWrite(controlDigits[i], LOW);
    }
  }

  if (buttonStates[1] == true && prevButtonStates[1] == false) {
    state = 3;
    startTimer = false;
    
  }

  if (buttonStates[3] == true && prevButtonStates[3] == false) {
    state = 0;
    for (int i = 0; i < 4; i++) {
      digitalWrite(controlDigits[i], LOW);
    }
  }

  switch (state) {
      // off
    case 0:
      offDisplay();
      break;
      // TEMP
    case 1:
      setTemp();
      break;
      // TEMP/SET
    case 2:
      temp();
      if (intTemp >= preHeatTemp) {
        tone(buzzer, 500, 100);
        for (int i = 0; i < 4; i++) {
      digitalWrite(controlDigits[i], LOW);
    }
      }
      break;
      // TIME
    case 3:
   // if (startTimer == false){
      setTimer();
        if (buttonStates[2] == true && prevButtonStates[2] == false) {
          startTimer = true;
         // digitalWrite(controlDigits[2], LOW);
        }
  

      if (startTimer == true) {
        
    if (millis() - timer >= minute) {
      time--;
      timer = millis();
      Serial.println(time);
    }
    if (time <= 0) {
      tone(buzzer, 1000, 300);
      time = 0;
     
    }
  }
      //Serial.println(timer);
      break;

  }
}




void DisplaySegments() {
  for (int x = 0; x < 4; x++) {
    for (int j = 0; j < 4; j++) {
      digitalWrite(controlDigits[j], LOW);  // turn off digits
    }
    digitalWrite(latchPin, LOW);
    if (bitRead(displayDigits[4], x) == 1) {
      // raw byte value is sent to shift register
      shiftOut(dataPin, clockPin, MSBFIRST, displayDigits[x]);
    } else {
      // table array value is sent to the shift register
      shiftOut(dataPin, clockPin, MSBFIRST, table[displayDigits[x]]);
    }

    digitalWrite(latchPin, HIGH);
    digitalWrite(controlDigits[x], HIGH);  // turn on one digit
    delay(1);                              // 1 or 2 is ok
  }
  for (int j = 0; j < 4; j++) {
    digitalWrite(controlDigits[j], LOW);  // turn off digits
  }
}




void temp() {
  byte ones;
  byte tens;
  byte hundreds;
  static boolean alarmTriggered = false;
  int periodIndex = temperature.indexOf('.');
  if (temperature != "0") {
    ones = (temperature.substring(periodIndex - 1, periodIndex)).toInt();
    tens = (temperature.substring(periodIndex - 2, periodIndex - 1)).toInt();
    displayDigits[0] = ones;
    displayDigits[1] = tens;
    if (intTemp < 100) {
      hundreds = "0";
    }
    if (intTemp >= 100) {
      hundreds = (temperature.substring(periodIndex - 3, periodIndex - 2)).toInt();
    }

    displayDigits[2] = hundreds;
    digitalWrite(controlDigits[3], LOW);
  } else {
    for (int d = 0; d < 4; d++) {
      digitalWrite(controlDigits[d], LOW);
    }
  }
  // Set digitSwitch option
  displayDigits[4] = B1000;

  if ((displayDigits[0] == 0) && (displayDigits[1] == 0) && (displayDigits[2] == 0)) {
    switchView = !switchView;
    for (int x = 0; x < 5; x++) {
      displayDigits[x] = 0;  // Reset array
    }
    displayDigits[4] = B0000;
  }
}




void setTemp() {
  if (buttonStates[2] == true && prevButtonStates[2] == false) {
    preHeatTemp = count;
    state = 2;
  }
  byte ones;
  byte tens;
  byte hundreds;

  String display;
  char periodIndex;
  if (buttonStates[5] == true && prevButtonStates[5] == false) {
    count += 25;
  }

  if (buttonStates[4] == true && prevButtonStates[4] == false) {
    count -= 25;
  }

  display = String(count);
  //Serial.println(display);
  periodIndex = display.indexOf('.');
  ones = (display.substring(periodIndex - 1, periodIndex)).toInt();
  tens = (display.substring(periodIndex - 2, periodIndex - 1)).toInt();

  if (count < 100) {
    hundreds = "0";
  }
  if (count >= 100) {
    hundreds = (display.substring(periodIndex - 3, periodIndex - 2)).toInt();
  }

  displayDigits[0] = ones;
  displayDigits[1] = tens;
  displayDigits[2] = hundreds;
  digitalWrite(controlDigits[3], LOW);

  displayDigits[4] = B1000;

  if ((displayDigits[0] == 0) && (displayDigits[1] == 0) && (displayDigits[2] == 0)) {
    switchView = !switchView;
    for (int x = 0; x < 5; x++) {
      displayDigits[x] = 0;  // Reset array
    }
    displayDigits[4] = B0000;
  }
}




void setTimer() {
  byte ones;
  byte tens;
  String display;
  char periodIndex;
  if (time < 58) {
    if (buttonStates[5] == true && prevButtonStates[5] == false) {
      time += 2;
    }
  }
  if (time > 1) {
    if (buttonStates[4] == true && prevButtonStates[4] == false) {
      time -= 1;
    }
  }

  display = String(time);
  Serial.println(time);
  periodIndex = display.indexOf('.');
  ones = (display.substring(periodIndex - 1, periodIndex)).toInt();
  if (time <= 9) {
    tens = "0";
  } else {
    tens = (display.substring(periodIndex - 2, periodIndex - 1)).toInt();
  }

  displayDigits[0] = ones;
  displayDigits[1] = tens;
  displayDigits[2] = "0";
  digitalWrite(controlDigits[2], LOW);
  digitalWrite(controlDigits[3], LOW);

  displayDigits[4] = B1000;

  if ((displayDigits[0] == 0) && (displayDigits[1] == 0) && (displayDigits[2] == 0)) {
    switchView = !switchView;
    for (int x = 0; x < 5; x++) {
      displayDigits[x] = 0;  // Reset array
    }
    displayDigits[4] = B0000;
  }
  noTone(buzzer);
}




void offDisplay() {
  // g, c, DP, d, e, b, f, a
  displayDigits[0] = B10001011;  // f
  displayDigits[1] = B10001011;  // f
  displayDigits[2] = B11011000;  // 0

  // Set digitSwitch option
  displayDigits[4] = B1111;

  if (counter < ShowSegCount) {
    counter++;
  } else {
    // Reset everything
    counter = 0;
    switchView = !switchView;
    for (int x = 0; x < 5; x++) { displayDigits[x] = 0; }  // Reset array
    displayDigits[4] = B0000;
  }
  noTone(buzzer);
}



boolean debouncedReading(int aButtonPin, boolean aPrevState) {
  boolean currentState = digitalRead(aButtonPin);
  if (currentState != aPrevState) {
    delay(15);
  }
  return currentState;
}
