# 🤖 Obstacle Avoidance Robot — Arduino

<img width="3072" height="4096" alt="IMG20260714074425" src="https://github.com/user-attachments/assets/7efbb474-10ee-4452-9cc7-3522e8f32da9" />


A fully autonomous robot that navigates on its own using an 
ultrasonic sensor and servo motor. No remote. No control. 
Just place it on the ground and it moves, detects, scans, 
decides, and avoids — all by itself.

Built by **Sarvesh Kumar** — 2nd Year B.Sc.(H) Electronics Student  
Completely self-taught. No mentor. Just curiosity.

---

## 📹 Demo Video
[Click here to watch Demo Video](https://github.com/user-attachments/assets/0c9da0d8-6eef-46cd-a503-34ce30c67bf6)


---

## 💡 How It Works
Robot moves forward
↓
Ultrasonic sensor detects object within 35cm
↓
Buzzer beeps — motors stop
↓
Servo rotates LEFT → reads distance
Servo rotates RIGHT → reads distance
Servo returns to CENTER
↓
More space on LEFT?  → Turn LEFT
More space on RIGHT? → Turn RIGHT
Both sides blocked?  → Reverse → Scan again → Pick best side
↓
Move forward to clear obstacle
↓
Resume forward — repeat forever

---

## ⚙️ Components Used

| Component | Quantity |
|---|---|
| Arduino Uno | 1 |
| L298N Motor Driver | 1 |
| HC-SR04 Ultrasonic Sensor | 1 |
| SG90 Servo Motor | 1 |
| DC Motors (with wheels) | 4 |
| Active Buzzer | 1 |
| LED (Headlight) | 1 |
| HC-05 Bluetooth Module | 1 |
| 18650 Li-ion Battery | 2 |
| Robot Car Chassis | 1 |
| Jumper Wires | As needed |

---

## 📌 Pin Connections

### L298N Motor Driver → Arduino

| L298N Pin | Arduino Pin |
|---|---|
| IN1 | D4 |
| IN2 | D5 |
| IN3 | D6 |
| IN4 | D7 |
| GND | GND (shared with Arduino) |
| 12V | Battery + |

### HC-SR04 Ultrasonic Sensor → Arduino

| HC-SR04 Pin | Arduino Pin |
|---|---|
| VCC | 5V |
| TRIG | D11 |
| ECHO | D12 |
| GND | GND |

### SG90 Servo Motor → Arduino

| Servo Wire | Arduino Pin |
|---|---|
| Signal (Orange) | D10 |
| VCC (Red) | 5V |
| GND (Brown) | GND |

### HC-05 Bluetooth Module → Arduino

| HC-05 Pin | Arduino Pin |
|---|---|
| RX | D2 |
| TX | D3 |
| VCC | 5V |
| GND | GND |

### Other Components

| Component | Arduino Pin |
|---|---|
| Buzzer | D8 |
| LED | D9 |

---

## 🔌 Motor Connections

### Left Side Motors
| Motor | L298N Pin |
|---|---|
| Front Left (+) | OUT1 |
| Front Left (−) | OUT2 |
| Rear Left (+) | OUT1 |
| Rear Left (−) | OUT2 |

### Right Side Motors
| Motor | L298N Pin |
|---|---|
| Front Right (+) | OUT3 |
| Front Right (−) | OUT4 |
| Rear Right (+) | OUT3 |
| Rear Right (−) | OUT4 |

---

## 📱 Bluetooth Commands

Even though this is an autonomous robot, Bluetooth is used
to switch between modes.

| Command | Action |
|---|---|
| `V` | Start Autonomous Mode |
| `v` or `M` | Switch to Manual BT Mode |
| `F` | Forward (manual only) |
| `B` | Backward (manual only) |
| `L` | Left (manual only) |
| `R` | Right (manual only) |
| `S` | Stop (manual only) |
| `U` | Headlight ON |
| `u` | Headlight OFF |
| `W` | Parking Mode ON |
| `w` | Parking Mode OFF |
| `X` | Emergency Mode ON |
| `x` | Emergency Mode OFF |
| `Y` | Truck Horn |

**Recommended App:** Bluetooth RC Controller (Play Store)

---

## 🔧 Servo Alignment

On every power-on the servo does an alignment sweep:
CENTER (90°) → LEFT (160°) → RIGHT (20°) → CENTER (90°)

This confirms the servo is working and sets the zero position
before autonomous mode starts.

---

## ⚡ Capacitor Connections (Noise Fix)

Motors generate electrical noise that can reset the Arduino.
Add these capacitors to prevent it.

| Capacitor | Where to Connect |
|---|---|
| 100nF ceramic | Directly across OUT1 and OUT2 on L298N |
| 100µF electrolytic (+→OUT1, −→OUT2) | Directly across OUT1 and OUT2 on L298N |
| 100nF ceramic | Directly across OUT3 and OUT4 on L298N |
| 100µF electrolytic (+→OUT3, −→OUT4) | Directly across OUT3 and OUT4 on L298N |
| 470µF electrolytic (+→12V pin, −→GND) | L298N power input pins |
| 10µF electrolytic (+→5V, −→RESET) | Arduino board |

---

## 🚨 Known Issues and Fixes

### Issue 1 — Servo vs SoftwareSerial Timer Conflict
**Problem:** Servo library and SoftwareSerial both use Timer1
on Arduino Uno. When servo is always attached, Bluetooth
commands stop working randomly.

**Fix:** Detach servo when in manual BT mode. Attach servo
only when entering autonomous mode.

```cpp
// When entering autonomous mode
scanner.attach(SERVO_PIN);

// When exiting autonomous mode
scanner.write(SERVO_CENTER);
delay(400);
scanner.detach();   // frees Timer1 for SoftwareSerial
```

---

### Issue 2 — Motors Stop Randomly in Autonomous Mode
**Problem:** Motor direction set once on state entry. Electrical
noise from motors briefly interrupts IN1-IN4 signals causing
motors to stop mid-movement.

**Fix:** Call motor function on every loop iteration inside
every wait state — not just once on entry.

```cpp
// Wrong — motor set once, can stop from noise
case A_WAIT_TURN:
  if (now - autoTimer >= TURN_TIME) nextState();
  break;

// Correct — motor called every loop
case A_TURN_L_WAIT:
  leftTurn();    // called every single loop iteration
  if (now - autoTimer >= TURN_TIME) nextState();
  break;
```

---

### Issue 3 — L298N Overheating
**Problem:** L298N wastes 2-4V internally as heat. Running
4 motors causes thermal shutdown after a few minutes.

**Fix:**
- Add heatsink with thermal paste on L298N chip
- Use a 3S battery (11.1V) instead of 2S (7.4V) for
  more voltage headroom
- Upgrade to TB6612FNG driver for permanent fix (only
  0.5V drop vs 3-4V drop in L298N)

---

### Issue 4 — pulseIn() Blocking Loop
**Problem:** pulseIn() with long timeout blocks the loop
for up to 23ms. During this time servo signal weakens and
motors can lose state.

**Fix:** Use short timeout (15000µs) and treat no echo
as clear path (return 250cm).

```cpp
long dur = pulseIn(ECHO, HIGH, 15000);
if (dur == 0) return 250;  // no echo = clear path
```

---

## 📄 Code

See `obstacle_avoidance_robot.ino` in this repository.

### Key Parameters You Can Tune

```cpp
#define OBSTACLE_DIST    35   // detection distance in cm
#define SIDE_CLEAR_DIST  25   // min cm on side to be "open"
#define SERVO_CENTER     90   // adjust if servo not straight
#define SERVO_LEFT      160   // adjust for your servo range
#define SERVO_RIGHT      20   // adjust for your servo range
#define TURN_TIME       800   // ms to turn (increase = sharper)
#define REVERSE_TIME    600   // ms to reverse when blocked
```

---

## 🛠️ How To Upload

1. Install [Arduino IDE](https://www.arduino.cc/en/software)
2. Install **Servo** library (comes built-in with Arduino IDE)
3. Install **SoftwareSerial** library (comes built-in)
4. Connect Arduino via USB
5. Select **Tools → Board → Arduino Uno**
6. Select **Tools → Port → COMX**
7. Open `obstacle_avoidance_robot.ino`
8. Click **Upload**
9. Disconnect USB, power via battery
10. Send `V` from Bluetooth app to start autonomous mode

---

## 🧠 What I Learned Building This

- How ultrasonic sensors work and their blind spots below 2cm
- Why Timer1 conflicts between Servo and SoftwareSerial
  cause random BT failures — and how detach() fixes it
- How motor back-EMF creates noise that resets Arduino
- Why delay() inside a state machine breaks everything
- How to build a proper non-blocking state machine using
  millis() so multiple things happen simultaneously
- That old salvaged capacitors dry out and stop working —
  always use new ones for noise filtering

---

## 📁 Repository Structure


---

## 🔗 Related Project

Also check out my **Bluetooth RC Car** — the car this
autonomous system was built on top of:

👉 [bluetooth-rc-car](https://github.com/the-sarvesh-k/bluetooth-rc-car)

---

## 📝 License
MIT License — free to use, modify, and share

---

## 👤 Author
**Sarvesh Kumar**
2nd Year B.Sc.(H) Electronics Student
GitHub: [github.com/the-sarvesh-k](https://github.com/the-sarvesh-k)

> Built completely alone. No mentor. Just course knowledge,
> curiosity, and a lot of debugging at 2AM.




