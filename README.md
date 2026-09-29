#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <HX711.h>

// =====================================================
// LCD
// =====================================================
LiquidCrystal_I2C lcd(0x27, 16, 2);

// =====================================================
// HX711 CONNECTIONS
// BUS 601 -> DT D2, SCK D3
// BUS 602 -> DT D4, SCK D5
// BUS 603 -> DT D11, SCK D12
// =====================================================

HX711 scale601;
HX711 scale602;
HX711 scale603;

#define DT601  2
#define SCK601 3

#define DT602  4
#define SCK602 5

#define DT603  11
#define SCK603 12

// =====================================================
// BUTTON
// =====================================================
#define BUTTON_PIN A0

// =====================================================
// LEDs
// =====================================================
#define RED_LED    8
#define YELLOW_LED 9
#define GREEN_LED  10

// =====================================================
// LOAD CELL CALIBRATION
// Wokwi 50 kg load cell
// Approximately 0 raw -> 0 kg
// Approximately 21000 raw -> 50 kg
// =====================================================
#define MAX_RAW 21000.0
#define MAX_KG  50.0

// =====================================================
// BUS DATA
// =====================================================
int occupancy601 = 0;
int occupancy602 = 0;
int occupancy603 = 0;

float load601 = 0;
float load602 = 0;
float load603 = 0;

// =====================================================
// ETA
// =====================================================
int eta601 = 4;
int eta602 = 6;
int eta603 = 4;

// =====================================================
// DISPLAY MODES
//
// 0 -> Recommended Bus 1
// 1 -> Recommended Bus 2
// 2 -> Recommended Bus 3
// 3 -> Bus 601
// 4 -> Bus 602
// 5 -> Bus 603
// =====================================================
int displayMode = 0;

bool lastButtonState = HIGH;

// =====================================================
// SENSOR STATUS
// =====================================================
bool ready601 = false;
bool ready602 = false;
bool ready603 = false;

// =====================================================
// TIMERS
// =====================================================
unsigned long lastSerialTime = 0;

// =====================================================
// SETUP
// =====================================================
void setup() {

  Serial.begin(9600);

  // LCD
  lcd.init();
  lcd.backlight();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SMART RIDE");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  delay(1500);

  // Pins
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  pinMode(RED_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  // HX711 initialization
  scale601.begin(DT601, SCK601);
  scale602.begin(DT602, SCK602);
  scale603.begin(DT603, SCK603);

  // ---------------------------------------------------
  // NO TARE HERE
  // The Wokwi load-cell raw values are directly mapped
  // to 0-50 kg.
  // ---------------------------------------------------

  ready601 = scale601.is_ready();
  ready602 = scale602.is_ready();
  ready603 = scale603.is_ready();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SMART RIDE");
  lcd.setCursor(0, 1);
  lcd.print("Ready!");
  delay(1000);

  // Show first recommendation
  showRecommendedBus(1);

  // ---------------------------------------------------
  // SENSOR STATUS
  // Do NOT show one-time READY/NOT READY status.
  // Sensors are checked continuously inside loop().
  // ---------------------------------------------------
  Serial.println();
  Serial.println("================================");
  Serial.println("SMART RIDE SYSTEM");
  Serial.println("================================");
  Serial.println("HX711 STATUS:");
  Serial.println("Sensors monitored continuously");
  Serial.println("during operation.");
  Serial.println();
}

// =====================================================
// LOOP
// =====================================================
void loop() {

  // ===================================================
  // READ BUS 601
  // ===================================================
  if (scale601.is_ready()) {

    long raw601 = scale601.get_value(3);

    if (raw601 < 0)
      raw601 = 0;

    load601 = ((float)raw601 / MAX_RAW) * MAX_KG;

    occupancy601 = round((load601 / MAX_KG) * 100.0);

    occupancy601 = constrain(occupancy601, 0, 100);

    ready601 = true;
  }

  // ===================================================
  // READ BUS 602
  // ===================================================
  if (scale602.is_ready()) {

    long raw602 = scale602.get_value(3);

    if (raw602 < 0)
      raw602 = 0;

    load602 = ((float)raw602 / MAX_RAW) * MAX_KG;

    occupancy602 = round((load602 / MAX_KG) * 100.0);

    occupancy602 = constrain(occupancy602, 0, 100);

    ready602 = true;
  }

  // ===================================================
  // READ BUS 603
  // ===================================================
  if (scale603.is_ready()) {

    long raw603 = scale603.get_value(3);

    if (raw603 < 0)
      raw603 = 0;

    load603 = ((float)raw603 / MAX_RAW) * MAX_KG;

    occupancy603 = round((load603 / MAX_KG) * 100.0);

    occupancy603 = constrain(occupancy603, 0, 100);

    ready603 = true;
  }

  // ===================================================
  // BUTTON
  // ===================================================
  bool currentButtonState = digitalRead(BUTTON_PIN);

  if (lastButtonState == HIGH && currentButtonState == LOW) {

    displayMode++;

    if (displayMode > 5)
      displayMode = 0;

    delay(200);
  }

  lastButtonState = currentButtonState;

  // ===================================================
  // DISPLAY
  // ===================================================

  if (displayMode == 0) {

    showRecommendedBus(1);
    updateRecommendationLED();

  }

  else if (displayMode == 1) {

    showRecommendedBus(2);
    updateRecommendationLED();

  }

  else if (displayMode == 2) {

    showRecommendedBus(3);
    updateRecommendationLED();

  }

  else if (displayMode == 3) {

    showIndividualBus(601, occupancy601, eta601);
    updateLED(occupancy601);

  }

  else if (displayMode == 4) {

    showIndividualBus(602, occupancy602, eta602);
    updateLED(occupancy602);

  }

  else if (displayMode == 5) {

    showIndividualBus(603, occupancy603, eta603);
    updateLED(occupancy603);
  }

  // ===================================================
  // SERIAL MONITOR
  // ===================================================
  if (millis() - lastSerialTime >= 1000) {

    lastSerialTime = millis();

    Serial.println("--------------------------------");

    Serial.print("BUS 601 | ");
    Serial.print(load601, 2);
    Serial.print(" kg | ");
    Serial.print(occupancy601);
    Serial.println("%");

    Serial.print("BUS 602 | ");
    Serial.print(load602, 2);
    Serial.print(" kg | ");
    Serial.print(occupancy602);
    Serial.println("%");

    Serial.print("BUS 603 | ");
    Serial.print(load603, 2);
    Serial.print(" kg | ");
    Serial.print(occupancy603);
    Serial.println("%");

    Serial.println("--------------------------------");

    printRecommendation();
  }

  delay(50);
}

// =====================================================
// SHOW INDIVIDUAL BUS
// =====================================================
void showIndividualBus(int busNumber, int occupancy, int eta) {

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("Bus ");
  lcd.print(busNumber);

  lcd.setCursor(0, 1);
  lcd.print(occupancy);
  lcd.print("% ");
  lcd.print("ETA:");
  lcd.print(eta);
  lcd.print(" min");
}

// =====================================================
// SHOW RECOMMENDED BUS
// =====================================================
void showRecommendedBus(int position) {

  int busNumbers[3] = {601, 602, 603};

  int occupancies[3] = {
    occupancy601,
    occupancy602,
    occupancy603
  };

  // Sort buses according to occupancy
  for (int i = 0; i < 2; i++) {

    for (int j = i + 1; j < 3; j++) {

      if (occupancies[j] < occupancies[i]) {

        int tempOcc = occupancies[i];
        occupancies[i] = occupancies[j];
        occupancies[j] = tempOcc;

        int tempBus = busNumbers[i];
        busNumbers[i] = busNumbers[j];
        busNumbers[j] = tempBus;
      }
    }
  }

  int selectedBus = busNumbers[position - 1];
  int selectedOccupancy = occupancies[position - 1];

  int selectedETA = 4;

  if (selectedBus == 602)
    selectedETA = 6;

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("Recommended Bus");
  lcd.print(position);

  lcd.setCursor(0, 1);
  lcd.print("Bus ");
  lcd.print(selectedBus);
  lcd.print(" ");
  lcd.print(selectedOccupancy);
  lcd.print("%");

  // Serial recommendation
  Serial.print("Recommended Bus ");
  Serial.print(position);
  Serial.print(" -> Bus ");
  Serial.print(selectedBus);
  Serial.print(" | ");
  Serial.print(selectedOccupancy);
  Serial.print("% | ETA ");
  Serial.print(selectedETA);
  Serial.println(" min");
}

// =====================================================
// UPDATE LED FOR INDIVIDUAL BUS
// =====================================================
void updateLED(int occupancy) {

  digitalWrite(RED_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(GREEN_LED, LOW);

  // LOW CROWD
  if (occupancy < 40) {

    digitalWrite(GREEN_LED, HIGH);
  }

  // MEDIUM CROWD
  else if (occupancy < 70) {

    digitalWrite(YELLOW_LED, HIGH);
  }

  // HIGH CROWD
  else {

    digitalWrite(RED_LED, HIGH);
  }
}

// =====================================================
// UPDATE LED FOR RECOMMENDATION
// Uses occupancy of the FIRST recommended bus
// =====================================================
void updateRecommendationLED() {

  int occupancies[3] = {
    occupancy601,
    occupancy602,
    occupancy603
  };

  int lowestOccupancy = occupancies[0];

  if (occupancies[1] < lowestOccupancy)
    lowestOccupancy = occupancies[1];

  if (occupancies[2] < lowestOccupancy)
    lowestOccupancy = occupancies[2];

  updateLED(lowestOccupancy);
}

// =====================================================
// PRINT RECOMMENDATIONS
// =====================================================
void printRecommendation() {

  int busNumbers[3] = {601, 602, 603};

  int occupancies[3] = {
    occupancy601,
    occupancy602,
    occupancy603
  };

  // Sort according to occupancy
  for (int i = 0; i < 2; i++) {

    for (int j = i + 1; j < 3; j++) {

      if (occupancies[j] < occupancies[i]) {

        int tempOcc = occupancies[i];
        occupancies[i] = occupancies[j];
        occupancies[j] = tempOcc;

        int tempBus = busNumbers[i];
        busNumbers[i] = busNumbers[j];
        busNumbers[j] = tempBus;
      }
    }
  }

  Serial.println("CURRENT RECOMMENDATION:");

  for (int i = 0; i < 3; i++) {

    int eta = 4;

    if (busNumbers[i] == 602)
      eta = 6;

    Serial.print("Recommended Bus ");
    Serial.print(i + 1);
    Serial.print(" -> Bus ");
    Serial.print(busNumbers[i]);
    Serial.print(" | ");
    Serial.print(occupancies[i]);
    Serial.print("% | ETA ");
    Serial.print(eta);
    Serial.println(" min");
  }
}
