# HARQ Reference — 5G NR MAC (TS 38.321 §5.4.2, TS 38.213 §9)

## HARQ Model

NR uses **asynchronous** HARQ for both DL and UL:
- Each HARQ process is independently scheduled via PDCCH
- No fixed timing relationship between DCI and HARQ-ACK (unlike LTE synchronous UL HARQ)
- HARQ process number carried explicitly in DCI (4 bits → 16 processes max)

### HARQ Process Counts
| Direction | Default | Max (RRC configurable) | NTN Max |
|---|---|---|---|
| DL | 8 | 16 (`nrofHARQ-ProcessesDL`) | 32 (Rel-17, `nrofHARQ-ProcessesNTN`) |
| UL | 16 | 16 | 32 (Rel-17) |

> **NTN (Rel-17)**: up to 32 HARQ processes per cell to fill the larger HARQ RTT pipeline (needed for LEO where RTT ≥ 16 ms, vs ~4 ms terrestrial).

---

## NDI and New vs Retransmission

**NDI (New Data Indicator)** — 1 bit per TB in DCI:
- **Toggle relative to last reception** → new transport block (initial transmission)
- **Same value as last reception** → retransmission (HARQ process must chase-combine)

UE tracks NDI per HARQ process independently. On MAC entity establishment or HARQ process reset, NDI is initialized to a known state.

---

## DL HARQ Operation (§5.4.2.1)

```
gNB → UE: PDCCH (DCI 1_0/1_1) + PDSCH (HARQ process P, RV=r)
             ↓ UE decodes (CRC check)
UE → gNB: HARQ-ACK on PUCCH after k1 slots
           ACK  = CRC passed → process P marked as ACKed
           NACK = CRC failed → UE stores soft bits, awaits retransmission
             ↓ if NACK
gNB → UE: PDCCH + PDSCH retransmission (same HARQ P, new RV)
```

### HARQ-ACK Timing (k1)
- k1 defined in DCI field "PDSCH-to-HARQ-ACK timing indicator"
- k1 values: configured per `dl-DataToUL-ACK` list in PUCCH-Config (TS 38.213 §9.2.3)
- Typical values: k1 ∈ {1, 2, 3, 4, 5, 6, 7, 8} slots
- For NTN: k1 is extended to accommodate large RTT; configured up to hundreds of slots

### HARQ-ACK Codebook (TS 38.213 §9.1)
| Type | Description |
|---|---|
| Type 1 (semi-static) | Fixed-size codebook; bits for all configured HARQ processes, all configured DL BWPs; DAI not needed |
| Type 2 (dynamic) | Compact, only processes with pending HARQ-ACK; DAI in DCI tracks count to detect missed PDCCHs |

---

## UL HARQ Operation (§5.4.2.2)

```
gNB → UE: PDCCH (DCI 0_0/0_1) with PUSCH grant (HARQ P, NDI, RV, k2)
             ↓ k2 slots later
UE → gNB: PUSCH (HARQ process P, RV=r)
             ↓ gNB decodes (CRC check)
           if NACK: gNB sends new PDCCH with same HARQ P, NDI unchanged, new RV
```

### PUSCH Timing (k2)
- k2: slot offset from PDCCH slot to PUSCH slot
- Defined in `pusch-TimeDomainAllocationList` (TS 38.214 §6.1.2); k2 ∈ {0, 1, 2, ..., 32}
- NTN: k2 extended significantly (up to 1024 slots) to cover propagation delay

### UL HARQ Retransmission Types
- **Adaptive**: gNB issues new DCI with same HARQ P, NDI unchanged, potentially different resources/MCS
- **Non-adaptive** (UL only, legacy): UE retransmits on configured resources without new DCI (not common in NR except configured grants)

---

## HARQ in NTN (3GPP Rel-17/18/19, TS 38.321 §5.4.2.2.1)

### Why NTN HARQ is Challenging
| Parameter | LEO 600 km | GEO 35786 km |
|---|---|---|
| One-way delay | ~2–10 ms | ~270 ms |
| HARQ RTT | ~16–25 ms | ~560 ms |
| Processes to fill pipe (@ 100 Mbps) | ~4–8 | >> 16 |
| HARQ overhead | Manageable | Prohibitive |

### HARQ Disable Mode (GEO, §5.4.2.2.1)
- Configured per cell via `harq-DisableNTN` (RRC, TS 38.331)
- When enabled:
  - UE always treats incoming DL TB as new (ignores NDI)
  - No HARQ soft-buffer combining
  - HARQ-ACK still transmitted on PUCCH (set to ACK after single decode attempt)
  - Reliability offloaded to RLC AM layer
- gNB side: no HARQ retransmissions scheduled; relies on RLC ARQ

### Extended HARQ Process Count (Rel-17)
- `nrofHARQ-ProcessesNTN` (RRC): up to 32 DL and 32 UL processes
- Ensures pipeline stays full during RTT at peak throughput:
  - Required processes ≈ ceiling(RTT / TTI_duration)
  - LEO RTT 20 ms, TTI 0.5 ms (μ=1): ~40 processes needed → 32 is partial mitigation
  - GEO: HARQ disable is the only practical solution

### HARQ Timing Extensions (Rel-17)
- `dl-DataToUL-ACK` list extended to large k1 values (hundreds of slots)
- k2 list extended for UL grant-to-PUSCH timing
- Specific NTN timing values configured per orbit type in TS 38.331

---

## HARQ State Variables (per UE per process)

| Variable | Notes |
|---|---|
| NDI_rx[P] | Last received NDI for process P — compare to detect new vs retx |
| HARQ_buffer[P] | Soft buffer (LLR storage) for DL combining |
| RV[P] | Last used RV — used for non-adaptive UL retransmission |
| HARQ_acked[P] | Whether process P has been positively acknowledged |

---

## HARQ and DRX Interaction (§5.7)

During DRX (Discontinuous Reception):
- UE must wake for HARQ retransmissions within the **HARQ RTT timer** window
- `drx-HARQ-RTT-TimerDL` / `drx-HARQ-RTT-TimerUL`: minimum time before gNB may retransmit
- `drx-RetransmissionTimerDL` / `drx-RetransmissionTimerUL`: how long UE monitors after HARQ RTT timer for retransmission
- NTN: these timers must be extended to cover the large propagation delay + processing time

---

## Implementation Notes

```
□ DL HARQ entity per process:
    - track NDI, RV, HARQ buffer (LLR storage, size per TBS/modulation)
    - detect new TB: NDI toggled → flush buffer, begin fresh
    - detect retx: NDI same → soft-combine with stored LLR, run LDPC decoder
    - generate HARQ-ACK (ACK/NACK/DTX) per process per slot

□ UL HARQ entity per process:
    - track NDI, last grant, encoded bits (for non-adaptive retx)
    - on new grant (NDI toggle): encode fresh TB, transmit RV=0
    - on retx grant (NDI same): transmit stored bits with new RV

□ NTN HARQ disable mode:
    - receive every TB as if NDI toggled
    - do not perform soft-combining
    - set HARQ-ACK = ACK immediately after single decode attempt

□ Process count:
    - dim arrays to max configured process count (8/16/32 per direction)
    - validate HARQ process number in DCI ≤ configured max
```
