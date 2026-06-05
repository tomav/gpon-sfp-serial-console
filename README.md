# GPON SFP Serial Console — Pinout & UART Access Guide

A practical, vendor-neutral reference for accessing the **serial console (UART)** on
**GPON / XGS-PON SFP ONU "stick" modules** — the boards you flash with OpenWrt, custom
GPON firmware, or use to bypass an ISP's ONT.

This repo documents the **standard SFP 20-pin connector pinout**, explains **how the UART
is exposed** on SFP ONU modules, and walks through getting a serial console **without
soldering and without fiddly pogo-pins**.

> **TL;DR** — An SFP module exposes 20 edge pins. Several of the low-speed control pins
> can be repurposed by the module's SoC as a UART. Break those pins out, hook up a 3.3 V
> USB-TTL adapter, and you get a console. The hard part is breaking out the pins cleanly:
> that's what an [SFP-to-TTL breakout adapter](#hardware-the-easy-way) is for.

---

## Why you'd want this

GPON SFP ONU modules (Huawei, Nokia/Alcatel, ZTE, BFW, CarlitoxxPro builds, the
[8311 community](https://hack-gpon.org/) XGS-PON sticks, …) run a full Linux SoC. A serial
console lets you:

- read **bootloader / U-Boot logs** and catch boot errors,
- **flash or recover firmware** (OpenWrt, stock, community images),
- change the **PLOAM password / serial number / VLAN** to register on an ISP's network,
- debug a brick.

The catch: the UART isn't on a friendly header. It's on the **SFP edge connector**, which
is meant to slot into a cage — not into your bench. You need to break those 20 pins out.

---

## The SFP 20-pin pinout (SFF-8074i / SFF-8419)

This is the **standard SFP/SFP+ electrical interface**, identical across all compliant
modules. Pin 1 is the bottom row nearest the latch; orientation is defined by the MSA.

| Pin | Name      | Direction | Function                                   |
|----:|-----------|-----------|--------------------------------------------|
| 1   | VeeT      | —         | Transmitter ground                         |
| 2   | TX_Fault  | Out       | Transmitter fault indication (LVTTL)       |
| 3   | TX_Disable| In        | Transmitter disable (LVTTL)                |
| 4   | SDA (MOD-DEF2) | I/O  | I²C data — EEPROM @0xA0, DDM @0xA2         |
| 5   | SCL (MOD-DEF1) | In   | I²C clock                                  |
| 6   | MOD_ABS (MOD-DEF0) | Out | Module present (pulled to ground in module) |
| 7   | RS0       | In        | Rate select 0                              |
| 8   | RX_LOS    | Out       | Receiver loss-of-signal (LVTTL)            |
| 9   | RS1       | In        | Rate select 1                              |
| 10  | VeeR      | —         | Receiver ground                            |
| 11  | VeeR      | —         | Receiver ground                            |
| 12  | RD−       | Out       | Receiver data − (CML/LVPECL high-speed)    |
| 13  | RD+       | Out       | Receiver data + (CML/LVPECL high-speed)    |
| 14  | VeeR      | —         | Receiver ground                            |
| 15  | VccR      | —         | Receiver power, +3.3 V                      |
| 16  | VccT      | —         | Transmitter power, +3.3 V                   |
| 17  | VeeT      | —         | Transmitter ground                         |
| 18  | TD+       | In        | Transmitter data + (CML high-speed)         |
| 19  | TD−       | In        | Transmitter data − (CML high-speed)         |
| 20  | VeeT      | —         | Transmitter ground                         |

**Power it** from pin 15/16 (VccR/VccT, +3.3 V) referenced to any VeeR/VeeT ground.
⚠️ SFP is **3.3 V logic** — never drive it from a 5 V FTDI.

A clean diagram lives in [`PINOUT.md`](PINOUT.md).

---

## Where the UART hides

The SFP MSA does **not** define a UART. Module designers repurpose the low-speed,
LVTTL-level pins their SoC isn't otherwise using. In practice, across the most common
GPON and XGS-PON sticks, the console lands on the **same two pins**:

- **Pin 2 (TX_Fault) and pin 7 (RS0)** — by far the most common. Huawei MA5671A, the whole
  Realtek RTL960x family (V2801F, FS, C-Data…), Nokia G-010S-P and the MaxLinear-based
  WAS-110 all use this pair. See [Known modules](#known-modules).
- **SDA / SCL** (pins 4 / 5) — on modules that expose the console over the I²C pins.
- **RX_LOS / RS1** (pins 8 / 9) — occasionally.

**Heads-up:** which of pins 2/7 is TX vs RX is labeled inconsistently across sources and
breakout silkscreens. If you get power but no readable text, **swap the two wires** — it's
the single most common mistake. When in doubt, break out all 20 pins and probe.

### Finding TX/RX (the method)

1. Power the module (3.3 V on pin 15/16, GND on a Vee pin) and let it boot.
2. With a scope or logic analyzer, watch the LVTTL pins above for **bursts of activity a
   few seconds after power-up** — that's the bootloader printing. The one that toggles on
   its own is **TX** (module → you).
3. **RX** (you → module) is usually the adjacent pin in the same functional pair.
4. Connect module-TX → adapter-RX and module-RX → adapter-TX (always cross over). Common
   baud rate is **115200 8N1**; some modules use 9600 or 38400.

### Known modules

Verified against the [hack-gpon.org](https://hack-gpon.org/) wiki and the 8311 community.
TX/RX orientation varies by source/breakout — **swap pins 2↔7 if you get no output**. All
use **+3.3 V on pin 15/16** and **GND on a Vee pin (14/10/…)**. **PRs welcome** — add your
module.

| Module | SoC / standard | UART pins (SFP) | Baud | Notes |
|--------|----------------|-----------------|------|-------|
| **Huawei MA5671A** | Lantiq PEB98035 · GPON | **2 & 7** | 115200 8N1 | The most popular GPON stick. Stock firmware also has SSH (`root` / `admin123`). |
| **V-SOL V2801F** | Realtek RTL9601CI · GPON | **2 & 7** | 115200 8N1 | Realtek-family reference stick (full L2 bridge). Needs `OMCI_FAKE_OK` or it auto-reboots. |
| **T&W TWCGPON657** | Realtek RTL9601CI · GPON | **2 & 7** | 115200 8N1 | Same SoC as the V2801F, 16 MB NAND. Same auto-reboot fix. |
| **ODI DFP-34X-2C2** | Realtek RTL9601D · GPON | **2 & 7** | 115200 8N1 | Newer/cheaper, runs cool, transparent bridge. Best pick for 4-port ONU emulation. |
| **Ubiquiti U-Fiber Instant** (UF-INSTANT) | Realtek RTL9601CI · GPON | **2 & 7** | 115200 8N1 | Rebranded Realtek stick. |
| **FS.com GPON-ONU-34-20BI** | Realtek RTL960x · GPON | **2 & 7** | 115200 8N1 | FS.com rebrand of the Realtek family. |
| **CarlitoxxPro CPGOS03-0490 v2.0** | Realtek RTL9601C · GPON | **2 & 7** | 115200 8N1 | Reference hardware for Carlito firmware. |
| **Nokia / Alcatel G-010S-P** | GPON | **2 & 7** | 115200 8N1 | Console on pin 2; swap to 7 if nothing appears. The **G-010S-Q** variant is Realtek RTL9601CI. |
| **BFW WAS-110** (a.k.a. X-2010G-2) | MaxLinear PRX126 · XGS-PON | **2 & 7** | 115200 8N1 | The 8311-community flagship. Console can be toggled in U-Boot (`8311_console_en`, `uart_select`). Spam `Esc` at power-on to drop into U-Boot. |
| **FS.com XGS-ONU-25-20NI** | CIG · XGS-PON | 2 & 7 *(family default — confirm)* | 115200 8N1 | FS.com XGS-PON stick, CIG-based: dual firmware slots, mgmt IP `192.168.100.1`, runs `/mnt/rwdir/setup.sh` at boot. A bad flash that breaks the PON stack needs UART to recover. Exact UART pins not yet independently confirmed. |

> **Realtek RTL960x family** (RTL8672 / RTL9601C / RTL9601CI / RTL9601D): the V2801F,
> TWCGPON657, DFP-34X-2C2, U-Fiber Instant, FS.com, CarlitoxxPro, C-Data FD511GX-RM0,
> OPTON GP801R, BAUDCOM BD-1234-SFM, EXOT EGS1 and most rebrands are **electrically
> identical** — same pins (2 & 7), same baud, same `TX_Fault`-shared-with-serial quirk
> (the module sits in a permanent "TX Fault" state). They differ mainly in NAND size and
> firmware. These chips also have a broken EEPROM emulator, which is why Linux needs the
> RTL8672/RTL9601C SFP workaround to bring the link up.

---

## What you need

- A **GPON/XGS-PON SFP ONU** module (the thing you want a console on).
- A **3.3 V USB-TTL / USB-UART adapter** (FTDI FT232, CP2102, CH340 — set to **3.3 V**).
- **Dupont jumper wires**.
- A way to break out the SFP pins → see below.
- A serial terminal: `minicom`, `picocom`, `screen`, or PuTTY.

```bash
# Linux example
picocom -b 115200 /dev/ttyUSB0
```

---

## Hardware: the easy way

You can build your own breakout, but soldering to a 0.8 mm-pitch SFP edge is painful and
pogo-pin jigs drift. The simplest path is a purpose-built breakout board: slot the module
in, and all 20 pins land on labeled 2.54 mm headers.

➡️ **[SFP-to-TTL Adapter (v1.2)](https://tvi.al/sfp-to-ttl-adapter/)** — a small PCB with a Molex SFP slot that exposes all 20 pins on through-hole headers with **RX/TX silkscreen**,
no soldering required. Built and tested by the author of this guide. Ships worldwide.

*(Disclosure: this guide and that board are made by the same person. The pinout and method
above work with any breakout — use what you like.)*

---

## Related communities & resources

- [hack-gpon.org](https://hack-gpon.org/) — wiki on GPON/XGS-PON modules & ISP bypass
- [8311 community](https://8311.community/) — XGS-PON stick firmware project
- [OpenWrt forum](https://forum.openwrt.org/)
- [lafibre.info](https://lafibre.info/) (FR) — ISP fiber / ONT bypass discussions
- SFF specifications: SFF-8074i (SFP MSA), SFF-8419 (low-speed signals)

---

## License

Documentation in this repository is released under **CC BY 4.0** — reuse and adapt with
attribution. Linking back here is appreciated.

## Contributing

Found a module's UART pinout? Tested a baud rate? Spotted an error? Open an issue or PR —
this is meant to become *the* reference for serial access on SFP ONU modules.
