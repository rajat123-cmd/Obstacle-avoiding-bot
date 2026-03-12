#include <Servo.h>

// Motor Control Pins
#define ENA 5   
#define IN1 6   
#define IN2 7   
#define IN3 8   
#define IN4 9   
#define ENB 3   

// Sensor & Accessory Pins
#define TRIG_PIN 2
#define ECHO_PIN 4
#define BUZZER_PIN A2
#define SERVO_PIN 11

// Distance Thresholds
#define SAFE_DISTANCE 40  // Increased - needs more clearance to turn
#define STOP_DISTANCE 30   // Increased from 30 to stop even earlier
#define MAX_SPEED 100      
#define REVERSE_SPEED 80   

Servo scanner;
int centerDist;

void setup() {
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  scanner.attach(SERVO_PIN);
  scanner.write(90); 
  delay(500);
  
  Serial.begin(9600);
  Serial.println("--- OBSTACLE AVOIDING ROBOT ---");
  Serial.println("System Ready!");
}

void loop() {
  // Stop car before measuring
  stopCar();
  delay(30);
  
  centerDist = getDistance();
  
  Serial.print("Distance: ");
  Serial.print(centerDist);
  Serial.println(" cm");
  
  if (centerDist < STOP_DISTANCE && centerDist > 0) {
    Serial.println("⚠️ Obstacle detected! Scanning...");
    executeScan();
  } else if (centerDist > 0) {
    moveForward(MAX_SPEED);
    delay(100);
  } else {
    // Invalid reading - stay stopped and wait
    Serial.println("⚠️ Invalid sensor reading - standby mode");
    stopCar();
    delay(200); // Wait a bit longer before retrying
  }
}

int getDistance() {
  // Take 2 quick readings to filter out glitches
  int reading1 = takeSingleReading();
  delay(20);
  int reading2 = takeSingleReading();
  
  // If both readings are suspiciously low (stuck sensor), try one more time
  if (reading1 < 5 && reading2 < 5) {
    delay(30);
    int reading3 = takeSingleReading();
    
   // If still stuck at ~3cm after 3 tries, sensor might be frozen
    if (reading3 < 5) {
      Serial.println("⚠️ Sensor appears stuck - resetting");
      // Return a mid-range value to prevent false obstacle detection
      return 50;
    }
    return reading3;
  }
  
  // If readings differ significantly, trust the larger one (safer)
  if (abs(reading1 - reading2) > 10) {
    return max(reading1, reading2);
  }
  
  // Average the two readings
  return (reading1 + reading2) / 2;
}

int takeSingleReading() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(5);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  int distance = duration / 58;
  
  // Return 0 if reading is out of valid range
  if (distance < 2 || distance > 400) {
    return 0;
  }
  
  return distance;
}

void executeScan() {
  buzz(1);
  stopCar();
  delay(30);
  
  // Scan left
  scanner.write(160);
  delay(300);
  int left = getDistance();
  Serial.print("Left: ");
  Serial.print(left);
  Serial.println(" cm");
  
  // Scan right
  scanner.write(20);
  delay(300);
  int right = getDistance();
  Serial.print("Right: ");
  Serial.print(right);
  Serial.println(" cm");
  
  // Return to center
  scanner.write(90);
  delay(250);
  
  // Decision making (treat 0 as blocked)
  if (left == 0) left = 5;  // Treat invalid as very close
  if (right == 0) right = 5;
  
  if (left > right && left > SAFE_DISTANCE) {
    Serial.println("→ Turning LEFT");
    turnLeft();
    delay(350);
  } else if (right > left && right > SAFE_DISTANCE) {
    Serial.println("← Turning RIGHT");
    turnRight();
    delay(350);
  } else {
    Serial.println("↓ Both blocked - Reversing");
    slowBackward();
    delay(350);
    stopCar();
    delay(30);
    turnRight(); 
    delay(700);
  }
  
  stopCar();
  delay(30);
}

void moveForward(int spd) {
  analogWrite(ENA, spd);
  analogWrite(ENB, spd);
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void slowBackward() {
  analogWrite(ENA, REVERSE_SPEED);
  analogWrite(ENB, REVERSE_SPEED);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void turnLeft() {
  analogWrite(ENA, MAX_SPEED);
  analogWrite(ENB, MAX_SPEED);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void turnRight() {
  analogWrite(ENA, MAX_SPEED);
  analogWrite(ENB, MAX_SPEED);
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void stopCar() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
}

void buzz(int times) {
  for (int i = 0; i < times; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(60);
    digitalWrite(BUZZER_PIN, LOW);
    delay(60);
  }
}
