# BSR, PHR, Scheduling Request & Timing Advance — 5G NR MAC (TS 38.321)

## Buffer Status Report (BSR, §5.4.3)

### Logical Channel Groups (LCG)
- Logical channels are mapped to up to **8 LCGs** (LCG ID 0–7) via RRC (`logicalChannelGroup`)
- BSR reports aggregate buffer per LCG, not per individual logical channel
- If a logical channel has no LCG assignment, it is excluded from BSR

### BSR Types and Formats (§6.1.3.1)

| BSR Type | Subheader LCID | LCGs Reported | When Used |
|---|---|---|---|
| Short BSR | 62 | 1 LCG (highest-priority non-empty) | Enough grant for 1 LCG |
| Long BSR | 61 | All non-empty LCGs (1–8) | Grant permits full report |
| Short Truncated BSR | 60 | 1 LCG (highest-priority non-empty) | BSR must fit, not all LCGs possible |
| Long Truncated BSR | 59 | As many LCGs as fit in grant | Partial long BSR |
| Padding BSR | Short/Long | — | Remaining grant < Short BSR + data |

**Short BSR format:**
```
[ LCG ID (3b) | Buffer Size Index (5b) ]  — 1 byte
```
**Long BSR format:**
```
[ LCG 7 present (1b) | ... | LCG 0 present (1b) ]  — 1 byte bitmap
[ Buffer Size Index (8b) ] × N  — one per reported LCG
```

### Buffer Size Index Table (TS 38.321 §6.1.3.1, Tables 6.1.3.1-1 to -3)
- 3 tables based on LCG configuration (`logicalChannelSR-DelayTimerApplied`, extended table)
- Index 0 = 0 bytes (empty); index 1 = 10 bytes; exponentially increasing
- Index 255 = ≥ 81338368 bytes (81 MB); reported when actual buffer exceeds max table value
- UE selects the smallest index whose value ≥ actual buffer size (round up)

### BSR Triggers (§5.4.3.1)

| Trigger | Condition |
|---|---|
| Regular BSR | New higher-priority data available in previously empty LCG, OR data arrives and no LCG was served in previous transmission |
| Periodic BSR | `periodicBSR-Timer` expires (values: 5/10/16/20/32/40/64/80/128/160/320/640/1280/2560 ms, infinity) |
| Padding BSR | Remaining grant after data < Short BSR MAC CE; avoid wasting UL resources |
| Regular BSR (after SR) | BSR triggered when UL grant received following SR |

### BSR Cancellation
- BSR is cancelled if it is included in the UL MAC PDU being transmitted
- If a BSR is triggered while UL grant is already available but BSR cannot fit → may generate SR

---

## Scheduling Request (SR, §5.4.4)

### SR Purpose
Request a UL grant from gNB when UE has UL data but no available UL resource.

### SR Resources (PUCCH)
- Configured via `schedulingRequestToAddModList` (TS 38.331)
- PUCCH resource and periodicity per SR config ID
- UE may have multiple SR configurations (one per logical channel group or purpose)

### SR Procedure
```
1. BSR triggered with no UL grant → pending SR flag set
2. Wait for SR PUCCH resource → transmit SR
3. If no UL grant received within dsr-TransMax attempts:
   - Cancel SR
   - If UE has no valid UL timing: initiate RACH
   - If UE has valid UL timing: notify RRC (RLF indication)
4. On receiving UL grant → transmit PUSCH with BSR MAC CE
```
- `dsr-TransMax`: maximum SR attempts (4/8/16/32/64)
- `sr-ProhibitTimer`: prevents SR retransmission before timer expires

---

## Power Headroom Report (PHR, §5.4.6)

### PHR Types
| Type | What It Measures |
|---|---|
| Type 1 (PH for PUSCH) | Headroom when UE transmits PUSCH only |
| Type 2 (PH for PUCCH) | Headroom when UE simultaneously transmits PUCCH + PUSCH |

### PHR MAC CE Formats (§6.1.3.6, §6.1.3.7)
**Single-entry PHR** (Type 1 for PCell):
```
[ R | R | PH (6b) | R | R | PCMAX,f,c (6b) ]  — 2 bytes
```

**Extended PHR** (multi-cell, Type 1 + Type 2 per serving cell):
```
Bitmap of serving cells with PHR entry (1 or 2 bytes)
For each reported cell:
  [ P (Type2 valid) | V (virtual) | PH (6b) ]
  [ R | R | PCMAX,f,c (6b) ]  — 2 bytes per entry
```

### PH Calculation
- **Actual PH**: `PH = P_CMAX - P_PUSCH_estimated` (for scheduled PUSCH resources)
- **Virtual PH**: `PH = P_CMAX - P_0_PUSCH + PL` (estimated for a hypothetical reference format, when UE is not scheduled in UL)
- `V` bit in extended PHR: 1 = virtual PH reported (no actual PUSCH scheduled)
- **P_CMAX**: configured maximum output power per cell

### PHR Triggers (§5.4.6.1)
| Trigger | Condition |
|---|---|
| Regular PHR | `periodicPHR-Timer` expires |
| Regular PHR | Path-loss change exceeds `dl-PathlossChange` threshold |
| Regular PHR | SRS resources change |
| Prohibit | `prohibitPHR-Timer` running → no new PHR triggered |
| Activation of PHR | First PUSCH after activation / SCell addition |

### PHR Scale (6-bit field)
- Range: −32 to +31 dB (in 1 dB steps; negative = UE has no headroom / overloaded)
- Index 0 → PH ≤ −32 dB; index 63 → PH ≥ 31 dB

---

## Timing Advance (TA, §5.4.5)

### Purpose
Align UE's uplink transmission so it arrives at the gNB within the cyclic prefix, compensating for propagation delay.

### TA Command Sources

| Source | Bits | Range | Applied At |
|---|---|---|---|
| RAR (MSG2) | 12 bits (N_TA) | 0–3846 steps | Initial TA on access |
| MAC CE (TA Command) | 6 bits (TA_command) | 0–63 (adjustment) | Ongoing TA maintenance |

### TA Formula
```
Adjusted N_TA = N_TA_old + (TA_command − 31) × 16 × 64 / 2^μ × Ts
```
Where:
- μ = numerology (subcarrier spacing: 0=15kHz, 1=30kHz, 2=60kHz, 3=120kHz, 4=240kHz)
- Ts = 1/(Δf_ref × N_u_ref) ≈ 32.55 ns (basic time unit)
- TA_command = 31 → no change; < 31 → decrease TA; > 31 → increase TA

### t-TimeAlignmentTimer (§5.4.5.1)
- Started/restarted on each TA command reception
- On expiry: UE considers uplink out of sync → flush HARQ buffers, stop UL transmission, initiate RACH
- Values: 500/750/1280/1920/2560/5120/10240 ms, infinity

### NTN Timing Advance (Rel-17, §5.4.5.1a)
- UE pre-compensates TA from GNSS + satellite ephemeris (SIB19):
  ```
  TA_pre_comp = 2 × propagation_delay(UE_position, satellite_position) / Ts
  ```
- `ta-CommandProhibitTimer` (NTN): UE ignores TA commands during this window (avoids stale commands during fast satellite movement)
- `ta-ReportRequired` (NTN): gNB can request UE to report TA pre-compensation value
- For GEO: pre-compensation stable; for LEO: updated per ephemeris validity period (SIB19 validity timer)

---

## DRX (Discontinuous Reception, §5.7)

### DRX Cycle
```
Active Time (On Duration):
  UE monitors PDCCH every slot

Inactive Time (Off Duration):
  UE sleeps → power saving

DRX Cycle = On Duration + Off Duration
```

### DRX Timers
| Timer | Purpose |
|---|---|
| `drx-onDurationTimer` | Length of active period at start of DRX cycle |
| `drx-InactivityTimer` | Extended active period after receiving PDCCH with new data |
| `drx-HARQ-RTT-TimerDL` | Minimum time before gNB retransmits DL TB |
| `drx-HARQ-RTT-TimerUL` | Minimum time before gNB sends UL retransmission grant |
| `drx-RetransmissionTimerDL/UL` | Time UE monitors for retransmission after HARQ RTT timer |
| `drx-LongCycleStartOffset` | Long DRX cycle period and start offset |
| `shortDRX-Cycle` | Short DRX cycle (optional; more responsive, less saving) |

### Active Time Definition (§5.7)
UE is in Active Time when ANY of:
- `drx-onDurationTimer` or `drx-InactivityTimer` running
- `drx-RetransmissionTimerDL/UL` running (awaiting HARQ retransmission)
- PDCCH received indicating HARQ retransmission expected
- RA procedure active (MSG3 HARQ, contention resolution)
- SR pending
- Configured grant active (Type 1 or active Type 2)
- Downlink assignment index (DAI) pending (HARQ-ACK outstanding)

### NTN DRX
- `drx-HARQ-RTT-TimerDL/UL` must be ≥ propagation RTT + processing time
  - LEO: ≥ ~25 ms; GEO: ≥ ~600 ms
- `drx-RetransmissionTimerDL/UL` must cover variable HARQ scheduling delay
- Consequence: NTN DRX cycles are much larger → reduced power saving benefit at HARQ granularity; long DRX cycle (seconds) still beneficial for IoT

---

## Implementation Checklist

```
□ BSR:
    - Track buffer size per LCG (aggregate of all LCs in group)
    - Trigger logic: Regular (new high-priority data / gap in service), Periodic, Padding
    - Select BSR type: Short if only 1 LCG non-empty + grant permits; Long otherwise
    - Apply buffer size index table rounding (round up)
    - Cancel pending BSR on MAC PDU transmission

□ PHR:
    - Compute actual PH from estimated PUSCH power (from power control)
    - Set V=1 and report virtual PH when no actual PUSCH scheduled
    - Track per-serving-cell PHR for extended PHR (CA scenarios)
    - Respect prohibit timer before triggering new PHR

□ SR:
    - One SR config per PUCCH resource
    - Count SR transmissions; on reaching dsr-TransMax → RACH or RRC failure
    - Cancel SR on receiving UL grant

□ Timing Advance:
    - Apply TA on each MAC CE reception
    - Restart t-TimeAlignmentTimer on each TA command
    - On expiry: flush HARQ, stop PUSCH, trigger RACH
    - NTN: add GNSS-based pre-compensation; respect ta-CommandProhibitTimer

□ DRX:
    - Gate PDCCH monitoring on Active Time computation
    - Extend HARQ RTT timers for NTN values
    - Coordinate DRX wakeup with configured grant periodicity
```
