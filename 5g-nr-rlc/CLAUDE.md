# 5G NR RLC Protocol — Project Context for Claude Code

This project implements the 5G NR Radio Link Control (RLC) protocol as specified in
**3GPP TS 38.322 v19.1.0 (Release 19, February 2026)**.

Spec PDF: https://www.etsi.org/deliver/etsi_ts/138300_138399/138322/19.01.00_60/ts_138322v190100p.pdf

---

## Protocol Overview

RLC sits between PDCP (above) and MAC (below) in the 5G NR protocol stack.
Three modes: **TM** (transparent), **UM** (unacknowledged), **AM** (acknowledged).

Stack position: `PDCP ↔ [RLC: TM/UM/AM] ↔ MAC ↔ PHY`

---

## PDU Formats

### TMD PDU — No header, raw SDU passthrough
```
|<————————— Data (RLC SDU) —————————>|
```

### UMD PDU — 6-bit SN, complete SDU (SI=00)
```
| R  R  SI  R  R | Data |
  7  6  5 4  3 2    ...
      ← 1B hdr →
```

### UMD PDU — 6-bit SN, with SN field
```
| R  R  SI  SN[5:0] | (SO[15:0] if SI≠00) | Data |
      ← 1B hdr →           ← 2B SO →
```

### UMD PDU — 12-bit SN
```
| R  R  SI  R  R  R  R  SN[11:8] | SN[7:0] | (SO if SI≠00) | Data |
           ←——— 2B header ———→
```

### AMD PDU — 12-bit SN
```
| D/C  P  SI[1:0]  SN[11:8] | SN[7:0] | (SO[15:0] if SI≠00) | Data |
          ←—— 2B header ——→
```

### AMD PDU — 18-bit SN
```
| D/C  P  SI[1:0]  R  R  SN[17:16] | SN[15:8] | SN[7:0] | (SO if SI≠00) | Data |
              ←——————— 3B header ———————→
```

### STATUS PDU
```
| D/C(0)  CPT(000)  ACK_SN[11:0 or 17:0]  E1 | [NACK_SN  E1  E2  E3] × N |
  [SOstart(16b)  SOend(16b)]  [NACK_range(8b)]
```

**Field values:**
- SI: `00`=complete SDU, `01`=first seg, `10`=last seg, `11`=middle seg
- D/C: `0`=control (STATUS), `1`=data
- SO: byte offset of first byte of segment within the original SDU
- SOend=`0xFFFF` means "last byte of SDU"
- E1=1: another NACK entry follows; E2=1: SOstart/SOend present; E3=1: NACK range present

---

## State Variables

### AM TX
```c
uint32_t TX_Next;           // SN for next new AMD PDU (init: 0)
uint32_t TX_Next_Ack;       // lower TX window edge (init: 0)
uint32_t POLL_SN;           // SN set when poll was last transmitted
uint32_t PDU_WITHOUT_POLL;  // counter reset on each poll
uint32_t BYTE_WITHOUT_POLL; // counter reset on each poll
uint32_t RETX_COUNT[...];   // per-SDU retransmission counter
```

### AM RX
```c
uint32_t RX_Next;                   // lower RX window edge; first incomplete SN
uint32_t RX_Next_Highest;           // highest received SN + 1
uint32_t RX_Highest_Status;         // max SN allowed in ACK_SN
uint32_t RX_Next_Status_Trigger;    // SN that triggered t-Reassembly + 1
uint32_t RX_Next_Discard_Trigger;   // SN that triggered t-RxDiscard + 1 (optional)
```

### UM TX
```c
uint32_t TX_Next;   // SN for next UMD PDU with segment (init: 0)
```

### UM RX
```c
uint32_t RX_Next_Reassembly;  // earliest SN pending reassembly (init: 0)
uint32_t RX_Next_Highest;     // upper window edge (init: 0)
uint32_t RX_Timer_Trigger;    // SN that triggered t-Reassembly + 1
```

---

## Constants

```c
// AM window sizes
#define AM_WINDOW_SIZE_12BIT   2048
#define AM_WINDOW_SIZE_18BIT   131072

// UM window sizes
#define UM_WINDOW_SIZE_6BIT    32
#define UM_WINDOW_SIZE_12BIT   2048

// SN moduli
#define AM_MODULUS_12BIT       4096
#define AM_MODULUS_18BIT       262144
#define UM_MODULUS_6BIT        64
#define UM_MODULUS_12BIT       4096
```

---

## Timers (values set by RRC via TS 38.331)

| Timer | Used by | Purpose |
|---|---|---|
| `t-PollRetransmit` | AM TX | Retransmit if poll not acknowledged |
| `t-Reassembly` | AM RX, UM RX | Detect gaps; trigger status/discard |
| `t-StatusProhibit` | AM RX | Rate-limit STATUS PDU generation |
| `t-RxDiscard` | AM RX (optional) | Force-discard stale AMD PDUs |

**NTN timer guidance:**
- LEO 600 km RTT ≈ 16–20 ms → `t-PollRetransmit` ≥ 25 ms, `t-Reassembly` ≥ 25 ms
- GEO 35786 km RTT ≈ 560 ms → `t-PollRetransmit` ≥ 600 ms, `t-Reassembly` ≥ 600 ms
- NTN RRC configures `t-Reassembly` up to 3000 ms (extended range, Rel-17)

---

## Configurable Parameters (from RRC)

| Parameter | Mode | Notes |
|---|---|---|
| `sn-FieldLength` | UM, AM | 6 or 12 (UM); 12 or 18 (AM) |
| `t-PollRetransmit` | AM | ms; must be >> RTT for NTN |
| `pollPDU` | AM | Poll every N PDUs; `infinity` to disable |
| `pollByte` | AM | Poll every N bytes; `infinity` to disable |
| `maxRetxThreshold` | AM | 1..32; triggers RLF notify on breach |
| `t-Reassembly` | UM, AM | ms; 0=disable; NTN range up to 3000 ms |
| `t-StatusProhibit` | AM | ms; 0=no rate limiting |
| `t-RxDiscard` | AM | ms; optional; absent=feature disabled |
| `stopReTxDiscardedSDU` | AM | Boolean; stop retransmitting PDCP-discarded SDUs |

---

## Key Procedures (normative — §5)

### SN window membership (modular)
```c
// AM receive window: RX_Next <= SN < RX_Next + AM_Window_Size
bool am_in_rx_window(uint32_t sn, uint32_t rx_next, uint32_t mod) {
    return ((sn - rx_next) % mod) < AM_WINDOW_SIZE;
}

// UM reassembly window: RX_Next_Highest - UM_Window_Size <= SN < RX_Next_Highest
bool um_in_reassembly_window(uint32_t sn, uint32_t rx_next_highest, uint32_t win, uint32_t mod) {
    return ((sn - (rx_next_highest - win + mod)) % mod) < win;
}
```

### AM retransmission trigger (§5.3.2)
```
NACK received for SN=x:
  if TX_Next_Ack <= x <= highest_submitted_SN:
    if stopReTxDiscardedSDU NOT set OR no discard indication received:
      mark SDU(x) for retransmission
      if first retransmit: RETX_COUNT[x] = 0
      else: RETX_COUNT[x]++
      if RETX_COUNT[x] == maxRetxThreshold: notify RRC/PDCP (→ possible RLF)
```

### Polling decision (§5.3.3.2)
```
set P=1 when ANY of:
  PDU_WITHOUT_POLL >= pollPDU
  BYTE_WITHOUT_POLL >= pollByte
  (TX_buffer EMPTY AND RETX_buffer EMPTY) after this AMD PDU
  window stall (SN would exceed TX_Next_Ack + AM_Window_Size)
  PDCP remaining-time-based polling indication
```

### STATUS PDU construction (§5.3.4)
```
ACK_SN = first SN > all contiguous received SNs starting from RX_Next
         (i.e. first not-received, not-NACKed SN)
For each SN in [RX_Next, RX_Highest_Status) with missing bytes:
  - add NACK_SN
  - if partial: add SOstart + SOend (set E2=1)
  - if consecutive missing SNs: use NACK range (set E3=1)
Send only when t-StatusProhibit not running.
After send: start t-StatusProhibit.
```

---

## Mode Selection

| Bearer | Mode | Reason |
|---|---|---|
| SRB (RRC) | AM | Must be reliable |
| VoNR DRB | UM | Retransmission arrives too late |
| Data DRB (TCP) | AM | TCP needs reliable transport |
| URLLC DRB | AM or UM+PDCP duplication | Use case dependent |
| MBS MTCH | UM | Broadcast — no feedback |
| Sidelink groupcast | UM (unidirectional) | No per-UE ACK |
| NTN NR DRB | AM (extended timers) | Reliability + large RTT |
| NTN NB-IoT | AM (Rel-17/18 IoT NTN) | Power-optimised + reliable |

---

## Spec Cross-References

| Topic | Spec |
|---|---|
| MAC scheduling, logical channels, BSR | TS 38.321 |
| PDCP (ROHC, ciphering, SN, discard) | TS 38.323 |
| RRC configuration of RLC | TS 38.331 |
| BAP (IAB backhaul RLC channels) | TS 38.340 |
| SRAP (sidelink relay RLC channels) | TS 38.351 |
| V2X / sidelink services | TS 23.287 |
| ProSe (proximity services) | TS 23.304 |
| Vocabulary | TR 21.905 |

---

*Source: 3GPP TS 38.322 v19.1.0, Release 19, February 2026*
*Always verify against the authoritative PDF for normative behaviour.*
