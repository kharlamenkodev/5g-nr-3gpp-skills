---
name: 5g-nr-rlc
description: Expert reference for the 5G NR Radio Link Control (RLC) protocol per 3GPP TS 38.322 v19.1.0. Use when the user asks about RLC modes (TM/UM/AM), PDU formats, state variables, timers, ARQ, segmentation, NTN/sidelink/MBS RLC, or implementing/debugging RLC entities.
---

# Skill: 5G NR RLC Protocol (TS 38.322 v19.1.0)

## Trigger
Use this skill when the user asks about:
- 5G NR RLC protocol implementation, architecture, or behaviour
- RLC modes: TM, UM, AM
- RLC PDU formats, headers, fields (TMD, UMD, AMD, STATUS PDU)
- RLC state variables, timers, or configurable parameters
- ARQ, polling, status reporting, retransmission
- SDU segmentation, reassembly, discard
- RLC for NTN (NB-IoT/NR over satellite), sidelink (V2X, ProSe), MBS (multicast/broadcast)
- Implementing or debugging RLC entities in C/C++, Python, MATLAB, or ns-3

## What to do
1. Answer strictly from TS 38.322 v19.1.0 (the authoritative source loaded below).
2. Cite clause numbers (e.g. "§5.2.3.2.3") when referencing normative behaviour.
3. Distinguish shall (normative) from should/may (non-normative).
4. When asked for implementation advice, map spec language to concrete code patterns.
5. Flag any NTN-specific considerations where relevant (extended timers, window sizes).

## Source
**3GPP TS 38.322 v19.1.0 (Release 19) — Feb 2026**
PDF: https://www.etsi.org/deliver/etsi_ts/138300_138399/138322/19.01.00_60/ts_138322v190100p.pdf
Related specs: TS 38.321 (MAC), TS 38.323 (PDCP), TS 38.331 (RRC)

---

## Reference: Complete Spec Summary

### 1. Architecture (§4.2)

Three RLC entity types:

| Mode | Abbreviation | ARQ | Segmentation | Header | Logical Channels |
|---|---|---|---|---|---|
| Transparent Mode | TM | No | No | None | BCCH, CCCH, PCCH, SBCCH |
| Unacknowledged Mode | UM | No | Yes | SN+SI+SO | DTCH, SCCH, STCH, MCCH, MTCH |
| Acknowledged Mode | AM | Yes | Yes | SN+SI+SO+P+D/C | DCCH, DTCH, SCCH, STCH |

- TM entity: configured as TX-only or RX-only
- UM entity: configured as TX-only or RX-only
- AM entity: bidirectional (TX+RX sides in same entity)
- RRC controls configuration of all RLC entities
- RLC PDUs submitted to MAC only when MAC signals a transmission opportunity

---

### 2. PDU Formats (§6.2.2)

#### TMD PDU
```
[ Data field (RLC SDU, no header) ]
```
No RLC header at all. SDU passed through unchanged.

#### UMD PDU (6-bit SN — complete SDU, SI=00)
```
[ R | R | SI(2b) | R | R | Data field ]
   ← 1 byte header →
```

#### UMD PDU (6-bit SN — with SN)
```
[ R | R | SI(2b) | SN(6b) | Data field ]
    ← 1 byte header →
```

#### UMD PDU (12-bit SN)
```
[ R | R | SI(2b) | R | R | R | R | R | SN(12b, split across 2nd byte) | (SO if SI≠00) | Data ]
    ←————— 2 byte header ——————→
```

**SI field values:**
- `00` = complete SDU (no SN field included)
- `01` = first segment
- `10` = last segment
- `11` = middle segment

#### AMD PDU
```
[ D/C(1b) | P(1b) | SI(2b) | SN(12b or 18b, MSB first) | (SO if SI≠00) | Data field ]
```
- **12-bit SN**: 2-byte header (D/C+P+SI+SN[11:8] in byte1, SN[7:0] in byte2)
- **18-bit SN**: 3-byte header
- SO (Segment Offset): 16-bit, present only when SI≠00 (i.e. when PDU is a segment)

#### STATUS PDU
```
[ D/C(1b) | CPT(3b) | ACK_SN(12b or 18b) | E1(1b) | padding ]
[ NACK_SN | E1 | E2 | E3 ] × N  (variable number of NACK entries)
[ SOstart(16b) | SOend(16b) ]    (present when E2=1)
[ NACK range(8b) ]               (present when E3=1)
```
- CPT = `000` for STATUS PDU (only value defined in Rel-19)
- ACK_SN = SN of next not-received SDU not in NACK list
- NACK_SN = SN of negatively acknowledged SDU
- SOstart/SOend = byte range within a partially received SDU
- NACK range = number of consecutive NACKed SNs (compact encoding)

---

### 3. Field Definitions (§6.2.3)

| Field | Size | Description |
|---|---|---|
| D/C | 1 bit | 0=control PDU, 1=data PDU |
| P | 1 bit | Poll bit — request STATUS report |
| SI | 2 bits | Segmentation Info: 00=complete, 01=first, 10=last, 11=middle |
| SN | 6/12/18 bits | Sequence Number (mode/config dependent) |
| SO | 16 bits | Segment Offset — byte position of first byte of segment |
| CPT | 3 bits | Control PDU Type (STATUS PDU = 000) |
| ACK_SN | 12/18 bits | Acknowledgement SN |
| E1 | 1 bit | Extension: 1 = NACK_SN+E1+E2+E3 follows |
| E2 | 1 bit | Extension: 1 = SOstart+SOend follows |
| E3 | 1 bit | Extension: 1 = NACK range follows |
| NACK_SN | 12/18 bits | Negatively acknowledged SN |
| SOstart | 16 bits | Start byte offset of missing segment |
| SOend | 16 bits | End byte offset of missing segment (0xFFFF = last byte) |
| NACK range | 8 bits | Number of consecutive NACKed SNs |
| R | 1 bit | Reserved = 0 |

---

### 4. State Variables (§7.1)

#### AM — Transmitting side
| Variable | Init | Description |
|---|---|---|
| TX_Next | 0 | SN to assign to next new AMD PDU |
| TX_Next_Ack | 0 | Lower edge of TX window; SN of next expected ACK |
| POLL_SN | 0 | Highest SN submitted when poll was set |
| PDU_WITHOUT_POLL | 0 | Counter: AMD PDUs sent since last poll |
| BYTE_WITHOUT_POLL | 0 | Counter: bytes sent since last poll |
| RETX_COUNT | 0 | Per-SDU retransmission counter |

#### AM — Receiving side
| Variable | Init | Description |
|---|---|---|
| RX_Next | 0 | Lower edge of RX window; first not-completely-received SN |
| RX_Next_Highest | 0 | Highest received SN + 1 |
| RX_Highest_Status | 0 | Highest SN allowed in ACK_SN field of STATUS PDU |
| RX_Next_Status_Trigger | — | SN after the one that triggered t-Reassembly |
| RX_Next_Discard_Trigger | — | SN after the one that triggered t-RxDiscard (if configured) |

#### UM — Transmitting side
| Variable | Init | Description |
|---|---|---|
| TX_Next | 0 | SN for next UMD PDU containing a segment |

#### UM — Receiving side
| Variable | Init | Description |
|---|---|---|
| RX_Next_Reassembly | 0 | Earliest SN still pending reassembly |
| RX_Next_Highest | 0 | Highest received SN + 1; upper edge of reassembly window |
| RX_Timer_Trigger | — | SN after the one that triggered t-Reassembly |

---

### 5. Constants (§7.2)

| Constant | SN bits | Value |
|---|---|---|
| AM_Window_Size | 12-bit | 2048 |
| AM_Window_Size | 18-bit | 131072 |
| UM_Window_Size | 6-bit | 32 |
| UM_Window_Size | 12-bit | 2048 |

**Window arithmetic**: all SN comparisons use modular arithmetic:
- 12-bit AM: modulo 4096
- 18-bit AM: modulo 262144
- 6-bit UM: modulo 64
- 12-bit UM: modulo 4096

---

### 6. Timers (§7.3)

| Timer | Entity | Purpose |
|---|---|---|
| t-PollRetransmit | AM TX | If poll ACK not received → retransmit |
| t-Reassembly | AM RX, UM RX | Gap detected → trigger status/discard after timeout |
| t-StatusProhibit | AM RX | Rate-limit STATUS PDU transmission |
| t-RxDiscard | AM RX | Discard AMD PDUs older than threshold (optional) |

All timer values configured by RRC via TS 38.331.

**NTN note**: t-Reassembly must be set large enough to accommodate the RTT:
- LEO (600 km): ~16–20 ms RTT → t-Reassembly ≥ 20 ms (terrestrial typical: 35 ms)
- GEO (35786 km): ~560 ms RTT → t-Reassembly ≥ 600 ms (may need application-level ARQ)

---

### 7. Configurable Parameters (§7.4, configured by RRC via TS 38.331)

| Parameter | Mode | Description |
|---|---|---|
| sn-FieldLength | UM, AM | SN bit width: UM=6 or 12; AM=12 or 18 |
| t-PollRetransmit | AM | Timer value |
| pollPDU | AM | Poll after every N PDUs |
| pollByte | AM | Poll after every N bytes |
| maxRetxThreshold | AM | Max retransmissions before RLF notification to upper layer |
| t-Reassembly | UM, AM | Reassembly timer value |
| t-StatusProhibit | AM | STATUS PDU rate limiter |
| t-RxDiscard | AM | Optional RX discard timer |
| stopReTxDiscardedSDU | AM | Stop retransmitting when PDCP discards SDU |

---

### 8. Key Procedures — Quick Reference

#### Entity Lifecycle (§5.1)
- **Establish**: init all state variables to 0; start procedures in §5.2
- **Re-establish**: discard all SDUs/PDUs; stop+reset all timers; reset all state vars
- **Release**: discard all; release entity

#### TM Transfer (§5.2.1)
- TX: submit SDU to MAC as-is (no header, no segmentation)
- RX: deliver PDU to PDCP as-is

#### UM TX (§5.2.2.1)
```
for each SDU from PDCP:
  assign SN = TX_Next to each UMD PDU segment
  if UMD PDU contains last segment of SDU:
    TX_Next = (TX_Next + 1) mod UM_modulus
  set SI field: 00=complete, 01=first, 10=last, 11=middle
  set SO = byte offset of segment within SDU
```

#### UM RX (§5.2.2.2)
```
on receive UMD PDU:
  if no SN in header (SI=00):
    deliver immediately to PDCP
  elif SN in [RX_Next_Highest - UM_Window_Size, RX_Next_Reassembly):
    discard (too old)
  else:
    place in reception buffer
    → run reassembly logic (§5.2.2.2.3)
    → manage t-Reassembly

on t-Reassembly expiry:
  advance RX_Next_Reassembly past gap
  discard all segments with SN < RX_Next_Reassembly
  restart t-Reassembly if still gaps present
```

#### AM TX (§5.2.3.1)
```
priority: STATUS PDU > retransmission > new data

for new SDU from PDCP:
  assign SN = TX_Next; TX_Next++
  do not submit AMD PDU if SN outside TX window [TX_Next_Ack, TX_Next_Ack + AM_Window_Size)

on positive ACK (STATUS PDU with ACK_SN):
  advance TX_Next_Ack to next un-ACKed SN
  notify PDCP of successful delivery

on NACK (STATUS PDU with NACK_SN):
  consider SDU/segment for retransmission
  increment RETX_COUNT
  if RETX_COUNT == maxRetxThreshold: notify upper layer (→ RLF)
```

#### AM RX (§5.2.3.2)
```
on receive AMD PDU (SN=x, bytes y..z):
  if x outside RX window OR duplicate bytes:
    discard
  else:
    place in reception buffer
    if all bytes of SN=x received:
      reassemble SDU → deliver to PDCP
      advance RX_Next, RX_Highest_Status as needed
    manage t-Reassembly and t-RxDiscard

on t-Reassembly expiry:
  update RX_Highest_Status
  trigger STATUS report

on t-RxDiscard expiry:
  discard all AMD PDUs with SN < RX_Next_Discard_Trigger
  advance RX_Next
  trigger STATUS report
```

#### ARQ — Polling (§5.3.3)
```
set P=1 in AMD PDU when:
  - PDU_WITHOUT_POLL >= pollPDU, OR
  - BYTE_WITHOUT_POLL >= pollByte, OR
  - TX+RETX buffers empty after this PDU, OR
  - window stall (no new SDU can be sent), OR
  - PDCP signals remaining-time-based polling

after setting P=1:
  POLL_SN = highest submitted SN
  (re)start t-PollRetransmit

on STATUS PDU received covering POLL_SN:
  stop t-PollRetransmit

on t-PollRetransmit expiry:
  retransmit highest SN or any un-ACKed SDU
  set P=1 in that PDU
```

#### ARQ — Status Reporting (§5.3.4)
```
trigger STATUS report when:
  - AMD PDU with P=1 received (after HARQ reordering)
  - t-Reassembly expires
  - t-RxDiscard expires

send STATUS PDU when:
  - t-StatusProhibit not running → send at next TX opportunity
  - t-StatusProhibit running → defer until it expires

STATUS PDU contains:
  ACK_SN: next SN not NACKed and not discarded
  NACK_SN list: all missing SNs in [RX_Next, RX_Highest_Status)
  SOstart/SOend: for partial segment losses
  NACK range: for consecutive missing SNs (compact)
```

#### SDU Discard (§5.4)
```
on PDCP indication to discard SDU:
  if SDU not yet submitted to MAC:
    discard immediately
  if AM and stopReTxDiscardedSDU configured:
    stop retransmitting submitted segments of that SDU
  NOTE: do not introduce SN gaps on TX side
```

---

### 9. Data Volume Reporting (§5.5)
RLC reports buffer size to MAC for BSR (Buffer Status Report):
- **BSR data volume** = un-PDU-ified SDUs + pending TX PDUs + pending retransmission (AM)
- **Delay-critical volume** (DSR, Rel-17+) = subset flagged by PDCP as delay-critical
- **Multi-entry DSR** = tiered thresholds per DSR-ReportingThreshold list
- If STATUS PDU triggered and t-StatusProhibit not running: add estimated STATUS PDU size to volume

---

### 10. Mode Selection Guide (for implementors)

| Use Case | RLC Mode | Rationale |
|---|---|---|
| SRB (RRC signalling) | AM | Reliability required |
| VoNR, streaming (DRB) | UM | Low latency; retransmission would be too late |
| URLLC | AM or UM | AM for reliability; UM+PDCP duplication for ultra-low latency |
| MBS multicast/broadcast | UM | No feedback channel from multiple receivers |
| Sidelink groupcast | UM (uni-dir) | No per-UE ACK in groupcast |
| NTN IoT (NB-IoT/eMTC) | AM | Reliability dominant; larger timer values required |
| NTN NR (LEO DRB) | AM with extended timers | t-Reassembly ≥ 2×RTT; t-PollRetransmit ≥ 2×RTT |

---

### 11. NTN-Specific Considerations (3GPP Rel-17/18/19)

- **Extended t-Reassembly**: standard range up to 200 ms (terrestrial); NTN requires values up to 3000 ms (configured via RRC)
- **Extended HARQ timers**: HARQ RTT up to 10.24 s for LEO; RLC must tolerate deep HARQ reordering before triggering STATUS
- **SN field length**: 12-bit SN for UM recommended for NTN to avoid SN wraparound in high-throughput LEO links
- **AM_Window_Size**: 18-bit SN (AM_Window_Size=131072) used in NTN high-throughput bearers to prevent window stall
- **t-PollRetransmit**: must be >> RTT to avoid spurious retransmissions (LEO: ≥ 25 ms; GEO: ≥ 600 ms)
- **stopReTxDiscardedSDU**: particularly useful in NTN to allow PDCP to signal SDU drops without wasting air link on stale retransmissions

---

### 12. Implementation Checklist

```
□ AM TX entity:
    - state: TX_Next, TX_Next_Ack, POLL_SN
    - counters: PDU_WITHOUT_POLL, BYTE_WITHOUT_POLL, RETX_COUNT (per SDU)
    - TX window enforcement: SN in [TX_Next_Ack, TX_Next_Ack + AM_Window_Size)
    - retransmission buffer + priority over new data
    - polling logic (pollPDU, pollByte, empty-buffer, window-stall)
    - t-PollRetransmit timer

□ AM RX entity:
    - state: RX_Next, RX_Next_Highest, RX_Highest_Status, RX_Next_Status_Trigger
    - optional: RX_Next_Discard_Trigger (when t-RxDiscard configured)
    - reception buffer (sparse, indexed by SN)
    - reassembly: segment tracking per SN (SOstart/SOend bitmask or interval list)
    - STATUS PDU construction (ACK_SN + NACK list with SO ranges + NACK range)
    - t-Reassembly, t-StatusProhibit, t-RxDiscard timers

□ UM TX entity:
    - state: TX_Next
    - segmentation based on MAC grant size
    - SI field encoding

□ UM RX entity:
    - state: RX_Next_Reassembly, RX_Next_Highest, RX_Timer_Trigger
    - reassembly window: [RX_Next_Highest - UM_Window_Size, RX_Next_Highest)
    - t-Reassembly timer

□ All entities:
    - modular SN arithmetic (correct modulus per mode/SN-length)
    - SDU discard on upper-layer indication (AM: no SN gap)
    - re-establishment: flush everything, reset state, reset timers
    - BSR data volume calculation (§5.5)
```
