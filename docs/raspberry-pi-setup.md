# Raspberry Pi 5 Setup and Demo Guide

Everything you need, explained from zero. You will never need a monitor plugged into the Pi.

---

## How You Actually Control the Pi (Read This First)

The Raspberry Pi is a small computer that runs without a screen. You control it entirely from your Windows laptop over WiFi using something called **SSH**.

**SSH in plain terms:** you open PowerShell on your laptop, type one command, and from that point on everything you type in that PowerShell window runs on the Pi. The Pi's output appears on your laptop screen. That's it. Your laptop becomes the Pi's keyboard and screen wirelessly.

**The full picture on presentation day:**
1. Pi is powered on (just plug it in), sits on the table next to you
2. You open PowerShell on your laptop
3. You type `ssh pi@raspberrypi.local` → you are now inside the Pi
4. You type the demo command → the script runs on the Pi
5. Your laptop screen shows the results (image name, pill count, status)
6. The audience watches the LEDs light up and the servo rotate

---

## What You Need (Hardware)

| Item | Qty | Notes |
|---|---|---|
| Raspberry Pi 5 | 1 | Any RAM size |
| MicroSD card | 1 | At least 16 GB |
| USB-C power supply (5V 5A) | 1 | Official Pi 5 PSU recommended |
| SG90 servo motor | 1 | Standard micro-servo |
| Red LED (5 mm) | 1 | |
| Blue LED (5 mm) | 1 | |
| Green LED (5 mm) | 1 | |
| Resistors (330Ω) | 3 | One per LED |
| Breadboard | 1 | |
| Jumper wires (female-to-male) | ~15 | The female end plugs into the Pi's pins |

No monitor. No keyboard for the Pi. No camera.

---

## Step 1 — Flash the SD Card (done once, on your laptop)

This is where you install the operating system onto the SD card AND configure WiFi + SSH before the Pi ever boots.

1. Download **Raspberry Pi Imager** on your Windows laptop:
   https://www.raspberrypi.com/software/

2. Insert the SD card into your laptop (use a USB adapter if needed)

3. Open Raspberry Pi Imager:
   - **Device:** Raspberry Pi 5
   - **Operating System:** Raspberry Pi OS (64-bit) — the default one
   - **Storage:** your SD card

4. **Before clicking Write**, click the **gear icon** (or "Edit Settings") — this is the critical step:
   - Set **hostname** to `raspberrypi`
   - Set a **username** (use `pi`) and **password** (remember this, you'll type it every SSH)
   - Check **Configure wireless LAN** → enter your WiFi name and password
   - Check **Enable SSH** → select "Use password authentication"

5. Click **Save**, then **Write**. Wait for it to finish (~5 minutes).

6. Remove the SD card from your laptop and insert it into the Pi.

---

## Step 2 — First Boot

Plug the USB-C power cable into the Pi. A red LED on the Pi will light up, then a green one will blink. Wait about 60 seconds for it to fully boot and connect to WiFi.

You do not need to do anything else. It connects automatically.

---

## Step 3 — Connect from Your Laptop via SSH

Open **PowerShell** on your Windows laptop (search for it in the Start menu).

Type:
```
ssh pi@raspberrypi.local
```

First time only, it will ask:
```
Are you sure you want to continue connecting? (yes/no)
```
Type `yes` and press Enter.

Then it asks for your password (the one you set in Raspberry Pi Imager). Type it and press Enter. **Note: you won't see the characters as you type — that's normal.**

You should now see something like:
```
pi@raspberrypi:~ $
```

You are now inside the Pi. Every command you type runs on the Pi.

> If `raspberrypi.local` doesn't work, you need to find the Pi's IP address. Log into your home router's admin page (usually 192.168.1.1 in your browser) and look for a device called "raspberrypi" in the connected devices list. Then use that IP: `ssh pi@192.168.1.XX`

---

## Step 4 — Install Dependencies (done once)

Once you are SSH'd in, type these commands one by one. Wait for each to finish before typing the next.

```bash
sudo apt update
sudo apt install -y python3-pip python3-opencv
pip3 install lgpio numpy
```

If `python3-opencv` fails, use this instead:
```bash
pip3 install opencv-python
```

---

## Step 5 — Copy the Project to the Pi

Still in the same PowerShell window (while SSH'd in), first **open a second PowerShell window** (not SSH'd — a normal one on your laptop) and run:

```
scp -r "C:\path\to\pill-blister-classifier" pi@raspberrypi.local:~/pill-blister-classifier
```

This copies the entire project folder from your laptop to the Pi over WiFi. It will ask for your Pi password again.

Then go back to your SSH window and check it arrived:
```bash
ls ~/pill-blister-classifier
```

You should see `pill_classifier_RPi5.py` and the other files listed.

---

## Step 6 — Put Your Test Images on the Pi

Create the images folder and copy your blister pack images into it. The easiest way is from the **normal PowerShell window** (not SSH):

```
scp C:\path\to\your\images\*.jpg pi@raspberrypi.local:~/pill-blister-classifier/images/
```

Or if you have them in a folder:
```
scp -r C:\path\to\images pi@raspberrypi.local:~/pill-blister-classifier/images
```

Confirm they arrived (in the SSH window):
```bash
ls ~/pill-blister-classifier/images
```

---

## Step 7 — Wire the Hardware

### LED wiring (do this for all 3 LEDs)

```
Pi GPIO pin  →  330Ω resistor  →  LED long leg (+)  →  LED short leg (-)  →  Pi GND pin
```

| LED | BCM pin | Physical pin on the Pi header |
|---|---|---|
| Red | GPIO 17 | Pin 11 |
| Blue | GPIO 27 | Pin 13 |
| Green | GPIO 22 | Pin 15 |
| GND (shared for all) | GND | Pin 6 |

### Servo wiring

The SG90 servo has 3 wires:

| Servo wire color | Connect to |
|---|---|
| Orange or Yellow (signal) | Physical Pin 32 (GPIO 12) |
| Red (power) | Physical Pin 2 or 4 (5V) |
| Brown or Black (ground) | Any GND pin |

> The servo needs to be powered from the **5V pin**, not from the GPIO signal pin. Only the signal wire goes to GPIO 12.

### Full pin map for reference

```
         3V3  (1) (2)  5V          ← Servo power (red wire) here
       GPIO2  (3) (4)  5V
       GPIO3  (5) (6)  GND         ← Shared GND for LEDs and servo here
       GPIO4  (7) (8)  GPIO14
         GND  (9) (10) GPIO15
      GPIO17 (11) (12) GPIO18      ← Red LED signal here (pin 11)
      GPIO27 (13) (14) GND
      GPIO22 (15) (16) GPIO23      ← Green LED signal here (pin 15)
         3V3 (17) (18) GPIO24
      GPIO10 (19) (20) GND
       GPIO9 (21) (22) GPIO25
      GPIO11 (23) (24) GPIO8
         GND (25) (26) GPIO7
       GPIO0 (27) (28) GPIO1
       GPIO5 (29) (30) GND
       GPIO6 (31) (32) GPIO12      ← Servo signal here (pin 32)
      GPIO13 (33) (34) GND
      GPIO19 (35) (36) GPIO16
      GPIO26 (37) (38) GPIO20
         GND (39) (40) GPIO21
                                   (Blue LED = GPIO 27 = pin 13)
```

---

## Step 8 — Test Hardware Before the Demo

**Test the servo** (in your SSH window):
```bash
python3 -c "
import lgpio, time
h = lgpio.gpiochip_open(0)
lgpio.gpio_claim_output(h, 12, 0)
for duty in [3.0, 7.5, 12.0, 7.5]:
    lgpio.tx_pwm(h, 12, 50, duty); time.sleep(1)
lgpio.tx_pwm(h, 12, 0, 0)
lgpio.gpiochip_close(h)
"
```
The servo should move to 0°, 90°, 180° and back to 90°.

**Test the LEDs** (in your SSH window):
```bash
python3 -c "
import lgpio, time
h = lgpio.gpiochip_open(0)
for pin in [17, 27, 22]:
    lgpio.gpio_claim_output(h, pin, 0)
lgpio.gpio_write(h, 17, 1); time.sleep(1); lgpio.gpio_write(h, 17, 0)
lgpio.gpio_write(h, 27, 1); time.sleep(1); lgpio.gpio_write(h, 27, 0)
lgpio.gpio_write(h, 22, 1); time.sleep(1); lgpio.gpio_write(h, 22, 0)
lgpio.gpiochip_close(h)
"
```
Red, then blue, then green should each light for 1 second.

---

## Running the Demo

In your SSH window:
```bash
cd ~/pill-blister-classifier
python3 pill_classifier_RPi5.py --use-gpio --wait-key
```

### What you see on your laptop screen

```
Headless mode — 3 image(s) in /home/pi/pill-blister-classifier/images
Ctrl-C to quit.

[1/3] good_pack.jpg
  Pills=10  Empty=0  Anomaly=False  PillColor=WHITE
  → OK  LEDs(R=False, B=False, G=True)  servo=7.5%
  [Enter] next image…

[2/3] empty_cells.jpg
  Pills=8  Empty=2  Anomaly=False  PillColor=WHITE
  → EMPTY_ONLY  LEDs(R=True, B=False, G=False)  servo=12.0%
  [Enter] next image…

[3/3] wrong_color.jpg
  Pills=10  Empty=0  Anomaly=True  PillColor=COLORED
  → ANOMALY_ONLY  LEDs(R=False, B=True, G=False)  servo=9.75%
  [Enter] next image…
```

### What the hardware does

| Result | Red LED | Blue LED | Green LED | Servo |
|---|---|---|---|---|
| All good | OFF | OFF | ON | 90° center |
| Empty cells | ON | OFF | OFF | 180° |
| Wrong color | OFF | ON | OFF | 135° |
| Both problems | ON | ON | OFF | 0° |

Press **Enter** to advance to the next image. Press **Ctrl+C** to stop early.

---

## Day-of-Presentation Checklist

- [ ] SD card flashed with WiFi + SSH configured
- [ ] Pi boots and connects to WiFi (wait 60 seconds after plugging power)
- [ ] `ssh pi@raspberrypi.local` works from your laptop PowerShell
- [ ] Dependencies installed (`lgpio`, `opencv`)
- [ ] Project files in `~/pill-blister-classifier/` on the Pi
- [ ] Demo images in `~/pill-blister-classifier/images/` on the Pi
- [ ] LEDs wired with 330Ω resistors, correct pins
- [ ] Servo signal on pin 32 (GPIO 12), power on pin 2 (5V), GND connected
- [ ] Servo test passes (moves to 0°, 90°, 180°)
- [ ] LED test passes (each lights up in sequence)
- [ ] Full demo run works: `python3 pill_classifier_RPi5.py --use-gpio --wait-key`

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `ssh: Could not resolve hostname raspberrypi.local` | Pi not on WiFi or wrong hostname | Check WiFi credentials entered in Imager; find Pi's IP from router and use that instead |
| SSH asks password and then says "Permission denied" | Wrong password | The password is what you set in Raspberry Pi Imager |
| `ModuleNotFoundError: lgpio` | Not installed | `pip3 install lgpio` |
| `ModuleNotFoundError: cv2` | Not installed | `pip3 install opencv-python` |
| `No images found` | Wrong folder | Run `ls ~/pill-blister-classifier/images` to check; re-copy images |
| Servo doesn't move | Wrong pin or power | Signal must be pin 32, power from pin 2 (5V) not from GPIO |
| LEDs don't light | Wrong pin, or LED backwards, or no resistor | Long leg of LED connects toward the GPIO pin side; short leg toward GND |
| `PermissionError` on GPIO | Permission issue | Run with `sudo python3 pill_classifier_RPi5.py --use-gpio --wait-key` |
