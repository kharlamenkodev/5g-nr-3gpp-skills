# Random Access Reference — 5G NR MAC (TS 38.321 §5.1)

## 4-Step RACH (Contention-Based and Contention-Free)

### Triggers for RACH (§5.1.1)
- Initial access (RRC Idle/Inactive → Connected)
- RRC re-establishment
- DL data arrival when UL sync lost
- Handover (UL sync to target cell)
- SR failure (no PUCCH resource or no response after max SR attempts)
- Beam failure recovery (BFR, Rel-16)
- Requests from upper layers (on-demand SI, positioning)

### 4-Step RACH Flow

```
MSG1 — UE → gNB: PRACH preamble
    - Select preamble index (0–63) from PRACH configuration
    - Contention-Based (CBRA): random selection from group A or B
    - Contention-Free (CFRA): gNB assigns specific preamble (e.g. handover, BFR)

MSG2 — gNB → UE: RAR (Random Access Response) on PDSCH, PDCCH via RA-RNTI
    - RA-RNTI = 1 + s_id + 14×t_id + 14×80×f_id + 14×80×8×ul_carrier_id
    - RAR window: ra-ResponseWindow (TS 38.331) — monitored on PDCCH after MSG1
    - RAR MAC PDU may contain multiple RAR entries (for different preambles)
    - RAR content per entry:
        ┌─ Preamble ID (6 bits) — matches MSG1 preamble
        ├─ Timing Advance Command (12 bits, N_TA granularity)
        ├─ UL Grant (27 bits): PUSCH resources for MSG3
        └─ TC-RNTI (16 bits): temporary identity until contention resolved

MSG3 — UE → gNB: PUSCH (using UL grant from RAR)
    - CBRA: includes CCCH SDU (RRC Setup Request, RRC Resume Request, etc.)
    - CFRA: may include C-RNTI MAC CE if UE already has a valid C-RNTI
    - HARQ retransmissions allowed for MSG3 (HARQ process 0)

MSG4 — gNB → UE: PDSCH via TC-RNTI (contention resolution)
    - CBRA: MAC CE = UE CONTENTION RESOLUTION IDENTITY (48 bits) = first 48 bits of MSG3
        UE compares with stored MSG3 content:
        Match → contention resolved, adopt TC-RNTI as C-RNTI (or keep existing C-RNTI)
        No match → back to MSG1, increment RA attempt counter
    - CFRA: no contention resolution MAC CE needed (preamble uniquely assigned)
    - HARQ-ACK for MSG4 sent on PUCCH
```

### Preamble Groups A and B (§5.1.2)
- **Group A**: smaller preamble sequences; default for most scenarios
- **Group B**: distinct set; selected when (message size threshold AND pathloss threshold) both met
- gNB interprets group selection as implicit UE buffer/pathloss indication

### RA Counter and Back-off
- `preambleTransMax`: max preamble transmission attempts before declaring failure → notify RRC
- Back-off: random wait drawn from [0, BI×bi_index] (BI = Back-off Indicator, 0–960 ms)
- Power ramping: each MSG1 attempt increases TX power by `powerRampingStep` (0/2/4/6 dB)

---

## 2-Step RACH (Rel-16, TS 38.321 §5.1.3)

Designed to reduce access latency by combining MSG1+MSG3 into MSGA and MSG2+MSG4 into MSGB.

### 2-Step RACH Flow

```
MSGA — UE → gNB: PRACH preamble (on dedicated 2-step RACH resources) + PUSCH payload
    - PRACH preamble: from 2-step-RACH-specific resource set (`msgA-PRACH-Configuration`)
    - PUSCH (appended): contains CCCH SDU or C-RNTI MAC CE + data
    - PUSCH resources: determined by preamble selection (`msgA-PUSCH-Config`)

MSGB — gNB → UE: PDSCH via MsgB-RNTI
    - MsgB-RNTI = f(PRACH slot, PRACH frequency, preamble ID) — like RA-RNTI but distinct
    - MSGB may contain:
        Case 1 — Success: RAR + contention resolution (merged)
            ┌─ TA command
            ├─ C-RNTI assignment
            └─ Optional DL/UL grant
        Case 2 — Fallback: Redirect to 4-step RACH
            → UE receives MSG2 (RAR with TC-RNTI), continues with MSG3/MSG4
```

### When to Use 2-Step vs 4-Step
| Condition | Preferred Mode |
|---|---|
| Strong coverage, low latency critical | 2-Step RACH |
| Large pathloss, weak signal | 4-Step RACH (more robust) |
| gNB unconfigured for 2-step | 4-Step RACH only |
| CFRA (handover, BFR) | 4-Step RACH (Rel-16 may use 2-step for BFR) |

---

## NTN Random Access Adaptations (Rel-17, §5.1.2.1a)

| Parameter | Terrestrial | NTN |
|---|---|---|
| RAR window (`ra-ResponseWindow`) | 10–80 slots | Extended up to 10240 ms (configured in SIB1-NTN) |
| PRACH guard interval | ~0.5 ms | Extended to cover max differential delay across cell |
| preambleTransMax | Same | Same; but backoff accounts for larger RTT |
| MSG3 k2 | 2–32 slots | Extended (up to 1024 slots) |
| Contention resolution timer | 64–4096 ms | Extended |
| RA preamble format | Short/long | Long preamble (format C0/C2) preferred for NTN coverage |

### NTN Cell Size and TA Range
- NTN cells can be very large (LEO footprint: 100–1000 km diameter)
- UE PRACH preamble timing must pre-compensate propagation delay:
  - UE computes PRACH transmission offset from GNSS + ephemeris (SIB19)
  - Remaining TA error corrected by RAR TA command
- N_TA_max: 12-bit RA TA field covers N_TA = 0–3846 × 16Tc steps

---

## Beam Failure Recovery (BFR, Rel-16, TS 38.321 §5.1.5)

Triggered when:
- Beam failure detection (BFD) timer expires (no valid RS from serving beams)
- Layer 1 indicates beam failure to MAC

### BFR Procedure
1. UE selects a candidate beam (NZP-CSI-RS or SS/PBCH block)
2. UE initiates RACH on that beam (using BFR-dedicated PRACH resources if configured)
3. gNB detects preamble, responds with RAR (CFRA path, no contention)
4. Recovery complete when PDCCH received on new beam

---

## RA-RNTI Calculation (TS 38.213 §8.1)

```
RA-RNTI = 1 + s_id + 14 × t_id + 14 × 80 × f_id + 14 × 80 × 8 × ul_carrier_id

Where:
  s_id = starting OFDM symbol of PRACH (0–13)
  t_id = slot of PRACH in a 1ms subframe (0–79, subcarrier spacing dependent)
  f_id = frequency index of PRACH (0–7)
  ul_carrier_id = 0 for NUL, 1 for SUL
```

---

## Implementation Checklist

```
□ PRACH selection:
    - Select correct preamble group (A or B) based on buffer size + pathloss
    - Power ramp on each attempt (track preamble TX power separately)
    - Monitor RA-RNTI on PDCCH within RAR window

□ RAR processing:
    - Parse MAC PDU: may have multiple RA preamble entries
    - Match preamble ID → extract TA command, UL grant, TC-RNTI
    - Apply TA command to uplink timing

□ MSG3 / Contention resolution:
    - Include UE identity (CCCH or C-RNTI MAC CE)
    - Store MSG3 content for contention resolution comparison
    - On MSG4: compare Contention Resolution Identity with stored MSG3 bytes
    - On match: promote TC-RNTI to C-RNTI

□ NTN:
    - Extend RAR window timer to configured NTN value
    - Pre-compensate PRACH timing from GNSS + SIB19 ephemeris
    - Use long PRACH preamble format (C0/C2)
```
