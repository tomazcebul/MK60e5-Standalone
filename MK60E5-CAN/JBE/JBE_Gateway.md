# JBE: CAN frames and gateway table from the firmware

The junction box electronics (JBE, SGBD `JBBF70.prg`) is the CAN gateway between PT-CAN and K-CAN in the E8x/E9x. This page describes how its program stores the frames it sends and receives and the routing table it uses, and how jbe_gateway.py extracts them.

| | |
|---|---|
| Firmware | 9364820A, SPdaten E89 `data/JBBFR3/9364820A.0pa`, BMW release 23.10.2014, header "Standard-Core", device JBBFE-IIIR |
| Converted with | `PYTHONUTF8=1 python Shared\cmg5461-mk60e5-firmware\tools\bmw_hex2bin.py -o MK60E5\MK60E5-CAN\external\jbe\banks <file.0pa>` (flat image, base 0) |
| Names | JBBF70.prg table `ZUORDNUNG_CAN_ID_SG` (CAN ID, BMW frame name, sender), dumped with `E38-M67\DDE41A-RE\tools\sgbd.py` |
| Results | JBE_9364820A_own_frames.csv, JBE_9364820A_routing.csv |
| Regenerate | `python MK60E5\MK60E5-CAN\JBE\jbe_gateway.py [image.bin]` (needs the converted image and the SGBD dump in the git-ignored `external\jbe\` folder) |

Addresses on this page are those of 9364820A. Other JBBFR3 releases use the same code at other addresses; JBBF81 runs on an ST10 (C166) and was not analysed.

## CPU and CAN controllers

* 32-bit ARM, little-endian, ARM mode. RAM at 0xA0000000, peripherals at 0xFFFF8000. The source paths left in the image (`C:\2\standardcore\SC_src\gw\GwMain.c`, `GwTabelle.c`, `GwMsgObj.c`, `SendQueue.c`) show BMW's "Standard-Core" gateway software.
* Three Bosch C_CAN cells. The OS names its CAN interrupts `Can_CellIsr_Pt`, `Can_CellIsr_K` and `Can_CellIsr_D`; there is no F-CAN controller. The cell base addresses are in a table at 0x74AE0: cell 0 = 0xFFFFC800, cell 1 = 0xFFFFCC00, cell 2 = 0xFFFFC400.
* **Cell to bus (inference from content):**
  * Cell 0 is PT-CAN: its gateway inputs come from the DME (0x0A8/0x0A9/0x0AA, 0x1D0), the EGS (0x0BA) and the DSC (0x19E, 0x1A0).
  * Cell 1 is K-CAN: its inputs come from the CAS (0x130), the FZD (0x226) and the instrument cluster (0x2CA, 0x328), and its own send list holds the body frames 0x0E2–0x0FD.
  * Cell 2 is D-CAN (diagnosis): it has only dynamic message objects.
* The ARM core and C_CAN cells match an ST STR73x (inference, not confirmed).

## Frames the JBE sends and receives itself

| Table | Address | Layout |
|---|---|---|
| Send objects | 0x73CD0 (arbitration), 0x73D4A (DLC), 0x73D88 (RAM buffer) | `u16 ID << 2` per message object; the init at 0x1F138 writes it with 0xA000 (MsgVal, Dir = transmit) into the cell. Object ranges per cell from 0x74A18: 0–16 PT-CAN, 17–54 K-CAN, 55–60 D-CAN |
| Receive lookup | 0x73FEA (keys), 0x742B8 (receive index), 0x74586 (length), 0x745D0 (RAM buffer) | Open-addressing hash with 359 slots, used by 0x20398. Key = `ID \| cell << 11` |
| Own send IDs per bus | 0x74B46, ranges from 0x74B2C | The same IDs as the send objects without the diagnostic/NM objects; 0x2107C uses it to mark a frame for sending again when a received request (byte 0 = 1) names its ID (inference) |

Result: 15 send frames on PT-CAN, 36 on K-CAN; 28 receive buffers on PT-CAN, 46 on K-CAN (JBE_9364820A_own_frames.csv). The buffer of each frame is the start point for decoding its signals, as done for 0x252 in [SteeringColumnSwitchCluster.md](../../MK60E5-Components/SteeringColumnSwitchCluster.md#0x252-in-the-jbe-firmware).

## Gateway table

The init at 0x705F4 selects one of two configurations: **A** (header 0x76BE0) when bit 31 of the RAM word 0xA000060C is 0, **B** (header 0x7835C) when it is 1. Where that word comes from (probably coding) was not traced.

The header holds `u16` offsets relative to itself: +4 inputs PT-CAN, +6 inputs K-CAN, +8 outputs, +10 routes, +12 default blocks, +14 end.

| Table | Entry | Layout (from the code at 0x70714 and 0x6D380) |
|---|---|---|
| Inputs (one table per source bus) | 8 bytes | +0 `u16` CAN ID (sorted ascending, bucket start per `ID >> 8` cached at 0xA0005DF0/0xA0005DF8); +2 `u16` first route; +4 `u8` 1 = match byte 0 of the data; +5 `u8` number of routes; +6 `u8` value for byte 0; +7 `u8` special block (0xFF = none) |
| Routes | 6 bytes | +0 bits 0–2 copy kind (0 whole frame, 1 bit field, 2 one byte, 3 two bytes), bit 4 destination bus (0 PT-CAN, 1 K-CAN); +1 bits 0–2 send mode, bits 4–7 minimum source DLC; +2 output index; +3 destination position; +4 bit length (kind 1); +5 source position. Positions are bit numbers for kind 1, byte numbers for kinds 2 and 3 |
| Outputs | 4 bytes | +0 `u16` CAN ID (bits 0–10); +2 low nibble DLC, bit 4 flag (meaning not traced); +3 default block (0xFF = fill with 0xFF) |
| Default blocks | 16 bytes | bytes 0–7 initial content of the output frame; used instead of the source data when the received frame is shorter than the minimum DLC; byte 8–9 checked by the special handling at 0x70F8C |

Send modes (from the branches at 0x6D4E4 and 0x6D8DC):
* 1: send on every received frame.
* 3 and 4: send when the copied content changed.
* 0 and 2: update the output without sending it straight away. Presumably the cyclic task `GwCycTask` sends it (not traced).

The E92 M3 K-CAN trace fits this. 0x0C4 and 0x1A0 (mode 0) appear on K-CAN every 100–200 ms instead of every 10–20 ms on PT-CAN. 0x0C8 (mode 1) keeps its PT-CAN rate.

### Result

| | Config A | Config B |
|---|---|---|
| Input entries | 162 | 145 |
| Routes | 207 (109 PT→K, 98 K→PT) | 175 (94 PT→K, 81 K→PT) |
| Whole frames / bit fields / bytes | 156 / 31 / 20 | 139 / 21 / 15 |
| Output frames | 161 | 144 |

Full list: JBE_9364820A_routing.csv.
* Config B adds 0x180, 0x1A7, 0x201, 0x212, 0x2EF, 0x37F and 0x1AC → 0x0BE, and drops 0x195, 0x1E1, 0x1F6, 0x27x, 0x34x and others.
* Which configuration an E9x uses is open.

Frames the gateway changes:

| Source | Destination | What happens |
|---|---|---|
| K-CAN 0x2CA Außentemperatur (byte 0) + 0x328 Relativzeit (bytes 0–5) | PT-CAN 0x310, DLC 7 | Combined into one frame: temperature in byte 0, relative time in bytes 1–6 |
| PT-CAN 0x0BA Getriebedaten | K-CAN 0x304, DLC 2 | Two bit fields: bits 0–3 → bits 0–3, bits 60–61 → bits 4–5 (gear display, inference from the names) |
| PT-CAN 0x19E Status DSC, 0x1A0 Geschwindigkeit, 0x2B2 Raddrücke | same ID on K-CAN | Rebuilt field by field (17, 11 and 14 copies); bits not listed keep the default block content |
| PT-CAN 0x592, 0x596–0x5DE; K-CAN 0x586–0x5F8 ("Dienste") | same ID on the other bus | Only frames whose byte 0 matches a listed value are routed (e.g. 0x592 from the DME: 0x01, 0x0C, 0x10, 0x11, 0x30, 0x32, 0x40, 0x41, 0x65, 0x80) |
| Config B: PT-CAN 0x1AC | K-CAN 0x0BE | Renamed |

### Steering column switch cluster

* The JBE takes the SZL frames 0x0C8, 0x1D6 and 0x2A6 from **PT-CAN** and copies them to K-CAN. It has no F-CAN controller.
* 0x0C9, 0x194 and 0x1D9 are not in either configuration.
* The E92 320i PT-CAN log contains 0x0C4, 0x0C8, 0x194, 0x1D6 and 0x2A6. The F-CAN → PT-CAN copy is done by the DSC (MK60E5 firmware, confirmed in the code; see [Bus topology](../MK60E5_DSC–CAN-Interface.md#bus-topology)). 0x0C8 arrives every 200 ms with DLC 6, and the JBE passes it on per received frame (send mode 1), which is why K-CAN shows the same 200 ms.

### Comparison with our DBC files

The script lists the routed IDs that are missing from [BMW_E9x_PT-CAN_full.dbc](../BMW_E9x_PT-CAN_full.dbc) and [BMW_E9x_K-CAN_full.dbc](../BMW_E9x_K-CAN_full.dbc):
* Config A: 55 on PT-CAN and 43 on K-CAN, about half of them "Dienste" frames 0x58x–0x5Fx.
* These are frames the JBE forwards but no log or source in our set has shown yet.
* 17 routed IDs have no name in `ZUORDNUNG_CAN_ID_SG`, among them 0x1E1, 0x278, 0x27A, 0x340–0x34E, 0x3A8, 0x3CA, 0x3CC, 0x3F7 and 0x436.

## Open points

* Cell-to-bus assignment is inferred from the frames; the interrupt vector that ties `Can_CellIsr_Pt/_K/_D` to the cells was not traced.
* Source of the configuration switch (RAM 0xA000060C bit 31), and which configuration the E9x M3 uses.
* Meaning of output flag bit 4, of send modes 0 and 2, and of the special blocks (input +7).
* JBE family of the E9x M3 (JBBFE / JBBFEII / JBBFE3R) and release; JBBF70 9187546A and the JBBF81 (ST10) images were not analysed. A superwofy note reports 0x1D9 (M Drive button) on the K-CAN of an M3, but neither configuration of 9364820A routes 0x1D9, so the M3 JBE probably runs other software or another configuration.
