# Washer-Sense: Smart IoT Upgrade for Any Washer

**Published:** July 20, 2026 by YonderLabo  
**Difficulty:** Beginner (3 hours 15 mins)

Stop forgetting your laundry! Stick this M5Stack device on your washer to track IMU vibrations and get instant Discord notifications.

## Things used in this project

### Hardware components
* M5Stack M5StickC PLUS ESP32-PICO Mini IoT Development Kit × 1

### Software apps and online services
* M5Stack UIFlow 2.0

---

## Story

### Washer-Sense: A Smart, Non-Intrusive IoT Laundry Monitor

#### The Inspiration: Why I Built This
* *"Is the laundry done yet?"* — Wasting time constantly walking back and forth to the laundry room to check.
* *"I forgot to hang the clothes!"* — Leaving wet clothes in the machine for hours, leading to a damp smell and forcing a re-wash.

These everyday frustrations are what initially inspired this project. To solve them, I wanted a straightforward, non-intrusive solution that notifies me the exact moment the laundry cycle finishes.

#### Designing for Accessibility
Furthermore, there is another crucial motivation behind this project.
I have been involved in developing IoT devices for the hearing impaired. Most household appliances rely on auditory cues (like a loud beep) to signal completion. This design is not accessible, and I have heard many complaints from the deaf and hard-of-hearing community.

I wanted to create a mechanism that not only eliminates inconvenience for the general public but also provides a simple, visual, and reliable smartphone alert system to assist those with hearing loss in their daily lives.

#### The Solution: Washer-Sense
*Washer-Sense* is a retrofitted, low-cost IoT add-on.
By simply attaching an *M5StickC Plus2* to the exterior of your washing machine, it detects physical vibrations and sends a real-time notification to your smartphone via Discord when the cycle completes.
* **Zero-Invasive:** No tools, wiring modifications, or disassembly required.
* **Easy Setup:** Just stick it on and power it up.

#### What Makes This Project Special
While there are many DIY laundry monitors, *Washer-Sense* stands out due to its practical, real-world design focus:
* **Zero-Invasive Retrofitting:** Unlike projects that splice into high-voltage appliance wiring or hack into control boards, Washer-Sense sits entirely on the outside. It is 100% safe, requires no electrical knowledge, and is completely renter-friendly.
* **Smart Auto-Calibration:** Most vibration sensors fail because different washing machines have different spin cycles and baseline vibrations. Instead of hardcoding thresholds in the code, Washer-Sense calibrates itself to *your* specific machine at the press of a button.
* **Lightweight & Modular Architecture:** By pairing a micro-controller (M5Stick) with a cloud integrator (Make.com), the system remains incredibly lightweight. You can easily swap Discord for Slack, Telegram, or SMS without rewriting a single line of device firmware.

#### How It Works
The entire workflow is split into three simple, lightweight steps:

1. **The Sensor Device (M5StickC Plus2):** Continuously monitors vibration data using its built-in IMU (Inertial Measurement Unit). When a state change is detected, it triggers an HTTP GET request to a Make.com webhook.
2. **Cloud Integration (Make.com):** Receives the webhook, parses the machine's current status, and forwards the message to Discord using the Discord API.
3. **Instant Notification (Discord):** Pushes a direct alert (e.g., ✅ Washer-Sense: Washing Finished!) straight to your smartphone or smartwatch.

### State Machine Logic
To prevent false alarms and ensure accuracy, the device dynamically tracks the washing machine's cycle through five distinct states based on the duration of physical vibrations:

* **IDLE:** The machine is completely at rest.
* **CHECKING:** Initial vibration is detected. If the vibration stops before reaching 5 seconds, the system treats it as a transient disturbance (false alarm) and returns directly to IDLE.
* **WASHING:** If vibration continues consistently for *5 seconds* , the system confirms the washing cycle has officially started.
* **PAUSED:** If vibration temporarily stops (e.g., during water fill or drain cycles), it enters a paused state to prevent premature finish alerts.
* **FINISHED!:** If the machine remains completely silent in the paused state for *10 seconds* , it triggers the final Discord notification. After 3 seconds, it automatically resets back to IDLE for the next load.

### Smart Features for Easy Setup
To make daily operation and debugging seamless, I utilized the device's physical buttons:

#### 1. One-Touch Calibration (Button A)
* **How it works:** Pressing Button A samples the machine's orientation and ambient environment for *3 seconds*.
* **Why it's useful:** Automatically calibrates and sets the optimal vibration threshold, ensuring it works perfectly on any washing machine model regardless of background noise.

#### 2. Live Test Mode (Button B)
* **How it works:** Toggles between two notification profiles:
  * **Normal Mode:** Only notifies you on cycle start and completion (ideal for daily use).
  * **Test Mode:** Sends a Discord notification on *every* state change (ideal for debugging and adjusting the sensor's position).

---

## Code

### Washer-Sense: Retrofit Smart Laundry IoT Sensor

```python
import os, sys, io
import M5
from M5 import *
import math
import time
import network
import requests

# ==========================================
# 1. Network & Make.com Settings
# ==========================================
WIFI_SSID = "YOUR_WIFI_SSID"
WIFI_PASS = "YOUR_WIFI_PASSWORD"

# Make.com Webhook URL (Base part)
MAKE_WEBHOOK_URL = "YOUR_MAKE_WEBHOOK_URL"

# ==========================================
# 2. Operational Parameters
# ==========================================
# Notify mode (0: Start & Finish only, 1: Test mode - all state changes)
NOTIFY_MODE = 0
TIME_TO_WASHING = 5.0 # Seconds of continuous vibration to determine "WASHING"
TIME_TO_FINISHED = 10.0 # Seconds of complete silence to determine "FINISHED!"

# ==========================================
# Global Variables
# ==========================================
# Label objects
lbl_state = None
lbl_v = None
lbl_th = None
lbl_wifi = None

# Graph buffer (for 120 points)
graph_data = [ 0 ] * 120

# Vibration and State management
V_THRESHOLD = 0.5
base_x, base_y, base_z = 0.0, 0.0, 1.0 # Gravity baseline on installation
last_time = 0
last_vibe_time = 0
start_vibe_time = 0
current_machine_state = "IDLE"

# Communication
wlan = None
wifi_state_prev = None

def setup():
    global lbl_state, lbl_v, lbl_th, lbl_wifi
    global current_machine_state, wlan, wifi_state_prev
    global base_x, base_y, base_z

    M5.begin()
    Widgets.setRotation(0)
    Widgets.fillScreen(0x000000)

    # UI Layout (Clean top-to-bottom arrangement)
    lbl_state = Widgets.Label("Connecting...", 10, 5, 1.2, 0x00FFFF, 0x000000, Widgets.FONTS.Montserrat18)
    lbl_v = Widgets.Label("0.000", 10, 40, 2.0, 0x00FF00, 0x000000, Widgets.FONTS.Montserrat18)
    lbl_th = Widgets.Label(f"Th: {V_THRESHOLD:.3f}", 10, 90, 1.0, 0x00FFFF, 0x000000, Widgets.FONTS.Montserrat18)
    lbl_wifi = Widgets.Label("WiFi: Wait", 10, 130, 1.0, 0xCCCCCC, 0x000000, Widgets.FONTS.Montserrat14)

    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        wlan.connect(WIFI_SSID, WIFI_PASS)
    while not wlan.isconnected():
        time.sleep(1)

    wifi_state_prev = True
    lbl_wifi.setText("WiFi: OK")
    lbl_wifi.setColor(0x00FF00, 0x000000)

    current_machine_state = "IDLE"
    restore_state_label()

    if NOTIFY_MODE == 1:
        send_to_make("🔵 Washer-Sense: Booted", "BOOT")

    sum_x = sum_y = sum_z = 0.0
    for _ in range(25):
        try:
            accel = M5.Imu.getAccel()
            sum_x += accel[0]
            sum_y += accel[1]
            sum_z += accel[2]
        except: pass
        time.sleep_ms(10)
    base_x = sum_x / 25.0
    base_y = sum_y / 25.0
    base_z = sum_z / 25.0

def draw_graph():
    M5.Lcd.fillRect(0, 170, 160, 70, 0x000000)

    # Scale Y-axis based on the threshold
    graph_max = V_THRESHOLD * 2.0
    if graph_max <= 0.01:
        graph_max = 0.05

    for i in range(1, len(graph_data)):
        x1 = i - 1
        y1 = 240 - int((graph_data[i - 1] / graph_max) * 70)
        x2 = i
        y2 = 240 - int((graph_data[i] / graph_max) * 70)

        # Clip values to fit within the graph area
        y1 = max(170, min(240, y1))
        y2 = max(170, min(240, y2))

        M5.Lcd.drawLine(x1, y1, x2, y2, 0x00FF00)

    # Draw threshold line (Red)
    th_y = 240 - int((V_THRESHOLD / graph_max) * 70)
    th_y = max(170, min(240, th_y))
    M5.Lcd.drawLine(0, th_y, 160, th_y, 0xFF0000)

def restore_state_label():
    global current_machine_state, lbl_state
    lbl_state.setText(current_machine_state)
    if current_machine_state == "IDLE": lbl_state.setColor(0xaaaaaa, 0x000000)
    elif current_machine_state == "CHECKING": lbl_state.setColor(0xFFFF00, 0x000000)
    elif current_machine_state == "WASHING": lbl_state.setColor(0xFF8800, 0x000000)
    elif current_machine_state == "PAUSED": lbl_state.setColor(0x4488FF, 0x000000)
    elif current_machine_state == "FINISHED!": lbl_state.setColor(0x00FF00, 0x000000)

def do_calibration():
    global V_THRESHOLD, lbl_state, lbl_th
    global base_x, base_y, base_z
    global current_machine_state

    lbl_state.setColor(0xFFFF00, 0x000000)

    # 3-second countdown (Wait for vibrations to settle)
    for i in range(3, 0, -1):
        lbl_state.setText(f"Wait {i}s")
        time.sleep(1)

    lbl_state.setColor(0x00FFFF, 0x000000)

    # 3-second measurement for posture and noise (Total 150 samples)
    sum_x = sum_y = sum_z = 0.0
    sum_noise = 0.0
    valid_samples = 0
    samples_per_sec = 50

    for sec in range(1, 4):
        lbl_state.setText(f"Meas {sec}/3s")
        for _ in range(samples_per_sec):
            try:
                accel = M5.Imu.getAccel()
                # For posture measurement
                sum_x += accel[0]
                sum_y += accel[1]
                sum_z += accel[2]

                # Temporary noise calculation (Using previous baseline)
                dx = accel[0] - base_x
                dy = accel[1] - base_y
                dz = accel[2] - base_z
                vibe = math.sqrt(dx * dx + dy * dy + dz * dz) * 10.0
                sum_noise += vibe
                valid_samples += 1
            except:
                pass
            time.sleep_ms(20)

    # After measurement, set new posture baseline and set threshold to 2x noise
    if valid_samples > 0:
        base_x = sum_x / valid_samples
        base_y = sum_y / valid_samples
        base_z = sum_z / valid_samples

        avg_noise = sum_noise / valid_samples
        V_THRESHOLD = avg_noise * 2.0

        # Safety minimum threshold
        if V_THRESHOLD < 0.15:
            V_THRESHOLD = 0.15

    lbl_th.setText(f"Th: {V_THRESHOLD:.3f}")
    lbl_state.setText("Calib Done!")
    lbl_state.setColor(0x00FF00, 0x000000)
    time.sleep(1.5)

    # Force back to IDLE after calibration
    current_machine_state = "IDLE"
    restore_state_label()

def url_encode(text):
    res = ""
    for b in text.encode('utf-8'):
        if 0x41 <= b <= 0x5A or 0x61 <= b <= 0x7A or 0x30 <= b <= 0x39 or b in (0x2D, 0x2E, 0x5F, 0x7E):
            res += chr(b)
        else:
            res += "%%%02X" % b
    return res

def send_to_make(msg_text, status_text):
    global lbl_state
    try:
        safe_msg = url_encode(msg_text)
        safe_status = url_encode(status_text)
        request_url = f"{MAKE_WEBHOOK_URL}?msg={safe_msg}&status={safe_status}"
        response = requests.get(request_url)
        response.close()
        return True
    except Exception as e:
        print(f"Make Error: {e}")
        return False

def loop():
    global graph_data, last_time, V_THRESHOLD
    global last_vibe_time, start_vibe_time
    global current_machine_state, lbl_state
    global wlan, wifi_state_prev, lbl_wifi
    global NOTIFY_MODE

    M5.update()
    current_time = time.ticks_ms()

    # Button A: Trigger calibration
    if M5.BtnA.wasPressed():
        do_calibration()
        last_time = time.ticks_ms()
        return

    # Button B: Toggle notify mode
    if M5.BtnB.wasPressed():
        NOTIFY_MODE = 1 if NOTIFY_MODE == 0 else 0
        lbl_state.setColor(0x00FFFF, 0x000000)

        if NOTIFY_MODE == 1:
            lbl_state.setText("MODE: TEST")
            send_to_make("🛠️ Washer-Sense: TEST MODE ON", "MODE")
        else:
            lbl_state.setText("MODE: NORMAL")
            send_to_make("🛠️ Washer-Sense: NORMAL MODE ON", "MODE")

        time.sleep(1.5)
        restore_state_label()
        last_time = time.ticks_ms()
        return

    # Check sensor and update state machine
    if time.ticks_diff(current_time, last_time) >= 200:
        last_time = current_time

        try:
            # Monitor Wi-Fi status
            current_wifi = wlan.isconnected()
            if current_wifi != wifi_state_prev:
                wifi_state_prev = current_wifi
                if current_wifi:
                    lbl_wifi.setText("WiFi: OK")
                    lbl_wifi.setColor(0x00FF00, 0x000000)
                else:
                    lbl_wifi.setText("WiFi: ERR")
                    lbl_wifi.setColor(0xFF0000, 0x000000)

            # High-sensitivity sensor reading (Peak hold & 10x amplifier)
            peak_vibe = 0.0
            for _ in range(5):
                accel = M5.Imu.getAccel()
                dx = accel[0] - base_x
                dy = accel[1] - base_y
                dz = accel[2] - base_z
                v = math.sqrt(dx * dx + dy * dy + dz * dz)
                if v > peak_vibe: peak_vibe = v
                time.sleep_ms(5)

            vibe = peak_vibe * 10.0
            lbl_v.setText(f"{vibe:.3f}")

            is_vibration = vibe > V_THRESHOLD
            now = time.ticks_ms()
            previous_state = current_machine_state

            # --- State Machine Logic ---
            if is_vibration:
                last_vibe_time = now
                if current_machine_state in ["IDLE", "FINISHED!"]:
                    start_vibe_time = now
                    current_machine_state = "CHECKING"
                    restore_state_label()

            if current_machine_state == "CHECKING":
                if time.ticks_diff(now, last_vibe_time) > 3000:
                    current_machine_state = "IDLE"
                    restore_state_label()
                elif time.ticks_diff(now, start_vibe_time) >= TIME_TO_WASHING * 1000:
                    current_machine_state = "WASHING"
                    restore_state_label()

            elif current_machine_state == "WASHING":
                if not is_vibration and time.ticks_diff(now, last_vibe_time) > 2000:
                    current_machine_state = "PAUSED"
                    restore_state_label()

            elif current_machine_state == "PAUSED":
                if is_vibration:
                    current_machine_state = "WASHING"
                    restore_state_label()
                elif time.ticks_diff(now, last_vibe_time) >= TIME_TO_FINISHED * 1000:
                    current_machine_state = "FINISHED!"
                    restore_state_label()

            elif current_machine_state == "FINISHED!":
                # Reset to IDLE 3 seconds after becoming FINISHED!
                if time.ticks_diff(now, last_vibe_time) >= (TIME_TO_FINISHED + 3.0) * 1000:
                    current_machine_state = "IDLE"
                    restore_state_label()

            if current_machine_state != previous_state:
                if NOTIFY_MODE == 1:
                    if current_machine_state == "FINISHED!":
                        send_to_make("✅ Washer-Sense: Washing Finished!", current_machine_state)
                    else:
                        send_to_make(f"🔄 State Changed: {current_machine_state}", current_machine_state)

                elif NOTIFY_MODE == 0:
                    if current_machine_state == "FINISHED!":
                        send_to_make("✅ Washer-Sense: Washing Finished!", current_machine_state)
                    elif current_machine_state == "WASHING" and previous_state in ["IDLE", "CHECKING"]:
                        send_to_make("▶️ Washer-Sense: Washing Started", current_machine_state)

            graph_data.pop(0)
            graph_data.append(vibe)
            draw_graph()

        except Exception as e:
            lbl_v.setText("ERR")

if __name__ == '__main__':
    try:
        setup()
        while True:
            loop()
    except Exception as e:
        sys.print_exception(e)
