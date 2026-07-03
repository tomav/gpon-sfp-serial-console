# SFP-to-TTL — Troubleshooting & FAQ

You powered a GPON/XGS-PON SFP ONU module through the
[SFP-to-TTL Adapter](https://tvi.al/sfp-to-ttl-adapter/) but you're not getting a serial
console? Work through this in order — the most common fix is first, and every step is safe
to try.

Full pinout and per-module reference: [`README.md`](README.md) · [`PINOUT.md`](PINOUT.md).

---

## First: what's actually working

The adapter is a **passive breakout PCB**. If the module **powers up and gets warm**, the
adapter is doing its job and power is fine. In ~95% of cases the problem is the **serial
link wiring or settings**, not the board. So don't worry — you're closer than you think.

---

## The 60-second checklist

Do these in order. Each is harmless.

1. **Confirm it's booting.** Module warm = powered = good.
2. **Swap TX ↔ RX** (the two data wires, pins 2 ↔ 7). *This is the single most common fix.*
   Which pin is TX vs RX is labeled inconsistently across modules, so just swap and retry.
3. **Add a common ground.** TX/RX alone don't work — connect the module's ground (any Vee
   pin: 1 / 10 / 11 / 14 / 17 / 20) to your adapter's / Pi's GND.
4. **Use 3.3 V only.** No Arduino Uno, no 5 V FTDI. Use a 3.3 V USB-TTL adapter
   (FTDI / CP2102 / CH340 set to 3.3 V) or the Raspberry Pi's GPIO UART.
5. **Serial settings:** 115200 baud, 8N1, no flow control.
6. **On a Raspberry Pi:** enable the real UART and get off the flaky mini-UART:
   - add `enable_uart=1` and `dtoverlay=disable-bt` to `/boot/config.txt`,
   - disable the serial **login shell** (`sudo raspi-config` → Interface → Serial →
     login shell **OFF**, serial hardware **ON**),
   - then `picocom -b 115200 /dev/ttyAMA0` (or `screen /dev/ttyAMA0 115200`).
7. **Still silent?** Take a clear photo of your wiring (both ends) noting which pin goes
   where — that's usually enough to spot it.

---

## Symptom → cause

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| Module warms up, **no serial output** | TX/RX swapped | Swap the two data wires (2 ↔ 7). #1 cause, harmless. |
| Module warms up, no output | **No common ground** | Connect module GND (a Vee pin) to adapter/Pi GND. |
| Module warms up, no output | Console not on pins 2/7 | Try SDA/SCL (4/5), then RX_LOS/RS1 (8/9). Break out all 20 and probe. |
| Using an **Arduino Uno** as bridge | Uno UART is **5 V** — wrong level, and not a transparent bridge | Use a 3.3 V USB-TTL or the Pi's GPIO UART instead. 5 V can damage the module. |
| **Garbage / gibberish** characters | Wrong baud, or 5 V logic | Set 115200 8N1. Confirm the adapter is 3.3 V. Some modules use 9600 / 38400. |
| Output shows, but **you can't type** | RX line missing, or flow control on | Wire module-RX ← adapter-TX. Disable HW/SW flow control. |
| Module **doesn't warm up** | No / weak power | +3.3 V on pin 15 or 16, GND on a Vee pin. GPON sticks pull up to ~1.5 A peak — a Pi 3.3 V rail is often too weak; use a dedicated 3.3 V supply. |
| Pi: nothing on `/dev/ttyS0`, unstable baud | Pi mini-UART on GPIO by default | `enable_uart=1` + `dtoverlay=disable-bt` → use `/dev/ttyAMA0` (stable PL011). |
| Pi: garbage / a login prompt fights you | Linux console (getty) owns the port | `raspi-config` → Serial → login shell OFF, hardware ON. |
| Module **reboots every few seconds** | OMCI / firmware — *not* the adapter | Realtek family needs `OMCI_FAKE_OK`; normal until registered on the network. |

---

## Per-module quick reference

Console pins, baud, quirks. Full list and Realtek-family notes in
[`README.md`](README.md#known-modules). **For any module: +3.3 V on pin 15/16, GND on a Vee
pin, console almost always on pins 2 & 7 — swap 2 ↔ 7 if silent.**

| Module | UART pins | Baud | Notes |
|--------|-----------|------|-------|
| Nokia / Alcatel **G-010S-A** | 2 & 7 *(family default; try 2, swap to 7)* | 115200 8N1 | Same Nokia family as G-010S-P. |
| Nokia / Alcatel **G-010S-P** | 2 & 7 (console on 2, swap to 7) | 115200 8N1 | — |
| Nokia / Alcatel **G-010S-Q** | 2 & 7 | 115200 8N1 | Realtek RTL9601CI inside. |
| Huawei **MA5671A** | 2 & 7 | 115200 8N1 | Stock firmware also has SSH (`root` / `admin123`). |
| **Realtek RTL960x family** (V2801F, TWCGPON657, DFP-34X, U-Fiber Instant, FS.com, CarlitoxxPro…) | 2 & 7 | 115200 8N1 | A permanent "TX Fault" state is normal (serial shares the TX_Fault pin). Needs `OMCI_FAKE_OK` or it auto-reboots. |
| **BFW WAS-110** (X-2010G-2, XGS-PON) | 2 & 7 | 115200 8N1 | Spam `Esc` at power-on for U-Boot. Console toggle: `8311_console_en`, `uart_select`. |

---

## Safety — read this before powering

- SFP modules are **3.3 V logic**. A 5 V adapter (Arduino Uno, a 5 V-only FTDI) can
  **permanently damage the module**. Always confirm your adapter is set to 3.3 V first.
- A Raspberry Pi's GPIO UART (GPIO 14 / 15) is 3.3 V — correct and safe.
- **Never** connect 5 V to any SFP pin.

---

## Still stuck?

Send a clear photo of the wiring on both ends and note which module pin connects to which
adapter pin. Nine times out of ten it's a swapped TX/RX or a missing ground.
