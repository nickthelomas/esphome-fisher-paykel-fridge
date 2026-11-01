# Board layout (Coreva 101S-UCC perfboard)

**Coordinates are for the COPPER side facing you, with "101S-UCC" at the top.** Columns 1–16 run left to right
and rows 1–20 top to bottom (the printed row numbers). Parts go through from the plain side and are soldered on
the copper. Jumpers are insulated wire on the copper side.

Every pad on this board is two holes joined **across** (columns 2 and 3 in the same row are one pad). Rows are
separate pads, so "bridge rows 2 and 3" means a solder bridge **up and down** between two pads.

| Column | What it is |
|---|---|
| 1 | left edge strip: **GND** |
| 2, 3 | pads: BUS, F-S and R-S dividers |
| 4 | strip: **GND** |
| 5 | strip: **3V3** |
| 6, 7 | pads: XIAO D0–D6 in column 6, rows 2–8; column 7 is each pin's spare hole |
| 8 | strip: **GND** |
| 9 | strip: **3V3** |
| 10, 11 | pads (under the XIAO) |
| 12 | strip, **cut** into single holes at rows 2–8 for the XIAO's other row of pins |
| 13 | strip: **3V3** |
| 14, 15 | pads: CN11 D-TX divider and the probe socket |
| 16 | right edge strip: **GND** |

- **GND** = the row-1 bar plus columns 1, 4, 8 and 16.
- **3V3** = the row-20 bar plus columns 5, 9 and 13, fed from the XIAO's 3V3 pin.

The XIAO's two rows of pins are 6 holes apart, which doesn't line up with this board's pattern, so the strip in
column 12 is cut between rows 1 and 2, and between each pair of rows down to row 8.

## XIAO ESP32-C3 (in header sockets on the plain side, USB-C toward row 20)

| Row | Column 6 | Column 12 |
|---|---|---|
| 2 | D6 (GPIO21), not used | D7, not used |
| 3 | D5 (GPIO7) ← BUS | D8, not used |
| 4 | D4 (GPIO6), not used | D9, not used |
| 5 | D3 (GPIO5) ← DS18B20 probe | D10 (GPIO10) ← CN11 D-TX |
| 6 | D2 (GPIO4) ← F-S | 3V3 → bare link to row 6, column 13 |
| 7 | D1 (GPIO3) ← R-S | GND → insulated link to row 7, column 16 |
| 8 | D0, not used | 5V (power in, if not using USB-C) |

Check before you go further: with the XIAO plugged in and unpowered, row 7 column 16 should beep to the metal
shell of the USB-C socket.

## Dividers

Each one uses two bridged pads for the node that goes to the XIAO pin, and a separate pad for the fridge wire,
with an empty row between units. A "standing" resistor is mounted upright, with one leg bent over into the
neighbouring hole.

**BUS, rows 2–4, columns 2/3**
- Bridge rows 2 and 3.
- 33k: row 2, from column 2 to column 1.
- 100k: from row 3 column 2 to row 4 column 2.
- Jumper: row 3, from column 7 to column 3.
- Fridge BUS wire → row 4, column 3.
- Checks (XIAO out): row 3 column 6 to GND = 33k; row 4 column 3 to GND = 133k.

**F-S freezer, rows 6–8, columns 2/3**
- Bridge rows 6 and 7.
- 1M: row 6, from column 2 to column 1.
- 100 nF: row 7, from column 3 to column 4.
- 1M: from row 7 column 2 to row 8 column 2.
- Jumper: from row 6 column 7 to row 6 column 3.
- Fridge F-S wire → row 8, column 3.

**R-S fridge, rows 10–12, columns 2/3**
- Bridge rows 10 and 11.
- 1M: row 10, from column 2 to column 1.
- 100 nF: row 11, from column 3 to column 4.
- 1M: from row 11 column 2 to row 12 column 2.
- Jumper: from row 7 column 7 to row 10 column 3.
- Fridge R-S wire → row 12, column 3.

**CN11 D-TX, rows 2–4, columns 14/15**
- Bridge rows 3 and 4.
- 220k: row 3, from column 15 to column 16.
- 100k: from row 3 column 14 to row 2 column 14.
- Jumper: from row 5 column 12 (the D10 joint) to row 4 column 14.
- Fridge D-TX wire → row 2, column 15.
- Checks: row 2 column 15 to GND = 320k.

**DS18B20 probe socket, column 14, rows 9–11** (module pins S, V, G)
- 3-pin female header with its pins in column 14: row 9 = S, row 10 = V, row 11 = G.
- V: bare link in row 10, from column 13 to column 15.
- G: bare link in row 11, from column 15 to column 16.
- Jumper: from row 5 column 7 (the D3 spare hole) to row 9 column 15. Run it down below row 8 before crossing
  column 12.
- Checks: S ↔ row 5 column 6 beeps; V ↔ column 13 beeps; G ↔ column 16 beeps; S to GND and to 3V3 = open.
- A second socket for a second probe sits at rows 13–15, wired the same way (V and G links in rows 14 and 15),
  with a jumper from row 13 column 15 onto the first socket's S joint (row 9, column 14).

**Fridge GND** → row 19, column 1.

**Power**: 5 V from an isolated supply into the XIAO's USB-C, or into its 5V pin (row 8, column 12) with the
supply's 0 V to column 16. If you use the 5V pin, never plug USB in at the same time: on the XIAO the 5V pin is
the USB socket's power line.
