# ATtiny85 SecurID-Style Token Generator

A standalone, battery-powered "token fob" that displays a rotating 6-digit random code on a tiny OLED screen — like an old-school RSA SecurID hardware token — built from a bare ATtiny85 chip on a breadboard.

<table>
<tr>
<td align="center" width="50%">
<img src="https://github.com/user-attachments/assets/fab10e43-40af-4404-8fa8-3cb2b5b4ccb9" width="380"><br>
<sub><b>The AtTiny85 "bug" </b></sub>
</td>
<td align="center" width="50%">
<img src="https://github.com/user-attachments/assets/2666dacc-7c1a-4d5f-9ec1-4b3889b13e3e width="380"><br>
<sub><b>Six Digit Code, with 60 second counter!</b></sub>
</td>
</tr>
</table>

---

## What it does

- Boots standalone (no computer attached) off a 5V source
- Displays a random 6-digit code on a 128x64 I2C OLED
- Rotates to a new random code every n seconds
- Runs entirely from an 8-pin DIP chip with 8KB flash / 512B RAM — no OS, no bootloader in the traditional sense

NB: This is intended for fun as is not meant to be secure in any manner.

---

## Requirements

### Hardware
| Part | Notes |
|---|---|
| ATtiny85 (DIP-8, `ATTINY85-20PU`) | The "bug" — main chip. Buy a few; you will lose/bend legs on at least one. |
| SSD1306 128x64 I2C OLED display | 4-pin (GND/VCC/SCL/SDA) breakout style |
| ISP or UPDI-style programmer | See [Programmer Notes](#programmer-notes) below — **this mattered a lot** |
| Breadboard (mini is fine) | Needs a center gap to straddle the DIP-8 chip |
| DIP-8 socket (optional but recommended) | Round/machined-pin style, not cheap stamped contacts |
| 5V power source | USB power bank, bench supply, or programmer board's own 5V pins |
| Jumper wires | Get a **known-good** set; cheap kits sometimes have dead wires |
| Multimeter | For continuity/voltage checks — see troubleshooting section |
| Resistors + LEDs (optional) | For a blink indicator |

### Software
- **Arduino IDE** (1.8.x tested)
- **ATTinyCore** board package by Spence Konde
  Boards Manager URL: `http://drazzy.com/package_drazzy.com_index.json`
- **U8g2 / U8x8 library** (search "U8g2" in Library Manager — U8x8lib.h ships inside it)
- **avrdude** (bundled with Arduino IDE, or install standalone via `apt`/`brew`)

### Board settings (Arduino IDE → Tools)
- Board: `ATtiny25/45/85 (No bootloader)`
- Chip: `ATtiny85`
- Clock: `8 MHz (internal)`
- Programmer: matches whatever ISP tool you're using (see below)

---

## Programmer Notes

This was, by far, the biggest source of pain in this project. Two programmer types were tried:

| Programmer | Result | avrdude flag | Photo |
|---|---|---|---|
| **USBASP** (10-pin, with 6-pin ISP adapter) | ❌ Never got a successful handshake, across 3 different chips, multiple breadboard positions, multiple wire swaps | `-c usbasp` | <img src="https://github.com/user-attachments/assets/08a44cea-86d9-4380-a8f6-091ae7482b16" width="140"> |
| **USB Tiny AVR Programmer ("FabISP")** with onboard 8-pin ZIF-style socket | ✅ Worked on the very first try | `-c usbtiny` |<img src="https://github.com/user-attachments/assets/a539ebb2-86d2-4420-870b-daad665597cb" width="140">  |

**Key lesson:** the difference wasn't the chips, the fuses, or the code — it was **hand-wired breadboard ISP connections being unreliable**. A programmer with its own **built-in chip socket** (no hand-wiring required for the 6 ISP signals) eliminated the entire failure category in one shot.

If you're starting this project fresh: **buy a programmer with an onboard DIP-8 ZIF/socket** (like a FabISP-style board) rather than a bare USBASP + adapter cable + breadboard jumpers. It costs about the same and will save you hours.

Workflow used:
1. Seat chip in the FabISP's onboard socket
2. Flash code via Arduino IDE (`Sketch → Upload Using Programmer`)
3. Pull chip out, move to breadboard for actual runtime use
4. Repeat steps 1–2 any time code needs updating

---

## Pinout

### ATtiny85 (DIP-8) physical pin map
```
        notch
      ┌───∪───┐
PB5 1 │       │ 8 VCC
PB3 2 │ATtiny │ 7 PB2 (SCK / OLED SCL)
PB4 3 │  85   │ 6 PB1 (MISO)
GND 4 │       │ 5 PB0 (MOSI / OLED SDA)
      └───────┘
```

### Standard 10-pin AVR ISP header (USBASP-style)
```
 1 MISO   2 VCC
 3 SCK    4 MOSI
 5 RESET  6 GND
 7  --    8 GND
 9  --   10 GND
```

### Runtime wiring (chip already flashed, connected to OLED)
| From | To |
|---|---|
| 5V source (+) | ATtiny85 pin 8 (VCC) |
| 5V source (−) | ATtiny85 pin 4 (GND) |
| ATtiny85 pin 8 | OLED VCC |
| ATtiny85 pin 4 | OLED GND |
| ATtiny85 pin 5 (PB0) | OLED SDA |
| ATtiny85 pin 7 (PB2) | OLED SCL |
| ATtiny85 pin 2 (PB3) | **leave floating/unconnected** — used as entropy source |
| ATtiny85 pin 3 (PB4) | optional: LED (+resistor) for a blink indicator |

Note: SDA/SCL reuse the same physical pins as MOSI/SCK from ISP programming. No conflict — ISP mode is only active while a programmer is actively driving RESET low.

---

## Troubleshooting Journey (chronological)

This is the real value of this README — the actual pitfalls hit, in order, so you can skip them.

### 1. "target does not answer (0x01)" — USBASP, first attempts
- Checked wiring against standard pinout — looked correct
- **Root cause candidate #1: 3.3V/5V jumper on the USBASP was set to 3.3V.** Moved to 5V — voltage confirmed correct at the source afterward.

### 2. Still failing after fixing the voltage jumper
- Measured voltage *at the chip itself* (not just the programmer) — read a partial voltage (2.24V), well below the ~5V measured at the source.
- **Lesson:** a partial voltage reading (not 0V, not full V) usually means a poor/loose connection somewhere in between, not a clean break.

### 3. Breadboard seating issue
- Discovered the chip's 8 legs were **all landed in the same row block** instead of straddling the breadboard's center gap — this shorts pins together internally via the breadboard's row strips.
- **Lesson:** always confirm the DIP chip straddles the center trench, with pins 1–4 and 5–8 in separate row blocks.

### 4. USB device dropping/reconnecting
- `dmesg` showed repeated USB disconnect/reconnect cycles.
- **False alarm** — this was caused by manually toggling a USB switch during testing, not a hardware fault. Worth ruling out early with `sudo dmesg -T -w` running live.

### 5. Continuity testing attempted, inconclusive
- Attempted wire-by-wire continuity tests with a multimeter to isolate a bad connection (particularly RESET).
- Results were unreliable — **multimeter continuity testing on tiny breadboard holes is genuinely fiddly** for a first-timer, and a "confirmed bad wire" finding earlier in the session turned out to be an unreliable read, not solid evidence. Don't over-trust a single continuity reading if you're new to the tool.

### 6. Root cause, functionally: breadboard/hand-wired ISP connection was unreliable
- After extensive troubleshooting across 3 different ATtiny85 chips, multiple breadboard positions, and multiple wire swaps, **the exact cause was never pinned to one single wire** — but switching to a programmer with an onboard chip socket (no hand-wiring for ISP) immediately and permanently resolved it.
- **Practical lesson:** if ISP hand-wiring on a breadboard is fighting you, stop debugging wire-by-wire and eliminate the whole category — use a programmer with a built-in socket.

### 7. Signature check failure after switching to FabISP
```
avrdude: device signature = 0xffffff (probably .xmega) (retrying)
```
- Cause: chip was inserted **backward** into the FabISP's onboard socket.
- **Lesson:** `0xFFFFFF` (all 1s) is the classic "nothing is actually responding" signature — always suspect orientation/seating first, not a wrong or dead chip.

### 8. Compile error: flash overflow
```
region `text' overflowed by 684 bytes
```
- Cause: used a large, full-alphabet bold font (`u8x8_font_courB18_2x3_r`) for a 6-digit-only display.
- **Fix:** switch to the numeric-only variant of the same font (`u8x8_font_courB18_2x3_n`) — the `_n` suffix means "digits only," dramatically smaller than `_r` (full ASCII), since it isn't storing unused letter glyphs.
- **Lesson:** RAM was never the constraint for text on this chip (U8x8 is bufferless, doesn't hold a screen image in RAM) — **flash** (program storage) is the actual budget font choice eats into.

### 9. First code took ~30 seconds to appear on boot
- Initially suspected a stalled entropy-seeding loop (8x `analogRead()` in a row) — simplified to a single `analogRead()` + `micros()` seed. Did not fix it.
- **Real root cause: fuses.** `Tools → Burn Bootloader` was never run after setting the clock to "8 MHz internal." A factory-fresh ATtiny85 defaults to running at 1MHz (8MHz internal oscillator ÷ 8 clock-divider fuse). Since `millis()` counts based on actual fuse-configured clock speed, the code's 30-second timer was actually running ~8x slower than intended.
- **Fix:** `Tools → Burn Bootloader` once (with ATTinyCore this only sets fuses, no actual bootloader is written), then re-upload the sketch.
- **Lesson:** Always run "Burn Bootloader" once on any freshly-acquired or previously-unconfigured chip, or any time the Tools → Clock setting changes — skipping it leaves fuses mismatched with what your code assumes.

---

## Hardware Tried — What Worked, What Didn't

| Item | Result | Notes |
|---|---|---|
| USBASP + 10-pin→6-pin ISP adapter | ❌ Failed | Root cause never fully isolated to one wire; category-eliminated by switching programmers |
| USB Tiny AVR Programmer ("FabISP") w/ onboard socket | ✅ Worked immediately | `-c usbtiny` in avrdude, not `-c usbasp` |
| 3x cheap AliExpress ISP programmer clones (~$3) | Not fully tested | Purchased as backups; FabISP solved the issue first |
| Bare ATtiny85 legs directly in breadboard | ⚠️ Unreliable | Suspected contributor to ISP failures; not conclusively proven as the sole cause |
| Machined round-pin DIP-8 socket | ✅ Used for OLED/runtime wiring | Better contact reliability than bare chip-in-breadboard, though not proven necessary in this case |
| Adafruit ATtiny1616 + seesaw breakout | Considered, not used | Requires UPDI programming (different protocol/tooling), not ISP — would need a different programmer entirely |
| 100µF electrolytic capacitor for decoupling | ❌ Wrong part | Too slow to respond to fast switching noise; 0.1µF ceramic is correct for this purpose. Never place any cap on RESET — interferes with programmer's reset pulse. |

---

## Final Code

```cpp
#include <U8x8lib.h>

U8X8_SSD1306_128X64_NONAME_SW_I2C u8x8(/* clock=*/ 2, /* data=*/ 0, /* reset=*/ U8X8_PIN_NONE);

const int ENTROPY_PIN = A3; // physical pin 2 (PB3) — leave unconnected/floating
unsigned long lastRefresh;
const unsigned long REFRESH_MS = 30000; // 30 seconds, like a real SecurID fob

void generateAndShowCode() {
  char code[7];
  for (int i = 0; i < 6; i++) {
    code[i] = '0' + random(0, 10);
  }
  code[6] = '\0';

  u8x8.clear();
  u8x8.setFont(u8x8_font_courB18_2x3_n); // numeric-only variant — fits in 8KB flash
  u8x8.drawString(2, 2, code);            // x=2 centers a 12-tile string on a 16-tile-wide screen
}

void setup() {
  u8x8.begin();
  u8x8.setPowerSave(0);

  // Single quick analog read for entropy — no loop, no stall risk
  randomSeed(analogRead(ENTROPY_PIN) + micros());

  generateAndShowCode();      // shows a code immediately
  lastRefresh = millis();     // starts the 30s clock AFTER first display
}

void loop() {
  if (millis() - lastRefresh >= REFRESH_MS) {
    generateAndShowCode();
    lastRefresh = millis();
  }
}
```

### Optional: non-blocking LED blink add-on
```cpp
const int LED_PIN = 4; // physical pin 3, PB4
unsigned long lastBlink = 0;
const unsigned long BLINK_MS = 500;
bool ledState = false;

// add to setup():
//   pinMode(LED_PIN, OUTPUT);

// add inside loop():
//   if (millis() - lastBlink >= BLINK_MS) {
//     ledState = !ledState;
//     digitalWrite(LED_PIN, ledState);
//     lastBlink = millis();
//   }
```

---

## Key Takeaways / Learnings Summary

1. **Programmers with an onboard chip socket beat hand-wired ISP breadboard setups**, hands down, for reliability. If you're fighting a "target does not answer" error for more than a few attempts, stop debugging wires and switch tooling instead.
2. **`0xFFFFFF` signature = nothing responding** — almost always orientation or seating, not a dead/wrong chip.
3. **A partial voltage reading (neither 0V nor full V) = a bad connection somewhere**, not a power source problem.
4. **DIP chips must straddle the breadboard's center gap** — all-pins-in-one-row-block silently shorts things together.
5. **Font size costs flash, not RAM**, when using U8x8 (bufferless text mode) — use numeric-only (`_n`) font variants when you only need digits.
6. **Always Burn Bootloader once** on a fresh chip or after changing clock settings in Arduino IDE — this sets the fuses that `millis()` and all timing depend on. Skipping it can make your code technically work but run 8x slower than intended, which looks like a mysterious hang.
7. **Never put a capacitor on RESET** — it interferes with the programmer's reset pulse and causes "target does not answer."
8. **USB switches/hubs can look like flaky hardware in `dmesg`** — rule out manual toggling before chasing a hardware fault.
9. Continuity testing with a multimeter is a real skill — don't over-trust a single reading if you're not experienced with the tool; corroborate with a second method (voltage checks, physical inspection, or swapping the whole component) before concluding.

---

## Possible Next Steps
- Add a physical button for on-demand refresh instead of/alongside the 30s auto-rotate
- Case/enclosure for portability
- Battery power (CR2032 coin cell or small LiPo) for a truly pocket-sized fob
- Flash Micronucleus bootloader once via ISP to get permanent USB-drag-and-drop uploading (no programmer needed for future code changes)
