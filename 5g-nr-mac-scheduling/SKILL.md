---
name: 5g-nr-mac-scheduling
description: Expert knowledge of the 5G NR MAC layer and radio resource scheduling based on 3GPP TS 38.321 v19.1.0 (Release 19). Use this skill when the user asks about MAC sublayer architecture, logical/transport/physical channel mapping, downlink/uplink scheduling and DCI formats, HARQ process management and timing, logical channel prioritization and MAC PDU construction, Buffer Status Reports (BSR), Power Headroom Reports (PHR), Scheduling Requests (SR), random access procedures (4-step and 2-step RACH), Timing Advance commands, configured scheduling (CS-RNTI), semi-persistent scheduling, or NTN-specific MAC adaptations (HARQ disable, extended timers, pre-compensation). Also covers related PHY control procedures in TS 38.213 (PDCCH monitoring, search spaces, DCI formats, PUCCH, PRACH).
---

# Skill: 5G NR MAC Layer & Radio Resource Scheduling (TS 38.321 v19.1.0)

## Trigger
Use this skill when the user asks about:
- MAC architecture: logical/transport/physical channel mapping, MAC sublayers
- DL/UL scheduling: PDCCH, DCI formats, RNTI types, grant processing, MCS/RV selection
- HARQ: process model, NDI, RV, timing (HARQ-ACK), HARQ disabling for NTN
- Logical Channel Prioritization (LCP): PBR, BSD, LCG, MAC PDU construction
- BSR: types (Short/Long/Truncated), triggers, LCG-to-buffer mapping, MAC CE format
- PHR: Type 1/2, actual vs virtual, triggers, MAC CE format
- Scheduling Request (SR): SR resources, failure handling
- Random access: 4-step RACH (MSG1–4), 2-step RACH (MSGA/MSGB, Rel-16+)
- Timing Advance: RAR TA, MAC CE TA, t-TimeAlignmentTimer, NTN pre-compensation
- Configured scheduling / semi-persistent scheduling (CS-RNTI, RRC activation)
- NTN MAC: HARQ disable, extended HARQ process count, pre-compensation, NTN-specific procedures

## What to Do
1. Answer strictly from TS 38.321 v19.1.0 and TS 38.213 v19.x (authoritative).
2. Cite clause numbers (e.g. "§5.4.3.1") when referencing normative behaviour.
3. Distinguish **shall** (normative) from **should/may** (non-normative).
4. Map spec language to implementation patterns when asked for code advice.
5. Flag NTN-specific procedures and parameter ranges where relevant.
6. Cross-reference TS 38.213 for PDCCH/DCI/PUCCH, TS 38.214 for PDSCH/PUSCH, TS 38.211 for physical channels, TS 38.331 for RRC configuration.

## Primary Sources
| Spec | Layer | Version |
|---|---|---|
| **TS 38.321** | NR MAC | v19.1.0 (Release 19, 2026-02) |
| **TS 38.213** | PHY — Control Procedures | v19.x |
| **TS 38.214** | PHY — Data Channels | v19.x |
| **TS 38.211** | Physical Channels | v19.x |
| **TS 38.331** | RRC | v19.x |

---

## 1. MAC Architecture (TS 38.321 §4)

### Channel Mapping

| Logical Channel | Transport Channel | Physical Channel |
|---|---|---|
| BCCH | BCH | PBCH |
| BCCH | DL-SCH | PDSCH |
| PCCH | PCH | PDSCH |
| CCCH, DCCH, DTCH, MCCH, MTCH | DL-SCH | PDSCH |
| CCCH, DCCH, DTCH | UL-SCH | PUSCH |
| — | RACH | PRACH |
| SCCH, STCH (sidelink) | SL-SCH | PSSCH/PSCCH |

### Logical Channel Types
| Channel | Direction | Carried Data |
|---|---|---|
| BCCH | DL | Broadcast control (SIBs, MIB) |
| PCCH | DL | Paging |
| CCCH | DL+UL | Common control (RRC setup, re-establishment) |
| DCCH | DL+UL | Dedicated control (RRC Reconfiguration, etc.) |
| DTCH | DL+UL | Dedicated traffic (user data) |
| MCCH | DL | Multicast/broadcast control (MBS) |
| MTCH | DL | Multicast/broadcast traffic (MBS) |
| SCCH | Sidelink | Sidelink control |
| STCH | Sidelink | Sidelink traffic |

### MAC Sublayer Functions (§4.4)
- Mapping between logical and transport channels
- Multiplexing/demultiplexing of MAC SDUs into/from MAC PDUs
- Scheduling information reporting (BSR, PHR)
- Error correction via HARQ
- Priority handling (LCP) between logical channels
- Padding
- Random access
- Discontinuous reception (DRX)
- Timing Advance
- Beam failure recovery (Rel-16+)

---

## 2. MAC PDU Structure (§6.1)

### DL-SCH / UL-SCH MAC PDU
```
[ MAC subheader | MAC SDU or MAC CE ] × N
[ Padding MAC subheader | Padding ]   (optional)
```

**Subheader formats:**
```
Fixed-size CE or short SDU:
[ R | F=0 | LCID(6b) ]  — 1 byte

Variable-size CE or long SDU (L ≤ 255 bytes):
[ R | F=0 | LCID(6b) | L(8b) ]  — 2 bytes

Variable-size CE or long SDU (L > 255 bytes):
[ R | F=1 | LCID(6b) | L(16b) ] — 3 bytes
```

**LCID values (uplink, key)**:
| LCID | Meaning |
|---|---|
| 0–32 | Logical channel ID (for MAC SDU) |
| 59 | Long Truncated BSR |
| 60 | Short Truncated BSR |
| 61 | Long BSR |
| 62 | Short BSR |
| 63 | CCCH 48-bit / Padding |

### MAC CE Ordering in PDU (§6.1.2)
- **DL**: MAC CEs (with subheaders) precede MAC SDUs; padding last
- **UL**: MAC SDUs (high priority first) precede MAC CEs; padding last
  - Exception: BSR and PHR CEs may precede data when they justify inclusion

---

## 3. RNTI Types and Usage (TS 38.213 §16)

| RNTI | Purpose |
|---|---|
| C-RNTI | Cell-level unique UE identifier; used for dynamic scheduling |
| CS-RNTI | Configured scheduling RNTI; activates/deactivates semi-persistent grants |
| TC-RNTI | Temporary C-RNTI during random access |
| P-RNTI | Paging |
| SI-RNTI | System information |
| RA-RNTI | Random Access Response |
| MCS-C-RNTI | Alternate MCS table (low spectral efficiency) |
| SL-RNTI | Sidelink scheduling |
| INT-RNTI | Interruption indicator |
| SFI-RNTI | Slot format indication (for TDD) |
| SP-CSI-RNTI | Semi-persistent CSI on PUSCH |

---

## 4. References

For detailed topic coverage, read the relevant reference file:

| Topic | File | When to Read |
|---|---|---|
| DL/UL Scheduling, DCI formats, PDCCH, LCP, configured scheduling | `references/scheduling.md` | Scheduling grants, DCI field decoding, MAC PDU assembly with LCP |
| HARQ process model, NDI, RV, timing, NTN HARQ | `references/harq.md` | HARQ operation, timing diagrams, NTN HARQ disable |
| Random access (4-step, 2-step RACH, RAR, contention resolution) | `references/random-access.md` | RACH procedures, MSG1–4, MSGA/MSGB, beam failure recovery |
| BSR, PHR, Scheduling Request, Timing Advance, DRX | `references/bsr-phr-ta.md` | Buffer reporting, power headroom, SR failure, TA commands, DRX |

---

## 5. NTN-Specific MAC Overview (TS 38.321 §5.4, 3GPP Rel-17/18/19)

NTN introduces unique challenges due to large propagation delays. Key MAC adaptations:

| Aspect | Terrestrial | NTN LEO (~600 km) | NTN GEO (~35786 km) |
|---|---|---|---|
| One-way propagation delay | ~0.1 ms | ~2–10 ms | ~270 ms |
| HARQ RTT | ~4–8 ms | ~16–25 ms | ~560 ms |
| HARQ mode | Enabled | Enabled (extended timers) | **Disabled** (GEO) |
| HARQ process count | 8–16 | Up to 32 (Rel-17) | N/A (disabled) |
| t-Reassembly | ~35 ms | ≥ 25 ms | ≥ 600 ms |
| Timing Advance | gNB-commanded | **UE pre-compensates** via GNSS + ephemeris (SIB19) | Same |
| SR / BSR timing | Normal PUSCH timing | Extended k2 (PUSCH delay offset) | Extended k2 |
| Random access | Standard | Extended RAR window, RA preamble guard period | Extended |

### HARQ Disable (GEO, §5.4.2.2.1)
- HARQ is disabled on a per-TB basis via RRC: `harq-DisableNTN`
- When disabled: NDI always toggled → every TB treated as new data
- Retransmission handled at RLC (AM) or application layer
- HARQ feedback (HARQ-ACK) still reported to avoid UE ambiguity

### UE Pre-Compensation (Rel-17, §5.4.5.1a)
- UE computes uplink TA using GNSS position + satellite ephemeris from SIB19
- Pre-compensation = 2 × (propagation delay to reference point) / T_s
- gNB can additionally send TA command (MAC CE or RAR) for residual correction
- `ta-CommandProhibitTimer` prevents processing outdated TA during fast handover
