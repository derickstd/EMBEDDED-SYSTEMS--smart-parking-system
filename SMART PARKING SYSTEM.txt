/*
 * Smart Parking System - AUTO DETECTION + BUTTON CONFIRMATION
 * Car must be detected by sensor AND button must be pressed to start timer
 */

#include <LiquidCrystal.h>

// Ultrasonic Sensor Pins
#define TRIG_PIN 9
#define ECHO_PIN 10

// LED Pins
#define GREEN_LED 2
#define YELLOW_LED 3
#define RED_LED 4

// Push Button Pin
#define BUTTON_PIN 7

// Buzzer Pin
#define BUZZER_PIN 8

// LCD Pins: RS, E, D4, D5, D6, D7
LiquidCrystal lcd(12, 11, 5, 4, 3, 2);

// System Parameters
#define DISTANCE_THRESHOLD 15
#define PARKING_TIME_LIMIT 10000
#define MEASUREMENT_DELAY 200
#define DEBOUNCE_DELAY 50

// Global Variables
long duration;
int distance;
bool slotOccupied = false;
bool carDetected = false;
bool alertActive = false;
unsigned long parkingStartTime = 0;
unsigned long lastButtonPress = 0;

int buttonState;
int lastButtonState = HIGH;

void setup() {
  Serial.begin(9600);
  Serial.println("=== Smart Parking System - Detection + Confirmation ===");
  Serial.println("Initializing...");
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(GREEN_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  digitalWrite(GREEN_LED, HIGH);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(RED_LED, LOW);
  
  lcd.begin(16, 2);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Smart Parking");
  lcd.setCursor(0, 1);
  lcd.print("System Ready...");
  
  tone(BUZZER_PIN, 1000, 200);
  delay(250);
  tone(BUZZER_PIN, 1500, 200);
  
  Serial.println("System Initialized");
  Serial.println("Waiting for car...");
  Serial.println("------------------------");
  
  delay(2000);
  updateDisplay();
}

void loop() {
  distance = measureDistance();
  bool currentCarDetection = (distance < DISTANCE_THRESHOLD && distance > 0);
  
  // Car detection logic
  if (currentCarDetection && !carDetected && !slotOccupied) {
    // New car detected!
    carDetected = true;
    tone(BUZZER_PIN, 800, 150);
    Serial.println("\n*** CAR DETECTED IN PARKING AREA ***");
    updateDisplay();
  } else if (!currentCarDetection && carDetected && !slotOccupied) {
    // Car left before confirming
    carDetected = false;
    Serial.println("Car left without confirming parking");
    updateDisplay();
  }
  
  // Button handling
  int reading = digitalRead(BUTTON_PIN);
  
  if (reading != lastButtonState) {
    lastButtonPress = millis();
  }
  
  if ((millis() - lastButtonPress) > DEBOUNCE_DELAY) {
    if (reading != buttonState) {
      buttonState = reading;
      
      if (buttonState == LOW) {
        handleButtonPress();
      }
    }
  }
  
  lastButtonState = reading;
  
  // Update system status
  if (slotOccupied) {
    checkParkingTime();
    
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(RED_LED, HIGH);
    
    if (alertActive) {
      blinkYellowLED();
      buzzerAlertPattern();
    } else {
      digitalWrite(YELLOW_LED, LOW);
      noTone(BUZZER_PIN);
    }
    
    // Check if car left without pressing button
    if (!currentCarDetection) {
      Serial.println("\n*** WARNING: Car left without checkout! ***");
      slotOccupied = false;
      carDetected = false;
      alertActive = false;
      tone(BUZZER_PIN, 500, 500);
      updateDisplay();
    }
    
  } else {
    digitalWrite(GREEN_LED, HIGH);
    digitalWrite(RED_LED, LOW);
    digitalWrite(YELLOW_LED, LOW);
    noTone(BUZZER_PIN);
  }
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.print(" cm | Car Detected: ");
  Serial.print(carDetected ? "YES" : "NO");
  Serial.print(" | Slot Occupied: ");
  Serial.println(slotOccupied ? "YES" : "NO");
  
  delay(MEASUREMENT_DELAY);
}

void handleButtonPress() {
  if (carDetected && !slotOccupied) {
    // Start parking timer
    slotOccupied = true;
    parkingStartTime = millis();
    alertActive = false;
    
    tone(BUZZER_PIN, 1200, 300);
    
    Serial.println("\n*** PARKING CONFIRMED - TIMER STARTED ***");
    updateDisplay();
    
  } else if (slotOccupied) {
    // Checkout
    slotOccupied = false;
    carDetected = false;
    alertActive = false;
    
    tone(BUZZER_PIN, 1500, 150);
    delay(200);
    tone(BUZZER_PIN, 1500, 150);
    
    Serial.println("\n*** CHECKOUT COMPLETE - SLOT NOW AVAILABLE ***");
    updateDisplay();
    
  } else if (!carDetected) {
    // Error - no car detected
    tone(BUZZER_PIN, 500, 300);
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("ERROR!");
    lcd.setCursor(0, 1);
    lcd.print("No Car Detected");
    Serial.println("ERROR: No car in parking area!");
    delay(2000);
    updateDisplay();
  }
}

void checkParkingTime() {
  unsigned long currentTime = millis();
  unsigned long elapsedTime = currentTime - parkingStartTime;
  
  long remainingTime = PARKING_TIME_LIMIT - elapsedTime;
  
  if (remainingTime <= 0 && !alertActive) {
    alertActive = true;
    activateAlert();
  }
  
  if (!alertActive && slotOccupied) {
    lcd.setCursor(0, 1);
    lcd.print("Time: ");
    lcd.print(remainingTime / 1000);
    lcd.print("s left  ");
  }
}

int measureDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  duration = pulseIn(ECHO_PIN, HIGH, 30000);
  int dist = duration * 0.0343 / 2;
  
  if (dist == 0 || dist > 400) {
    return 999;
  }
  
  return dist;
}

void updateDisplay() {
  lcd.clear();
  lcd.setCursor(0, 0);
  
  if (!slotOccupied && !carDetected) {
    lcd.print("Parking Slot:");
    lcd.setCursor(0, 1);
    lcd.print("*** FREE ***");
    
  } else if (carDetected && !slotOccupied) {
    lcd.print("Car Detected!");
    lcd.setCursor(0, 1);
    lcd.print("Press to Start");
    
  } else if (slotOccupied) {
    lcd.print("Slot: OCCUPIED");
    lcd.setCursor(0, 1);
    lcd.print("Timer Running..");
  }
}

void activateAlert() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("*** ALERT! ***");
  lcd.setCursor(0, 1);
  lcd.print("TIME EXCEEDED!");
  
  Serial.println("\n*** ALERT: PARKING TIME EXCEEDED! ***");
}

void blinkYellowLED() {
  static unsigned long lastBlink = 0;
  static bool ledState = false;
  
  if (millis() - lastBlink >= 500) {
    ledState = !ledState;
    digitalWrite(YELLOW_LED, ledState);
    lastBlink = millis();
  }
}

void buzzerAlertPattern() {
  static unsigned long lastBeep = 0;
  static bool beepState = false;
  
  unsigned long currentMillis = millis();
  
  if (!beepState && (currentMillis - lastBeep >= 300)) {
    tone(BUZZER_PIN, 2000);
    beepState = true;
    lastBeep = currentMillis;
  } 
  else if (beepState && (currentMillis - lastBeep >= 200)) {
    noTone(BUZZER_PIN);
    beepState = false;
    lastBeep = currentMillis;
  }
}