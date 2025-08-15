# Code-for-Robotic-Hand-with-Arduino-UNO-Nano-MEGA-
Code for Robotic Hand-with Arduino (UNO/Nano/MEGA) = Features  Type simple serial commands (e.g., OPEN, CLOSE, PINCH, F2 120)  Smooth motion (easing)  Per-finger min/max calibration



#include <Servo.h>

// === CONFIG ===
const uint8_t SERVO_PINS[5] = {3, 5, 6, 9, 10}; // Thumb, Index, Middle, Ring, Pinky
// Set safe mechanical limits (tune these for your hand!)
int SERVO_MIN[5] = {20, 10, 10, 15, 20};
int SERVO_MAX[5] = {150, 170, 170, 165, 160};

// Motion
const uint8_t STEP_DEG = 2;     // smaller = smoother, slower
const uint16_t STEP_DELAY = 10; // ms per step

Servo servos[5];
int currentPos[5]; // live positions in degrees

// === HELPERS ===
int clampInt(int v, int lo, int hi){ return v < lo ? lo : (v > hi ? hi : v); }

void moveFingerTo(uint8_t i, int targetDeg){
  targetDeg = clampInt(targetDeg, SERVO_MIN[i], SERVO_MAX[i]);
  int dir = (targetDeg > currentPos[i]) ? 1 : -1;
  while(currentPos[i] != targetDeg){
    currentPos[i] += dir * min(STEP_DEG, abs(targetDeg - currentPos[i]));
    servos[i].write(currentPos[i]);
    delay(STEP_DELAY);
  }
}

void moveAllTo(const int targets[5]){
  // step all together for coordinated motion
  bool moving = true;
  int tgt[5];
  for(int i=0;i<5;i++) tgt[i] = clampInt(targets[i], SERVO_MIN[i], SERVO_MAX[i]);
  while(moving){
    moving = false;
    for(int i=0;i<5;i++){
      if(currentPos[i] != tgt[i]){
        int dir = (tgt[i] > currentPos[i]) ? 1 : -1;
        currentPos[i] += dir * min(STEP_DEG, abs(tgt[i] - currentPos[i]));
        servos[i].write(currentPos[i]);
        moving = true;
      }
    }
    delay(STEP_DELAY);
  }
}

// === PRESETS ===
void poseOpen(){
  int t[5] = {SERVO_MIN[0], SERVO_MIN[1], SERVO_MIN[2], SERVO_MIN[3], SERVO_MIN[4]};
  moveAllTo(t);
}
void poseClosed(){
  int t[5] = {SERVO_MAX[0], SERVO_MAX[1], SERVO_MAX[2], SERVO_MAX[3], SERVO_MAX[4]};
  moveAllTo(t);
}
void posePoint(){ // index extended
  int t[5] = {SERVO_MAX[0], SERVO_MIN[1], SERVO_MAX[2], SERVO_MAX[3], SERVO_MAX[4]};
  moveAllTo(t);
}
void posePinch(){ // thumb + index
  int t[5] = {SERVO_MAX[0], SERVO_MAX[1], (SERVO_MIN[2]+SERVO_MAX[2])/2, (SERVO_MIN[3]+SERVO_MAX[3])/2, (SERVO_MIN[4]+SERVO_MAX[4])/2};
  moveAllTo(t);
}

// === SERIAL PARSER ===
// Commands:
//  OPEN
//  CLOSE
//  POINT
//  PINCH
//  F<n> <deg>      -> e.g., "F2 120" sets Middle = 120° (fingers: 0..4)
//  ALL a b c d e   -> set all fingers at once (deg)
String rx;

void handleCommand(const String& cmd){
  String c = cmd; c.trim(); c.toUpperCase();
  if(c == "OPEN"){ poseOpen(); Serial.println("OK OPEN"); return; }
  if(c == "CLOSE"){ poseClosed(); Serial.println("OK CLOSE"); return; }
  if(c == "POINT"){ posePoint(); Serial.println("OK POINT"); return; }
  if(c == "PINCH"){ posePinch(); Serial.println("OK PINCH"); return; }

  if(c.startsWith("F")){
    int sp = c.indexOf(' ');
    if(sp > 1){
      int idx = c.substring(1, sp).toInt();
      int deg = c.substring(sp+1).toInt();
      if(idx >=0 && idx < 5){
        moveFingerTo(idx, deg);
        Serial.println("OK F" + String(idx) + " " + String(currentPos[idx]));
        return;
      }
    }
  }

  if(c.startsWith("ALL")){
    int vals[5]; int count = 0;
    int start = 3; // after "ALL"
    while(count < 5){
      // skip spaces
      while(start < (int)c.length() && c[start] == ' ') start++;
      int end = start;
      while(end < (int)c.length() && isDigit(c[end])) end++;
      if(end == start) break;
      vals[count++] = c.substring(start, end).toInt();
      start = end;
    }
    if(count == 5){
      moveAllTo(vals);
      Serial.println("OK ALL");
      return;
    }
  }

  Serial.println("ERR CMD");
}

void setup(){
  Serial.begin(115200);
  for(int i=0;i<5;i++){
    servos[i].attach(SERVO_PINS[i]);   // power: use external 5–6V, shared GND with Arduino
    currentPos[i] = (SERVO_MIN[i]+SERVO_MAX[i])/2;
    servos[i].write(currentPos[i]);
    delay(200);
  }
  poseOpen();
  Serial.println("Robotic Hand Ready. Commands: OPEN, CLOSE, POINT, PINCH, F<n> <deg>, ALL a b c d e");
}

void loop(){
  while(Serial.available()){
    char ch = (char)Serial.read();
    if(ch == '\n' || ch == '\r'){
      if(rx.length()) handleCommand(rx);
      rx = "";
    }else{
      rx += ch;
      if(rx.length() > 60) rx = ""; // guard
    }
  }
}

Wiring

Pi → PCA9685: SDA to SDA, SCL to SCL, 3V3 to VCC, GND to GND

Servos → PCA9685 channels; servo power from dedicated 5–6 V supply (common GND)






