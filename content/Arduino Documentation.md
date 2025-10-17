
## Overview
1. Python sends command like: `SERVO:move=wave,speed=normal,repeat=1`
2. Arduino parses the command
3. Arduino moves servos and controls LEDs
4. Returns to rest position
5. Waits for next command

## Common Variables

```cpp
// Speed settings (controls sweep movement)
int sweepDelay = 15;
```

**Speed Control:**
- **sweepDelay**: Milliseconds between each degree of movement
- *Lower* = faster (5ms = fast)
- *Higher* = slower (30ms = slow)
- *Default* = 15ms (normal speed)

```cpp
// Breathing animation variables
unsigned long lastBreathUpdate = 0;
int breathBrightness = 0;
int breathDirection = 1;
bool breathingEnabled = true;
```

**Breathing Animation Variables:**

|Variable|Purpose|Values|
|---|---|---|
|`lastBreathUpdate`|Timestamp of last brightness change|Milliseconds since startup|
|`breathBrightness`|Current LED brightness|0-100|
|`breathDirection`|Fading in or out?|1 = getting brighter, -1 = getting dimmer|
|`breathingEnabled`|Is breathing active?|true/false|

---
## Servo Angle Ranges

### Servo Angles

```cpp
// Servo angle ranges for ARMS (both move 0-90 degrees)
const int ARM_REST = 0;      // Fully down
const int ARM_LOW = 30;      // Slightly raised
const int ARM_MID = 45;      // Halfway up
const int ARM_HIGH = 70;     // Mostly up
const int ARM_MAX = 90;      // Fully raised
```

**Arm Positions:**

```
ARM_MAX (90°)    ──┐ 
                   │  Arm fully extended up
ARM_HIGH (70°)   ──┤
                   │
ARM_MID (45°)    ──┤  Arm at various heights
                   │
ARM_LOW (30°)    ──┤
                   │
ARM_REST (0°)    ──┘  Arm down at side
```

### Claw Angles

```cpp
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
```

## Setup Function

```cpp
  pinMode(leftLED, OUTPUT);
  pinMode(rightLED, OUTPUT);
```

**LED Setup:**

- Sets LED pins as outputs (Arduino controls them)
- Enables PWM (brightness control) automatically

```cpp
  resetToRest();
  
  Serial.println("Scylla ready!");
}
```

- Move all servos to starting position (0°, closed)
- Print message to Serial Monitor

---
## Main Loop

```cpp
void loop() {
  // Handle breathing animation when idle
  if (breathingEnabled) {
    updateBreathing();
  }
```

**Breathing Animation**

- Runs continuously when no commands active
- Makes LEDs pulse softly (like breathing)
- Only runs if `breathingEnabled` is true

```cpp
  if (Serial.available() > 0) {
    String command = Serial.readStringUntil('\n');
    command.trim();
    processCommand(command);
  }
}
```

**Check for Commands**

1. **Serial.available()**: Returns number of bytes waiting
2. If data waiting: read until newline (`\n`)
3. **trim()**: Remove extra spaces/whitespace
4. Send to `processCommand()` for parsing

**loop repeats forever**

---
## Breathing Animation

```cpp
void updateBreathing() {
  unsigned long currentMillis = millis();
  if (currentMillis - lastBreathUpdate >= 20) {
    lastBreathUpdate = currentMillis;
```

**Timing Control:**
- **millis()**: Time since Arduino started (milliseconds)
- Check if 20ms passed since last update
- If yes: update brightness and save new timestamp
- Creates smooth 50 updates/second (1000ms ÷ 20ms)

```cpp
    breathBrightness += breathDirection * 2;
    
    if (breathBrightness >= 100) {
      breathBrightness = 100;
      breathDirection = -1;
    } else if (breathBrightness <= 0) {
      breathBrightness = 0;
      breathDirection = 1;
    }
```

**Brightness Control:**

1. Increase/decrease brightness by 2 each update
2. If reached 100: start fading down (direction = -1)
3. If reached 0: start fading up (direction = 1)
4. Creates smooth pulsing effect

**Math:**

- 100 brightness ÷ 2 per update = 50 updates to fade
- 50 updates × 20ms = 1000ms (1 second) per fade
- Full breath cycle = 2 seconds (in + out)

```cpp
    analogWrite(leftLED, breathBrightness);
    analogWrite(rightLED, breathBrightness);
  }
}
```

**Apply to LEDs:**

- **analogWrite()**: Sets PWM brightness (0 - 255)
- Both LEDs get same brightness
- Creates synchronized breathing

### Enable/Disable Functions

```cpp
void disableBreathing() {
  breathingEnabled = false;
  analogWrite(leftLED, 0);
  analogWrite(rightLED, 0);
}

void enableBreathing() {
  breathingEnabled = true;
}
```

**Purpose:**

- Turn off breathing during movements (prevents conflicts)
- Turn on breathing after movements (back to idle)
- Ensures LEDs don't fight between breathing and commands

---
## Command Processing

### Main Command Handler

```cpp
void processCommand(String cmd) {
  if (cmd.startsWith("SERVO:")) {
    handleServoCommand(cmd.substring(6));
  } else if (cmd.startsWith("BLINK:")) {
    handleBlinkCommand(cmd.substring(6));
  }
}
```

**Command Routing:**

- Check what type of command
- **startsWith()**: Does string begin with this?
- **substring(6)**: Remove first 6 characters ("SERVO:" or "BLINK:")
- Send to appropriate handler

**Example:**

```
Input:  "SERVO:move=wave,speed=fast"
After substring(6): "move=wave,speed=fast"
```

### Servo Command Handler

```cpp
void handleServoCommand(String params) {
  String move = extractParam(params, "move");
  String speed = extractParam(params, "speed");
  int repeat = extractParam(params, "repeat").toInt();
  
  if (repeat < 1) repeat = 1;
```

**Extract Parameters**

- Parse command string to get values
- **extractParam()**: Custom function (explained later)
- Convert repeat to integer
- Default to 1 if invalid

```cpp
  if (speed == "slow") sweepDelay = 30;
  else if (speed == "fast") sweepDelay = 5;
  else sweepDelay = 15;
```

**Set Speed**

- **slow**: 30ms between each degree (smooth, slow)
- **fast**: 5ms between each degree (quick, snappy)
- **normal** (default): 15ms (balanced)

```cpp
  disableBreathing();
  
  for (int i = 0; i < repeat; i++) {
    executeMove(move);
    if (i < repeat - 1) delay(300);
  }
```

**Execute Movement**

1. Turn off breathing (prevent LED conflicts)
2. Loop for number of repeats
3. Execute the movement
4. Wait 300ms between repeats (except last one)

```cpp
  resetToRest();
  enableBreathing();
}
```

**Cleanup**

1. Return all servos to starting position
2. Turn breathing back on

### Movement Executions

```cpp
void executeMove(String move) {
  if (move == "wave") {
    doWave();
  } else if (move == "double_wave") {
    doDoubleWave();
  } 
  // ... more movements ...
  else if (move == "rest") {
    resetToRest();
  }
}
```

**Movement Routing:**

- Checks movement name
- Calls corresponding function

---
## Basic Movements

### Wave Movement

```cpp
void doWave() {
  smoothMove(leftArm, ARM_HIGH);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  delay(300);
  smoothMove(leftArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  delay(200);
```

**Left Arm Sequence:**
1. Raise left arm to high position
2. Open left claw
3. Pause for 300ms
4. Lower arm to rest (0°)
5. Close claw (0°)
6. Wait 200ms

```cpp
  smoothMove(rightArm, ARM_HIGH);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, true);
  delay(300);
  smoothMove(rightArm, ARM_REST);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, true);
  delay(200);
}
```

**Right Arm Sequence:**
- Same as left, but using right arm/claw

### Double Wave

```cpp
void doDoubleWave() {
  for (int i = 0; i < 3; i++) {
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
    delay(200);
```

**Up Phase:**
- Both arms raise simultaneously
- Both claws open
- Hold 200ms

```cpp
    smoothMove(leftArm, ARM_REST);
    smoothMove(rightArm, ARM_REST);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
    delay(200);
  }
}
```

**Down Phase:**
- Both arms lower
- Both claws close
- Repeat 3 times

### Clap Movement

```cpp
void doClap() {
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(200);
```

- Raise both arms to middle height (45°)
- Hold position briefly

```cpp
  // Clap 3 times
  for (int i = 0; i < 3; i++) {
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
    delay(150);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
    delay(150);
  }
}
```

**Clapping:**

1. Close both claws (together = clap sound)
2. Wait 150ms
3. Open both claws
4. Wait 150ms
5. Repeat 3 times

### Nod Movement

```cpp
void doNod() {
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(400);
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  delay(300);
}
```

- Both arms raise slightly (like nodding head)
- Hold 400ms (acknowledgment)
- Lower back down
- Claws don't move

### Pinch Attack

```cpp
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
```

- Arms at mid height
- **Fast** claw snapping (no smooth move = instant)
- 100ms timing = rapid snaps

### Curious Movement

```cpp
void doCurious() {
  smoothMove(leftArm, ARM_HIGH);
  delay(300);
  
  smoothMoveClaw(rightClaw, RIGHT_CLAW_MID, false);
  delay(200);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, false);
  delay(200);

  smoothMove(leftArm, ARM_REST);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
}
```

- Raises one arm (like raising hand with question)
- Other claw opens slightly then closes (examining)
- Returns to rest

### Sad Movement

```cpp
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
```

1. Save current speed
2. Set to slow (30ms = sluggish)
3. Arms raise slightly
4. Long pause (sadness)
5. Lower slowly
6. Restore original speed

### Angry Shake

```cpp
void doAngry() {
  for (int i = 0; i < 4; i++) {
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
```

- Fast, jerky movements (no smooth)
- Oscillates 15° above/below mid
- Claws snap closed
- 4 rapid shakes

### Victory Pose

```cpp
void doVictory() {
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
  delay(1000);
}
```

- Both arms fully extended (90°)
- Claws wide open
- Hold 1 full second

### Startled Reaction

```cpp
void doStartled() {
  leftArm.write(ARM_HIGH);
  rightArm.write(ARM_HIGH);
  leftClaw.write(LEFT_CLAW_CLOSED);
  rightClaw.write(RIGHT_CLAW_CLOSED);
  delay(500);
  
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
}
```

**Surprise Jump:**

1. **Instant** jump up (no smooth = shock)
2. Claws snap closed
3. Freeze 500ms
4. Slowly lower (recover from fright)

---
## Complex Dance Movements

### Crab Dance

```cpp
void doCrabDance() {
  // Duration: ~8 seconds
  // Both arms start halfway
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  delay(300);
```

- Position arms at middle height
- Prepare for side-to-side motion

```cpp
  for (int i = 0; i < 4; i++) {
    smoothMoveParallel(leftArm, ARM_HIGH, rightArm, ARM_LOW);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
    blinkLED(leftLED, 100, 255);
    delay(500);
    
    smoothMoveParallel(leftArm, ARM_LOW, rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
    blinkLED(rightLED, 100, 255);
    delay(500);
  }
```

1. Left arm up, right arm down (opposite)
2. Left claw open, right closed
3. Flash left LED
4. Switch! Right arm up, left down
5. Flash right LED
6. Repeat 4 times


```cpp
  // Final pose: both arms up, claws open
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
  analogWrite(leftLED, 255);
  analogWrite(rightLED, 255);
  delay(800);
  analogWrite(leftLED, 0);
  analogWrite(rightLED, 0);
}
```

- Both arms fully raised
- Claws wide open
- LEDs bright

### Claw Samba

```cpp
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
```

**Percussion Phase:**

- Arms steady at mid height
- 4 rapid clicks (300ms each)

```cpp
  // Bop up and down twice
  for (int i = 0; i < 2; i++) {
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    delay(300);
    smoothMove(leftArm, ARM_LOW);
    smoothMove(rightArm, ARM_LOW);
    delay(300);
  }
```

- Arms bounce up and down
- 2 bops = 1.2 seconds

```cpp
  // Final sync: slow rise and snap
  int oldDelay = sweepDelay;
  sweepDelay = 25;
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
  delay(400);
  
  // Sharp close (snap!)
  sweepDelay = 5;
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
  sweepDelay = oldDelay;
}
```

**Dramatic Ending:**

1. Slow rise (building tension)
2. Claws open wide
3. (cymbal crash)
4. Restore original speed

### Tidal Flow

```cpp
void doTidalFlow() {
  // Duration: ~10 seconds
  // Slow, hypnotic wave motion
  int oldDelay = sweepDelay;
  sweepDelay = 25;
```

- Save current speed
- Set to slow (25ms = smooth)

```cpp
  for (int cycle = 0; cycle < 2; cycle++) {
    // Rise slowly
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_HALF, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_HALF, false);
    delay(500);
    
    // Lower slowly
    smoothMove(leftArm, ARM_REST);
    smoothMove(rightArm, ARM_REST);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
    delay(500);
```

- Slow rise (like tide coming in)
- Claws half open
- Slow fall (tide going out)
- Claws close
- 2 full cycles

```cpp
    // Alternate claw direction
    if (cycle == 0) {
      smoothMoveClaw(leftClaw, LEFT_CLAW_HALF, true);
      delay(300);
      smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    } else {
      smoothMoveClaw(rightClaw, RIGHT_CLAW_HALF, false);
      delay(300);
      smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
    }
  }
```

- First cycle: left claw accent
- Second cycle: right claw accent
- Adds asymmetry (natural ocean motion)

```cpp
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
```

- Small sway motion (5° range)
- Slow rhythm (600ms each)
- Like gentle waves subsiding
- Restore speed

### Battle Stance

```cpp
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
```

- Fast movements (5ms = aggressive)
- Arms up quickly
- Claws snap shut (instant)
- Hold aggressive pose

```cpp
    // Claws open wide
    leftClaw.write(LEFT_CLAW_OPEN);
    rightClaw.write(RIGHT_CLAW_OPEN);
    delay(300);
    
    // Lower 
        
**Tempo Variation:**
- Claws open wide (threatening)
- Second combo has different speed (i == 1 → slow, else → fast)
- Varies tempo for unpredictability
- 3 attack combos total

```cpp
  // Victory finish
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
  delay(1000);
}
```

**Victorious Finish:**
- Full extension after combat
- Claws open (showing dominance)
- Hold 1 second
- "I win!" pose

**Creates:** Aggressive boss entrance/combat sequence! ⚔️

### Heart Dance

```cpp
void doHeartDance() {
  // Duration: ~8 seconds
  // Charming, affectionate
  int oldDelay = sweepDelay;
  sweepDelay = 20;
  
  // Start position
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, false);
  delay(300);
```

**Gentle Setup:**
- Slow speed (20ms = smooth)
- Arms at heart level
- Claws slightly open

```cpp
  // Heart pumping motion (4 times)
  for (int i = 0; i < 4; i++) {
    // Arms up, claws open (top of heart)
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
    delay(500);
    
    // Arms down, claws close (bottom of heart)
    smoothMove(leftArm, ARM_MID + 15);
    smoothMove(rightArm, ARM_MID + 15);
    smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, false);
    delay(500);
  }
```

**Heart Rhythm:**
- Expands (arms up, claws open)
- Contracts (arms down, claws close)
- Like heartbeat: ba-dum, ba-dum
- 4 beats = 4 seconds

**Visual:**
```
     ╱ ╲         ← Arms up, claws open (systole)
    ╱   ╲
    
     \ /         ← Arms down, claws close (diastole)
      V
```

```cpp
  // Final pose: arms fully raised, claws open (heart complete)
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
  
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
```

**Finale:**
- Full heart shape (arms max, claws open)
- LEDs pulse 3 times (heartbeat effect)
- Restore original speed
- Extra pause before reset

**Creates:** Affectionate, loving gesture! ❤️

### Party Mode

```cpp
void doPartyMode() {
  // Duration: ~20 seconds
  // Combines multiple dance moves
  
  // 1. Crab Dance (4 beats - simplified)
  smoothMove(leftArm, ARM_MID);
  smoothMove(rightArm, ARM_MID);
  for (int i = 0; i < 2; i++) {
    smoothMoveParallel(leftArm, ARM_HIGH, rightArm, ARM_LOW);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
    pwmBlinkLED(leftLED, 100, 255);
    delay(400);
    smoothMoveParallel(leftArm, ARM_LOW, rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
    pwmBlinkLED(rightLED, 100, 255);
    delay(400);
  }
```

**Section 1: Crab Dance (abbreviated)**
- Side shimmy (2 cycles instead of 4)
- LED flashing with rhythm
- ~3 seconds

```cpp
  // 2. Claw Samba (2 beats - simplified)
  for (int i = 0; i < 2; i++) {
    leftClaw.write(LEFT_CLAW_OPEN);
    rightClaw.write(RIGHT_CLAW_OPEN);
    delay(150);
    leftClaw.write(LEFT_CLAW_CLOSED);
    rightClaw.write(RIGHT_CLAW_CLOSED);
    delay(150);
  }
```

**Section 2: Claw Samba (abbreviated)**
- Quick clap beats
- 2 snaps instead of 4
- ~0.6 seconds

```cpp
  // 3. Victory Pose
  smoothMove(leftArm, ARM_MAX);
  smoothMove(rightArm, ARM_MAX);
  smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
  analogWrite(leftLED, 255);
  analogWrite(rightLED, 255);
  delay(1000);
  analogWrite(leftLED, 0);
  analogWrite(rightLED, 0);
```

**Section 3: Victory Pose**
- Full celebration
- Bright LEDs
- Hold 1 second

```cpp
  // 4. Heart Dance (simplified)
  int oldDelay = sweepDelay;
  sweepDelay = 20;
  for (int i = 0; i < 2; i++) {
    smoothMove(leftArm, ARM_MID + 15);
    smoothMove(rightArm, ARM_MID + 15);
    smoothMoveClaw(leftClaw, LEFT_CLAW_SLIGHT, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_SLIGHT, false);
    delay(400);
    smoothMove(leftArm, ARM_HIGH);
    smoothMove(rightArm, ARM_HIGH);
    smoothMoveClaw(leftClaw, LEFT_CLAW_OPEN, true);
    smoothMoveClaw(rightClaw, RIGHT_CLAW_OPEN, false);
    delay(400);
  }
  sweepDelay = oldDelay;
}
```

**Section 4: Heart Dance (abbreviated)**
- 2 heartbeats instead of 4
- Slow, affectionate finish
- Restore speed

**Creates:** Complete celebration medley! 🎉

---

## Helper Functions

### Reset to Rest

```cpp
void resetToRest() {
  smoothMove(leftArm, ARM_REST);
  smoothMove(rightArm, ARM_REST);
  smoothMoveClaw(leftClaw, LEFT_CLAW_CLOSED, true);
  smoothMoveClaw(rightClaw, RIGHT_CLAW_CLOSED, false);
}
```

**Purpose:**
- Returns all servos to starting position
- Arms down (0°)
- Claws closed
- Called after every movement
- Ensures clean state for next command

### Smooth Move (Single Servo)

```cpp
void smoothMove(Servo &servo, int targetAngle) {
  int currentAngle = servo.read();
  int step = (targetAngle > currentAngle) ? 1 : -1;
```

**Step 1: Determine Direction**
- **servo.read()**: Get current angle
- If target > current: move up (+1 per step)
- If target < current: move down (-1 per step)
- Ternary operator: `condition ? if_true : if_false`

```cpp
  for (int angle = currentAngle; angle != targetAngle; angle += step) {
    servo.write(angle);
    delay(sweepDelay);
  }
  servo.write(targetAngle);
}
```

**Step 2: Move Gradually**
1. Start at current angle
2. Loop until reaching target
3. Move 1 degree per iteration
4. Wait `sweepDelay` milliseconds
5. Final write ensures exact target reached

**Example:**
```
Current: 30°, Target: 60°, Speed: 15ms
→ Moves: 30→31→32→...→59→60
→ Total time: 30 steps × 15ms = 450ms
```

**Why smooth movement?**
- Prevents servo jerking
- Looks more natural
- Reduces mechanical stress

### Smooth Move Claw

```cpp
void smoothMoveClaw(Servo &claw, int targetAngle, bool isLeftClaw) {
  // Ensure claws stay within their defined ranges
  if (isLeftClaw) {
    targetAngle = constrain(targetAngle, LEFT_CLAW_CLOSED, LEFT_CLAW_OPEN);
  } else {
    targetAngle = constrain(targetAngle, RIGHT_CLAW_OPEN, RIGHT_CLAW_CLOSED);
  }
```

**Range Safety:**
- **constrain(value, min, max)**: Limits value to range
- Left claw: force between 0-45°
- Right claw: force between 135-180°
- Prevents servo damage from invalid angles

```cpp
  int currentAngle = claw.read();
  int step = (targetAngle > currentAngle) ? 1 : -1;
  
  for (int angle = currentAngle; angle != targetAngle; angle += step) {
    claw.write(angle);
    delay(sweepDelay);
  }
  claw.write(targetAngle);
}
```

**Movement:**
- Same as `smoothMove()` but with safety checks
- Protects claws from going outside their mechanical limits

### Smooth Move Parallel

```cpp
void smoothMoveParallel(Servo &servo1, int target1, Servo &servo2, int target2) {
  int current1 = servo1.read();
  int current2 = servo2.read();
  int diff1 = abs(target1 - current1);
  int diff2 = abs(target2 - current2);
  int maxSteps = max(diff1, diff2);
```

**Step 1: Calculate Movement**
- Get both current positions
- Calculate distance each needs to travel
- **abs()**: Absolute value (always positive)
- **max()**: Use larger distance as step count
- Ensures both finish at same time

```cpp
  for (int step = 0; step <= maxSteps; step++) {
    int angle1 = map(step, 0, maxSteps, current1, target1);
    int angle2 = map(step, 0, maxSteps, current2, target2);
    servo1.write(angle1);
    servo2.write(angle2);
    delay(sweepDelay);
  }
}
```

**Step 2: Synchronized Movement**
- **map()**: Scales step number to angle
- Both servos move proportionally
- Reach targets simultaneously

**Example:**
```
Servo1: 30° → 90° (60° travel)
Servo2: 70° → 40° (30° travel)
maxSteps = 60

Step 0:  Servo1=30°, Servo2=70°
Step 30: Servo1=60°, Servo2=55°
Step 60: Servo1=90°, Servo2=40° ← Both finish together!
```

**Why parallel?**
- Creates coordinated movements
- More natural-looking dances
- Both arms move in sync

### PWM Blink LED

```cpp
void pwmBlinkLED(int pin, int duration, int brightness) {
  analogWrite(pin, brightness);
  delay(duration);
  analogWrite(pin, 0);
}
```

**Simple LED Flash:**
- **pin**: Which LED to blink
- **duration**: How long to stay on (milliseconds)
- **brightness**: 0-255 (0=off, 255=brightest)
- Turn on → wait → turn off

**Usage Examples:**
```cpp
pwmBlinkLED(leftLED, 100, 255);    // Bright flash, 100ms
pwmBlinkLED(rightLED, 500, 128);   // Dim flash, 500ms
pwmBlinkLED(leftLED, 200, 200);    // Medium flash, 200ms
```

---

## LED Control

### Blink Command Handler

```cpp
void handleBlinkCommand(String params) {
  String pattern = extractParam(params, "pattern");
  int delayTime = extractParam(params, "delay").toInt();
  int repeat = extractParam(params, "repeat").toInt();
  
  if (delayTime < 50) delayTime = 300;
  if (repeat < 1) repeat = 1;
```

**Extract Parameters:**
- Get pattern type, delay, and repeat count
- Validate: minimum 50ms delay (prevent too fast)
- Minimum 1 repeat

```cpp
  // Disable breathing during blink commands
  disableBreathing();
```

**Pause Breathing:**
- Prevents conflict between breathing animation and blink pattern
- Ensures clean LED control

```cpp
  for (int i = 0; i < repeat; i++) {
    if (pattern == "blink_both") {
      analogWrite(leftLED, 255);
      analogWrite(rightLED, 255);
      delay(delayTime);
      analogWrite(leftLED, 0);
      analogWrite(rightLED, 0);
      delay(delayTime);
```

**Blink Both Pattern:**
1. Turn both LEDs to full brightness
2. Wait delay time
3. Turn both LEDs off
4. Wait delay time
5. Repeat as specified

```cpp
    } else if (pattern == "wink") {
      analogWrite(leftLED, 255);
      delay(delayTime);
      analogWrite(leftLED, 0);
      delay(delayTime);
```

**Wink Pattern:**
- Only left LED
- On → off
- Like winking one eye 😉

```cpp
    } else if (pattern == "alternate") {
      analogWrite(leftLED, 255);
      delay(delayTime);
      analogWrite(leftLED, 0);
      analogWrite(rightLED, 255);
      delay(delayTime);
      analogWrite(rightLED, 0);
    }
  }
```

**Alternate Pattern:**
1. Left LED on
2. Wait
3. Left off, right on
4. Wait
5. Right off
6. Creates back-and-forth effect

```cpp
  // Re-enable breathing after blink
  enableBreathing();
}
```

**Resume Breathing:**
- Turn breathing animation back on
- Return to idle state

### Pattern Examples

**Command:** `BLINK:pattern=blink_both,delay=200,repeat=3`
```
Time:    0ms   200ms  400ms  600ms  800ms  1000ms
Left:    ██    ░░     ██     ░░     ██     ░░
Right:   ██    ░░     ██     ░░     ██     ░░
         ^ON   ^OFF   ^ON    ^OFF   ^ON    ^OFF
```

**Command:** `BLINK:pattern=alternate,delay=300,repeat=2`
```
Time:    0ms   300ms  600ms  900ms  1200ms
Left:    ██    ░░     ██     ░░
Right:   ░░    ██     ░░     ██
```

---

## String Parsing

### Extract Parameter

```cpp
String extractParam(String params, String key) {
  int startIndex = params.indexOf(key + "=");
  if (startIndex == -1) return "";
```

**Step 1: Find Key**
- **indexOf()**: Searches for substring
- Looks for "key=" (e.g., "move=")
- Returns -1 if not found
- Return empty string if key doesn't exist

```cpp
  startIndex += key.length() + 1;
  int endIndex = params.indexOf(",", startIndex);
```

**Step 2: Find Value**
- Skip past "key=" to start of value
- **key.length() + 1**: Length of key + "=" character
- Find next comma (end of value)

```cpp
  if (endIndex == -1) {
    return params.substring(startIndex);
  }
  return params.substring(startIndex, endIndex);
}
```

**Step 3: Extract Value**
- If no comma: value goes to end of string
- Otherwise: value is between start and comma
- **substring()**: Extracts portion of string

**Example:**
```
Input: "move=wave,speed=fast,repeat=2"
Key: "speed"

1. Find "speed=" at position 10
2. startIndex = 10 + 6 = 16 (start of "fast")
3. Find "," at position 20
4. Extract substring(16, 20) = "fast"
```

---

## Complete Command Flow

### Example: Wave Movement

**Python sends:**
```
SERVO:move=wave,speed=normal,repeat=1\n
```

**Arduino receives:**
```
1. loop() detects serial data
2. Reads: "SERVO:move=wave,speed=normal,repeat=1"
3. Calls processCommand()
4. Recognizes "SERVO:" prefix
5. Calls handleServoCommand("move=wave,speed=normal,repeat=1")
6. Extracts: move="wave", speed="normal", repeat=1
7. Sets sweepDelay = 15 (normal speed)
8. Disables breathing
9. Calls executeMove("wave")
10. Executes doWave():
    - Left arm up, claw open, wait, down, claw close
    - Right arm up, claw open, wait, down, claw close
11. Calls resetToRest()
12. Enables breathing
13. Done! Ready for next command
```

---

## Timing Breakdown

### Movement Durations

| Movement | Approximate Time | Notes |
|----------|-----------------|-------|
| Wave | 1.0s | Each arm waves sequentially |
| Double Wave | 2.4s | 3 cycles × 0.8s |
| Clap | 1.1s | Setup + 3 claps |
| Nod | 0.7s | Simple up-down |
| Pinch | 0.8s | 3 rapid snaps |
| Curious | 1.2s | Slow, thoughtful |
| Sad | 1.5s | Slow movement |
| Angry | 0.8s | 4 quick shakes |
| Victory | 1.5s | Hold pose |
| Startled | 1.0s | Jump + recover |
| **Crab Dance** | **8.0s** | Full dance sequence |
| **Claw Samba** | **6.0s** | Rhythmic performance |
| **Tidal Flow** | **10.0s** | Meditative waves |
| **Battle Stance** | **7.0s** | Combat sequence |
| **Heart Dance** | **8.0s** | Affectionate gesture |
| **Party Mode** | **20.0s** | Complete medley |

---

## Customization Tips

### Adding New Movements

**Steps:**
1. Create new function (follow naming pattern):
```cpp
void doMyMove() {
  // Your movement code here
  smoothMove(leftArm, ARM_HIGH);
  // etc...
}
```

2. Add to `executeMove()`:
```cpp
else if (move == "my_move") {
  doMyMove();
}
```

3. Update Python system prompt to include new move

### Adjusting Angles

**Safe ranges:**
- Arms: 0-90° (can extend to 0-180° if mechanical design allows)
- Claws: Depends on gripper design, test carefully

**How to find limits:**
```cpp
void setup() {
  // ... existing setup ...
  
  // Test servo range
  leftArm.write(0);
  delay(2000);
  leftArm.write(180);
  delay(2000);
  // Watch servo - note where it binds or stalls
}
```

### Creating Custom LED Patterns

```cpp
void myLEDPattern() {
  // Fade in
  for (int i = 0; i <= 255; i += 5) {
    analogWrite(leftLED, i);
    analogWrite(rightLED, i);
    delay(20);
  }
  
  // Fade out
  for (int i = 255; i >= 0; i -= 5) {
    analogWrite(leftLED, i);
    analogWrite(rightLED, i);
    delay(20);
  }
}
```

### Changing Breathing Speed

```cpp
void updateBreathing() {
  unsigned long currentMillis = millis();
  if (currentMillis - lastBreathUpdate >= 20) {  // ← Change this!
    // 10 = faster breathing
    // 30 = slower breathing
    // 20 = default
```

---

## Advanced Features

### Servo Position Feedback

Check current servo position:
```cpp
int currentPos = leftArm.read();
Serial.print("Left arm at: ");
Serial.println(currentPos);
```

### Non-blocking Delays

Current code uses `delay()` which blocks. For advanced projects, use `millis()`:

```cpp
unsigned long previousMillis = 0;
const long interval = 1000;

void loop() {
  unsigned long currentMillis = millis();
  
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;
    // Do something every second without blocking
  }
  
  // Other code runs continuously
}
```

### Smooth Speed Curves

For more natural movement, implement easing:
```cpp
void smoothMoveEased(Servo &servo, int targetAngle) {
  int current = servo.read();
  int distance = targetAngle - current;
  
  for (int step = 0; step <= 100; step++) {
    // Ease-in-out formula
    float t = step / 100.0;
    float eased = t < 0.5 ? 2*t*t : -1+(4-2*t)*t;
    int angle = current + (distance * eased);
    servo.write(angle);
    delay(10);
  }
}
```


---
## Code Structure Summary

```
Main Components:
├── Setup (runs once)
│   ├── Serial communication
│   ├── Servo attachment
│   ├── LED pinMode
│   └── Reset to rest
│
├── Loop (runs forever)
│   ├── Breathing animation
│   └── Command processing
│
├── Command Handlers
│   ├── processCommand() - Router
│   ├── handleServoCommand() - Servo movements
│   └── handleBlinkCommand() - LED patterns
│
├── Movements
│   ├── Basic (15 movements)
│   └── Complex Dances (6 sequences)
│
└── Helpers
    ├── smoothMove() - Single servo
    ├── smoothMoveClaw() - Claw with limits
    ├── smoothMoveParallel() - Two servos
    ├── pwmBlinkLED() - LED flash
    ├── resetToRest() - Home position
    └── extractParam() - String parsing
```

---
