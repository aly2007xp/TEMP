# dormakaba / Saflok card reader — repair log

Status: **powers up but accepts no cards — triage in progress (section 0).**

## 0. History and current symptom

- **Originally:** completely dead.
- **Then:** sent to an outside repair technician, who got it powering up. The hand-soldered wires (section 3) are probably his work.
- **Now:** powers up but **accepts no card**.

### Triage: what does it do when a card is presented?

| Reaction to a card | Meaning | Where to look |
|---|---|---|
| **No reaction at all** (no LED, no beep) | Card is not being detected | RF side: reader board, antenna, matching network (L1–L6, C1/C2...), transceiver QFN, the flex cable, wax-covered joints on the reader board |
| **Red LED / reject beep** | Card is read, then refused | Programming, not hardware: lost site data or wrong clock. Re-initialise from the property's own Saflok system |
| **Green LED but no click / door doesn't release** | Card is accepted, output fails | Relay (DSP1a-L2), its driver transistor, relay coil supply, J5 wiring |

### Questions for the repair technician
- What exactly did he replace or rewire, and why?
- Did he replace, reflash or touch the MCU, or the SOIC-8 `U1` (likely the EEPROM)?
- What are the two brown wires on the main board for?

Started 2026-11-04.

---

## 1. What the unit is

Three boards, identified from the silkscreen and labels in `photos/`:

| Board | Markings | Function |
|---|---|---|
| **Main / controller** | PCB `023-512734-3R2` · assy `124-512734-302` · firmware label `FW-513895-302` (11/20/19) | Mains PSU, MCU, lock relay, I/O |
| **RFID reader front end** | PCB fab code `JOVE JVE-M2 1933-18` | 13.56 MHz transceiver QFN + crystal + antenna matching network |
| **LED / interface** | `023-513200-2 Rev 1` · `SaflokK by KABA` · `A38480-SF-W` | Status LEDs D1/D2/D3, connector J1, flex tail |

The three are linked by the **16-pin flat-flex (FFC)** that lands in the ZIF connector at the top of the main board (silicone-staked at both ends from the factory).

### Main board, notable parts
- `FH1` fuse + red MOV + bridge — **this board runs directly off AC mains**
- Relay: Panasonic **DSP1a-L2** 2-coil latching, contacts rated **250 V AC**
- `J5` yellow terminal block, `J4` / `J8` white JST headers, `J?` 16-pin FFC
- `C8` 100 µF 25 V electrolytic (bulk rail), SOT-23 regulators `U5` etc.
- QFP MCU + SOIC `U1`, `U3`, crystal `X1`

---

## 2. ⚠️ Safety — read before touching it

The main board is **not** a low-voltage board. It has a line fuse, a MOV and a 250 V relay, with mains landing on the white/black wires at `H1`/`H2`.

- Unplug it from mains before any probing. Full stop.
- After unplugging, **discharge `C8` (100 µF 25 V)** and any bulk cap on the primary side before touching the board.
- Do all diagnostic bring-up from a **bench supply on the DC rail**, never from mains. There is no reason to energise the mains side to test the reader.
- If you ever must probe live, use an isolation transformer and a differential probe. Scope ground on a non-isolated mains board will make a very loud mistake.

---

## 3. Confirmed fault

### ⚪ `U3` area on the reader board — wax residue, not a confirmed fault

`photos/05-ZOOM-U3-wax-residue.jpg` shows a black glossy blob over `U3`, right of `D5`. It first looked like a ruptured package, but **the unit was originally covered in black wax**, so this is most likely leftover wax and `U3` may be fine underneath.

To confirm:
1. Soften the wax with gentle hot air (~80–100 °C, low flow) and wipe it off with a cotton swab. Finish with IPA or naphtha. Don't scrape — the pads lift easily.
2. Inspect `U3` at 10×: cracked body, bulging, burn marks or discoloured board underneath = failed. Clean, flat, intact = probably OK.
3. Clean the wax off the whole board the same way. Wax hides cracked joints, corrosion and burnt parts, and can trap moisture.

**No fault is confirmed yet.** Diagnosis now depends on the symptoms (section 9) and the measurements in section 5.

### 🟠 Prior repair attempts — treat as suspect

This unit has already been worked on by somebody, and not carefully:

- **Main board**, bottom right (`photos/01-main-board-top.jpg`): two lengths of stiff enamelled copper hand-soldered to the `H7`/`DM` and `J9`/`H5` pads, with heavy unwashed flux residue around the joints.
- **LED board** (`photos/04-led-board-and-flex.jpg`): a pink wire tacked onto a via near `D12`/`R27`.
- **Main board**, bottom left and the `TP94`/`TP95` through-holes: large reworked solder blobs.

Any of these bodges is a candidate for having put the wrong voltage onto the reader board's supply. **Buzz them out and work out what they were for before you power anything.**

### 🟡 Flex cable — probably fine, handle gently

The flex is soldered permanently to the LED board under silicone; only the far end is removable. The exposed fingers on the free end look intact and unworn. But the bend radius right at the solder joint is sharp, and flex traces crack there. Check continuity end-to-end on all 16 conductors before blaming anything else.

---

## 4. The one thing that decides whether this is worth fixing

The MCU on the main board holds the unit's firmware **and its site/property keying**. That is not field-reflashable and cannot be cloned off a donor.

- If the damage is confined to the **power path** (`U3`, `D5`, a regulator, a fuse), the MCU survives with its programming and the repair is genuinely viable.
- If the **MCU or the RFID transceiver QFN** is dead, replacing the silicon gets you a board that powers up and does nothing useful, because you cannot restore the firmware or the keying.

So: **repair passives and power components, never swap the MCU.** Establish early which case you are in — step 5.4 below is the test that tells you.

---

## 5. Diagnostic sequence

Work through in order. Record every reading in section 6.

### 5.1 Visual, before any power
- Inspect both sides of all three boards at 10× under good light.
- Look for: other cracked/discoloured packages, lifted pads, green/blue corrosion, cracked solder joints at the relay and the terminal blocks, hairline cracks in the flex.
- Clean every flux residue off the bodge joints with IPA and a brush so you can actually see the copper underneath.

### 5.2 Map the bodge wires
With everything unpowered, use continuity mode:
- Where does each brown wire actually go? Trace to a named net or test point at both ends.
- Same for the pink wire on the LED board.
- Write it down. If one of them bridges a supply rail to something it shouldn't, you have found your root cause.

### 5.3 Short hunt on the reader board (isolated)
Reader board **disconnected** from everything. Meter in resistance mode, black probe on a ground test point:
- Measure every one of the 16 flex-connector pads to GND.
- A pad reading **< 5 Ω to GND** is your shorted supply rail — almost certainly the one `U3` was feeding or fed from.
- Also diode-test `D5` and `D13` in both directions. A TVS or zener that failed short is a very common partner to a blown regulator, and may be the part that actually died first.

### 5.4 Is the main board alive? (the decisive test)
- **Unplug the flex**, isolating the reader board completely.
- Feed the main board's DC rail from a **bench supply with the current limit set low** (start ~100 mA). Do not use mains.
- Does the rail come up and hold? Does the MCU's crystal `X1` oscillate (scope, or check for the expected rail on `VDD`)? Do the LEDs do anything?

| Result | Meaning |
|---|---|
| Main board runs fine without the reader | Damage is contained to the reader board. **Best case — proceed to repair `U3`.** |
| Supply goes into current limit even with the flex unplugged | The main board is damaged too. Short-hunt it the same way before going further. |
| Main board dead but not shorted | MCU may be gone. Section 4 applies — stop and reconsider. |

### 5.5 Finding a short you can't see
Inject **1–2 V at ~1 A** into the shorted rail from the bench supply. The short dissipates the power and gets warm within seconds. Find it with a thermal camera, a fingertip, or a smear of IPA and watch where it flashes off first.

---

## 6. If a regulator such as `U3` does turn out to be dead

1. **Remove the remains.** Hot air at ~300 °C, low flow, shield the nearby QFN and the crystal with kapton. Lift the carcass with tweezers. Do not pry — the pads will come with it.
2. **Clean the footprint.** Solder braid plus flux, then IPA. Get it back to bare, bright pads.
3. **Photograph the bare footprint** and count the pads. Note the silkscreen pin-1 marker and the package outline. Add the photo to `photos/`. This identifies the package (SOT-23-3 / -5 / -6) and is the single most useful piece of evidence for picking a replacement.
4. **Trace the net of every pad** with a meter: which goes to the flex connector's supply pin, which to GND, which to the downstream rail that feeds the QFN transceiver.
5. **Measure the downstream rail's expected voltage.** Work it out from the QFN's requirements (read the part number off the transceiver — likely 3.3 V) and from any feedback resistors still on the board.
6. **Identify the replacement** from pinout + package + output voltage. With the footprint photo and the net map, the part is usually narrowable to a handful of standard LDOs.
7. **Fix the root cause first.** Do not fit the new part until step 5.2 has explained why the old one died.
8. **Bring up on the bench supply, current-limited**, flex still disconnected. Check the rail is correct and stable before reconnecting anything.

---

## 7. Tools you'll want

Essential:
- Bench PSU with adjustable current limit — this is the one tool that makes the difference between a repair and a second dead board
- Hot air rework station
- Fine-tip iron, good flux, solder braid, IPA + brush
- Multimeter with diode mode
- 10× loupe or a USB microscope

Very helpful:
- Thermal camera (even a cheap phone attachment) for short hunting
- Oscilloscope, for confirming the crystal oscillates and the 13.56 MHz carrier is present
- Kapton tape

---

## 8. Measurements — fill these in

| # | Measurement | Expected | Measured | Date |
|---|---|---|---|---|
| 1 | Flex pad ↔ GND, all 16, reader board isolated | no pad < 5 Ω | | |
| 2 | `D5` diode drop, both directions | one-way ~0.3–0.7 V | | |
| 3 | `D13` diode drop, both directions | one-way ~0.3–0.7 V | | |
| 4 | Brown bodge wire #1 — both endpoints | — | | |
| 5 | Brown bodge wire #2 — both endpoints | — | | |
| 6 | Pink bodge wire (LED board) — both endpoints | — | | |
| 7 | Flex continuity, all 16 conductors end to end | < 1 Ω each | | |
| 8 | Main board DC rail, flex unplugged, bench supply | stable, in spec | | |
| 9 | Main board current draw, flex unplugged | not in current limit | | |
| 10 | `U3` condition after wax removal | intact | | |
| 11 | RFID transceiver QFN part number | — | | |
| 12 | Downstream rail voltage after `U3` repair | likely 3.3 V | | |

---

## 9. Open questions

These change the plan and aren't answerable from the photos:

1. **What are the symptoms?** Completely dead, no LEDs? LEDs on but cards not read? Beeps but won't unlock? Intermittent?
2. **What happened before it failed?** Lightning, a power event, water, or did somebody's earlier repair attempt kill it?
3. **Did it ever work after those bodge wires went in?** If the previous repair is what killed it, those wires are the first thing to undo.
4. **What does the unit normally control** — a door strike, a maglock, an elevator relay? Determines what's on `J5` and the relay contacts.
5. **Is a donor unit available?** A second dead board of the same type is enormously useful for comparison measurements, even if it's also faulty.

---

## 10. Next actions

- [ ] Answer the questions in section 9
- [ ] Remove the black wax (gentle heat + IPA/naphtha), re-inspect all boards at 10×
- [ ] Check `U3` specifically once uncovered
- [ ] Map the bodge wires (5.2) — **do this before applying any power**
- [ ] Short hunt on the isolated reader board (5.3)
- [ ] Bench-supply bring-up of the main board with flex unplugged (5.4)
- [ ] Only if a dead part is found: replace it per section 6
