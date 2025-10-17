```cpp
#include <Servo.h>

Servo leftArm;
Servo rightArm;
Servo leftClaw;
Servo rightClaw;

const int leftLED = 7;
const int rightLED = 11;

const int leftArmPin = 10;
const int rightArmPin = 6;
const int leftClawPin = 5;
const int rightClawPin = 9;

const int ARM_REST = 90;
const int ARM_LOW = 70;
const int ARM_MID = 45;
const int ARM_HIGH = 30;
const int ARM_MAX = 0;

const int LEFT_CLAW_CLOSED = 0;
const int LEFT_CLAW_SLIGHT = 15;
const int LEFT_CLAW_HALF = 25;
const int LEFT_CLAW_MID = 30;
const int LEFT_CLAW_OPEN = 45;

const int RIGHT_CLAW_CLOSED = 135;
const int RIGHT_CLAW_SLIGHT = 150;
const int RIGHT_CLAW_HALF = 160;
const int RIGHT_CLAW_MID = 165;
const int RIGHT_CLAW_OPEN = 180;

int sweepDelay = 15;

unsigned long lastBreathUpdate = 0;
int breathBrightness = 0;
int breathDirection = 1;
bool breathingEnabled = true;

void setup() {
  Serial.begin(9600);
  
  leftArm.attach(leftArmPin);
  rightArm.attach(rightArmPin);
  leftClaw.attach(leftClawPin);
  rightClaw.attach(rightClawPin);
  
  pinMode(leftLED, OUTPUT);
  pinMode(rightLED, OUTPUT);
  
  resetToRest();
  
  Serial.println("Scylla ready!");
}

void loop() {
  if (breathingEnabled) {
    updateBreathing();
  }
  
  if (Serial.available() > 0) {
    String command = Serial.readStringUntil('\n');
    command.trim();
    processCommand(command);
  }
}

void updateBreathing() {
  unsigned long currentMillis = millis();
  if (currentMillis - lastBreathUpdate >= 20) {
    lastBreathUpdate = currentMillis;
    
    breathBrightness += breathDirection * 2;
    
    if (breathBrightness >= 100) {
      breathBrightness = 100;
      breathDirection = -1;
    } else if (breathBrightness <= 0) {
      breathBrightness = 0;
      breathDirection = 1;
    }
    
    analogWrite(leftLED, breathBrightness);
    analogWrite(rightLED, breathBrightness);
  }
}

void disableBreathing() {
  breathingEnabled = false;
  analogWrite(leftLED, 0);
  analogWrite(rightLED, 0);
}

void enableBreathing() {
  breathingEnabled = true;
}

void processCommand(String cmd) {
  if (cmd.startsWith("SERVO:")) {
    handleServoCommand(cmd.substring(6));
  } else if (cmd.startsWith("BLINK:")) {
    handleBlinkCommand(cmd.substring(6));
  }
}

void handleServoCommand(String params) {
  String move = extractParam(params, "move");
  String speed = extractParam(params, "speed");
  int repeat = extractParam(params, "repeat").toInt();
  
  if (repeat < 1) repeat = 1;
  
  if (speed == "slow") sweepDelay = 30;
  else if (speed == "fast") sweepDelay = 5;
  else sweepDelay = 15;

  disableBreathing();
  
  for (int i = 0; i < repeat; i++) {
    executeMove(move);
    if (i < repeat - 1) delay(300);
  }

  resetToRest();

  enableBreathing();
}

void executeMove(String move) {
  if (move == "wave") {
    doWave();
  } else if (move == "double_wave") {
    doDoubleWave();
  } else if (move == "clap") {
    doClap();
  } else if (move == "nod") {
    doNod();
  } else if (move == "pinch") {
    doPinch();
  } else if (move == "curious") {
    doCurious();
  } else if (move == "sad") {
    doSad();
  } else if (move == "angry") {
    doAngry();
  } else if (move == "victory") {
    doVictory();
  } else if (move == "apology") {
    doApology();
  } else if (move == "startled") {
    doStartled();
  } else if (move == "sleep") {
    doSleep();
  } else if (move == "idle") {
    doIdle();
  } else if (move == "cheer") {
    doCheer();
  } else if (move == "salute") {
    doSalute();
  } 
  else if (move == "crab_dance") {
    doCrabDance();
  } else if (move == "claw_samba") {
    doClawSamba();
  } else if (move == "tidal_flow") {
    doTidalFlow();
  } else if (move == "battle_stance") {
    doBattleStance();
  } else if (move == "heart_dance") {
    doHeartDance();
  } else if (move == "party_mode") {
    doPartyMode();
  } else if (move == "rest") {
    resetToRest();
  }
}

// MOVEMENTS

void doWave() {
  // Left arm wave
  smoothMove(leftArm, ARM_HIGH);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  delay(300);
  smoothMove(leftArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  delay(200);
  
  // Right arm wave
  smoothMove(rightArm, ARM_HIGH);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(300);
  smoothMove(rightArm, ARM_REST);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
  delay(200);
}

void doDoubleWave() {
  for (int i = 0; i < 3; i++) {
    // Both arms up
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
    delay(200);
    
    // Both arms down
    smoothMove(leftArm, ARM_REST);
    smoothMove(rightArm, ARM_REST);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
    delay(200);
  }
}

void doClap() {
  // Arms to mid position
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(200);
  
  // Clap 3 times
  for (int i = 0; i < 3; i++) {
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
    delay(150);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
    delay(150);
  }
}

void doNod() {
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(400);
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  delay(300);
}

void doPinch() {
  // Arms to mid position
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(200);
  
  // Snap claws 3 times
  for (int i = 0; i < 3; i++) {
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(100);
    leftClaw.write(LEFT_CLAW_OPEN);
    rightClaw.write(RIGHT_CLAW_OPEN);
    delay(100);
  }
}

void doCurious() {
  // Left arm raises slowly
  smoothMove(leftArm, ARM_HIGH);
  delay(300);
  
  // Right claw slight movement
  smoothMoveClaw(rightClaw, RIGHT_CLAW_MID, true);
  delay(200);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, true);
  delay(200);
  
  // Return
  smoothMove(leftArm, ARM_REST);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
}

void doSad() {
  // Both arms rise slightly and slowly
  int oldDelay = sweepDelay;
  sweepDelay = 30;
  
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(500);
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  
  sweepDelay = oldDelay;
}

void doAngry() {
  for (int i = 0; i < 4; i++) {
    // Quick jerky movements
    leftArm.write(ARM_MID + 15);
    rightArm.write(ARM_MID + 15);
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(100);
    
    leftArm.write(ARM_MID - 15);
    rightArm.write(ARM_MID - 15);
    delay(100);
  }
}

void doVictory() {
  // Arms up high
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(1000);
}

void doApology() {
  // Gentle left arm wave
  smoothMove(leftArm, ARM_HIGH - 10);
  smoothMoveClaw(leftClaw, LEFT_CLAW_MID, true);
  delay(400);
  smoothMove(leftArm, ARM_MID);
  delay(400);
  smoothMove(leftArm, ARM_HIGH - 10);
  delay(400);
  smoothMove(leftArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
}

void doStartled() {
  // Quick jerk up
  leftArm.write(ARM_HIGH);
  rightArm.write(ARM_HIGH);
  leftClaw.write(LEFT_CLAW_CLOSED);
  rightClaw.write(RIGHT_CLAW_CLOSED);
  delay(500);
  
  // Drop down
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
}

void doSleep() {
  int oldDelay = sweepDelay;
  sweepDelay = 35;
  
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
  
  sweepDelay = oldDelay;
}

void doIdle() {
  // Breathing motion
  for (int i = 0; i < 2; i++) {
    smoothMove(leftArm, ARM_MID);
    smoothMove(rightArm, ARM_MID);
    smoothMoveClaw(leftClaw, LEFT_CLAW_HALF, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_HALF, true);
    delay(800);
    
    smoothMove(leftArm, ARM_LOW);
    smoothMove(rightArm, ARM_LOW);
    smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, true);
    delay(800);
  }
}

void doCheer() {
  // Quick up
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(800);
  
  // Slow down
  int oldDelay = sweepDelay;
  sweepDelay = 25;
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
  sweepDelay = oldDelay;
}

void doSalute() {
  // Right arm up halfway
  smoothMove(rightArm, ARM_MID + 15);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(600);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
  delay(200);
  smoothMove(rightArm, ARM_REST);
}

// ========== COMPLEX DANCE MOVEMENTS ==========

void doCrabDance() {
  // Duration: ~8 seconds
  // Both arms start halfway
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(300);
  
  // Alternate left/right shimmy (4 times)
  for (int i = 0; i < 4; i++) {
    // Left up, right down
    smoothMoveParallel(leftArm, ARM_HIGH, rightArm, ARM_LOW);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
    pwmBlinkLED(leftLED, 100, 255);
    delay(500);
    
    // Right up, left down
    smoothMoveParallel(leftArm, ARM_LOW, rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
    pwmBlinkLED(rightLED, 100, 255);
    delay(500);
  }
  
  // Final pose: both arms up, claws open
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  analogWrite(leftLED, 255);
  analogWrite(rightLED, 255);
  delay(800);
  analogWrite(leftLED, 0);
  analogWrite(rightLED, 0);
}

void doClawSamba() {
  // Duration: ~6 seconds
  // Arms half-raised, hold
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(300);
  
  // Rapid claw snapping (4 times)
  for (int i = 0; i < 4; i++) {
    leftClaw.write(LEFT_CLAW_OPEN);
    rightClaw.write(RIGHT_CLAW_OPEN);
    delay(150);
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(150);
  }
  
  // Bop up and down twice
  for (int i = 0; i < 2; i++) {
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    delay(300);
    smoothMove(leftArm, ARM_LOW);
    smoothMove(rightArm, ARM_LOW);
    delay(300);
  }
  
  // Final sync: slow rise and snap
  int oldDelay = sweepDelay;
  sweepDelay = 25;
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(400);
  
  // Sharp close (snap!)
  sweepDelay = 5;
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
  sweepDelay = oldDelay;
  
  delay(300);
}

void doTidalFlow() {
  // Duration: ~10 seconds
  // Slow, hypnotic wave motion
  int oldDelay = sweepDelay;
  sweepDelay = 25;
  
  for (int cycle = 0; cycle < 2; cycle++) {
    // Rise slowly
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_HALF, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_HALF, true);
    delay(500);
    
    // Lower slowly
    smoothMove(leftArm, ARM_REST);
    smoothMove(rightArm, ARM_REST);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
    delay(500);
    
    // Alternate claw direction
    if (cycle == 0) {
      smoothMoveClaw(leftClaw, LEFT_CLAW_HALF, true);
      delay(300);
      smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    } else {
      smoothMoveClaw(rightClaw, RIGHT_CLAW_HALF, true);
      delay(300);
      smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
    }
  }
  
  // Final gentle sway
  for (int i = 0; i < 2; i++) {
    smoothMove(leftArm, ARM_MID + 5);
    smoothMove(rightArm, ARM_MID + 5);
    delay(600);
    smoothMove(leftArm, ARM_MID - 5);
    smoothMove(rightArm, ARM_MID - 5);
    delay(600);
  }
  
  sweepDelay = oldDelay;
}

void doBattleStance() {
  // Duration: ~7 seconds
  // Aggressive, boss intro
  
  for (int i = 0; i < 3; i++) {
    // Fast rise, claws snap
    int oldDelay = sweepDelay;
    sweepDelay = 5;
    smoothMove(leftArm, ARM_HIGH - 10);
    smoothMove(rightArm, ARM_HIGH - 10);
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(300);
    
    // Claws open wide
    leftClaw.write(LEFT_CLAW_OPEN);
    rightClaw.write(RIGHT_CLAW_OPEN);
    delay(300);
    
    // Lower halfway, snap again
    sweepDelay = (i == 1) ? 20 : 8; // Vary tempo
    smoothMove(leftArm, ARM_MID);
    smoothMove(rightArm, ARM_MID);
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(200);
    
    sweepDelay = oldDelay;
  }
  
  // Victory finish
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(1000);
}

void doHeartDance() {
  // Duration: ~8 seconds
  // Charming, affectionate
  int oldDelay = sweepDelay;
  sweepDelay = 20;
  
  // Start position
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, true);
  delay(300);
  
  // Heart pumping motion (4 times)
  for (int i = 0; i < 4; i++) {
    // Arms up, claws open (top of heart)
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
    delay(500);
    
    // Arms down, claws close (bottom of heart)
    smoothMove(leftArm, ARM_MID + 15);
    smoothMove(rightArm, ARM_MID + 15);
    smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, true);
    delay(500);
  }
  
  // Final pose: arms fully raised, claws open (heart complete)
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  
  // Blink LEDs in heart rhythm
  for (int i = 0; i < 3; i++) {
    analogWrite(leftLED, 255);
    analogWrite(rightLED, 255);
    delay(200);
    analogWrite(leftLED, 0);
    analogWrite(rightLED, 0);
    delay(200);
  }
  
  sweepDelay = oldDelay;
  delay(500);
}

void doPartyMode() {
  // Duration: ~20 seconds
  // Combines multiple dance moves
  
  // 1. Crab Dance (4 beats - simplified)
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  for (int i = 0; i < 2; i++) {
    smoothMoveParallel(leftArm, ARM_HIGH, rightArm, ARM_LOW);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
    pwmBlinkLED(leftLED, 100, 255);
    delay(400);
    smoothMoveParallel(leftArm, ARM_LOW, rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
    pwmBlinkLED(rightLED, 100, 255);
    delay(400);
  }
  
  // 2. Claw Samba (2 beats - simplified)
  for (int i = 0; i < 2; i++) {
    leftClaw.write(LEFT_CLAW_OPEN);
    rightClaw.write(RIGHT_CLAW_OPEN);
    delay(150);
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(150);
  }
  
  // 3. Victory Pose
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  analogWrite(leftLED, 255);
  analogWrite(rightLED, 255);
  delay(1000);
  analogWrite(leftLED, 0);
  analogWrite(rightLED, 0);
  
  // 4. Heart Dance (simplified)
  int oldDelay = sweepDelay;
  sweepDelay = 20;
  for (int i = 0; i < 2; i++) {
    smoothMove(leftArm, ARM_MID + 15);
    smoothMove(rightArm, ARM_MID + 15);
    smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, true);
    delay(400);
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
    delay(400);
  }
  sweepDelay = oldDelay;
}

// ========== HELPER FUNCTIONS ==========

void resetToRest() {
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
}

void smoothMove(Servo &servo, int targetAngle) {
  int currentAngle = servo.read();
  int step = (targetAngle > currentAngle) ? 1 : -1;
  
  for (int angle = currentAngle; angle != targetAngle; angle += step) {
    servo.write(angle);
    delay(sweepDelay);
  }
  servo.write(targetAngle);
}

void smoothMoveClaw(Servo &claw, int targetAngle, bool isLeftClaw) {
  // Ensure claws stay within their defined ranges
  if (isLeftClaw) {
    targetAngle = constrain(targetAngle, LEFT_CLAW_CLOSED, LEFT_CLAW_OPEN);
  } else {
    targetAngle = constrain(targetAngle, RIGHT_CLAW_CLOSED, RIGHT_CLAW_OPEN);
  }
  
  int currentAngle = claw.read();
  int step = (targetAngle > currentAngle) ? 1 : -1;
  
  for (int angle = currentAngle; angle != targetAngle; angle += step) {
    claw.write(angle);
    delay(sweepDelay);
  }
  claw.write(targetAngle);
}

void smoothMoveParallel(Servo &servo1, int target1, Servo &servo2, int target2) {
  int current1 = servo1.read();
  int current2 = servo2.read();
  int diff1 = abs(target1 - current1);
  int diff2 = abs(target2 - current2);
  int maxSteps = max(diff1, diff2);
  
  for (int step = 0; step <= maxSteps; step++) {
    int angle1 = map(step, 0, maxSteps, current1, target1);
    int angle2 = map(step, 0, maxSteps, current2, target2);
    servo1.write(angle1);
    servo2.write(angle2);
    delay(sweepDelay);
  }
}

void pwmBlinkLED(int pin, int duration, int brightness) {
  analogWrite(pin, brightness);
  delay(duration);
  analogWrite(pin, 0);
}

// Blink command handler
void handleBlinkCommand(String params) {
  String pattern = extractParam(params, "pattern");
  int delayTime = extractParam(params, "delay").toInt();
  int repeat = extractParam(params, "repeat").toInt();
  
  if (delayTime < 50) delayTime = 300;
  if (repeat < 1) repeat = 1;
  
  // Disable breathing during blink commands
  disableBreathing();
  
  for (int i = 0; i < repeat; i++) {
    if (pattern == "blink_both") {
      analogWrite(leftLED, 255);
      analogWrite(rightLED, 255);
      delay(delayTime);
      analogWrite(leftLED, 0);
      analogWrite(rightLED, 0);
      delay(delayTime);
    } else if (pattern == "wink") {
      analogWrite(leftLED, 255);
      delay(delayTime);
      analogWrite(leftLED, 0);
      delay(delayTime);
    } else if (pattern == "alternate") {
      analogWrite(leftLED, 255);
      delay(delayTime);
      analogWrite(leftLED, 0);
      analogWrite(rightLED, 255);
      delay(delayTime);
      analogWrite(rightLED, 0);
    }
  }
  
  // Re-enable breathing after blink
  enableBreathing();
}

// Extract parameter value from command string
String extractParam(String params, String key) {
  int startIndex = params.indexOf(key + "=");
  if (startIndex == -1) return "";
  
  startIndex += key.length() + 1;
  int endIndex = params.indexOf(",", startIndex);
  
  if (endIndex == -1) {
    return params.substring(startIndex);
  }
  return params.substring(startIndex, endIndex);
}
```