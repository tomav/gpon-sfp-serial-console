# SFP Connector Pinout — Diagram

Standard SFP / SFP+ 20-pin electrical interface (SFF-8074i / SFF-8419). Looking at the
**contact side** of the module (the gold fingers), latch facing away from you. The MSA
splits the pads into two rows; pins 1–10 are the lower row, 11–20 the upper.

```
  Module gold-finger edge (contact side up)
  ┌─────────────────────────────────────────────┐
  │ 11 12 13 14 15 16 17 18 19 20   (upper row) │
  │ 10  9  8  7  6  5  4  3  2  1    (lower row)│
  └───────────────────┬─────────────────────────┘
                      │  ← inserts into SFP cage / breakout
```

| Pin | Name        | 3.3 V? | Use when hunting for UART        |
|----:|-------------|:------:|----------------------------------|
| 1   | VeeT        | GND    | ground reference                 |
| 2   | TX_Fault    | LVTTL  | **console — most sticks (pair w/ 7)** |
| 3   | TX_Disable  | LVTTL  | input — usually not console      |
| 4   | SDA         | LVTTL  | UART candidate (I²C-pin variant) |
| 5   | SCL         | LVTTL  | UART candidate (I²C-pin variant) |
| 6   | MOD_ABS     | LVTTL  | presence — leave alone           |
| 7   | RS0         | LVTTL  | **console — most sticks (pair w/ 2)** |
| 8   | RX_LOS      | LVTTL  | UART candidate                   |
| 9   | RS1         | LVTTL  | UART candidate                   |
| 10  | VeeR        | GND    | ground reference                 |
| 11  | VeeR        | GND    | ground reference                 |
| 12  | RD−         | HS     | high-speed diff — not UART       |
| 13  | RD+         | HS     | high-speed diff — not UART       |
| 14  | VeeR        | GND    | ground reference                 |
| 15  | VccR        | +3.3 V | **power in**                     |
| 16  | VccT        | +3.3 V | **power in**                     |
| 17  | VeeT        | GND    | ground reference                 |
| 18  | TD+         | HS     | high-speed diff — not UART       |
| 19  | TD−         | HS     | high-speed diff — not UART       |
| 20  | VeeT        | GND    | ground reference                 |

**Rules of thumb**
- Power: +3.3 V to pin 15 or 16, ground to any Vee pin (1/10/11/14/17/20).
- The console is on the **LVTTL low-speed pins** — never the high-speed differential pairs
  (RD±/TD±, pins 12/13/18/19).
- 3.3 V logic only. A 5 V adapter can damage the module — set USB-TTL adapters to 3.3 V.

Sources: SFF-8074i (SFP MSA), SFF-8419 (low-speed electrical). Cross-checked against
multiple vendor SFP datasheets.
