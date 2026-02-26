# Scheduling Reference — 5G NR MAC (TS 38.321 + TS 38.213)

## DL Scheduling Flow

```
gNB → UE: PDCCH [DCI Format 1_0 / 1_1 / 1_2]
              ↓ decoded with C-RNTI, CS-RNTI, or MCS-C-RNTI
         PDSCH [1 or 2 transport blocks, up to 8 HARQ layers]
              ↓
         HARQ-ACK on PUCCH (k1 slots after PDSCH)
```

### Key DCI Formats for Scheduling
| DCI Format | RNTI | Purpose |
|---|---|---|
| 0_0 | C-RNTI / CS-RNTI | UL scheduling (fallback, compact) |
| 0_1 | C-RNTI / CS-RNTI | UL scheduling (full, multi-BWP/SRS) |
| 0_2 | C-RNTI / CS-RNTI | UL scheduling (compact, Rel-16) |
| 1_0 | C-RNTI / CS-RNTI / P-RNTI / SI-RNTI / RA-RNTI | DL scheduling (fallback) |
| 1_1 | C-RNTI / CS-RNTI | DL scheduling (full) |
| 1_2 | C-RNTI / CS-RNTI | DL scheduling (compact, Rel-16) |
| 2_0 | SFI-RNTI | Slot format indication |
| 2_1 | INT-RNTI | Pre-emption indication |
| 2_2 | TPC-PUCCH-RNTI / TPC-PUSCH-RNTI | Transmit power control |
| 2_3 | TPC-SRS-RNTI | SRS TPC |

### Critical DCI Fields (DCI 1_1 — DL, DCI 0_1 — UL)
**Common to both:**
- **Frequency domain resource assignment**: RBs allocated (bitmap or start+len)
- **Time domain resource assignment**: row index into PDSCH/PUSCH-TimeDomainAllocationList (slot offset + symbols)
- **MCS** (5 bits): indexes into MCS table (64QAM or 256QAM tables per TS 38.214 §5.1.3/6.1.4)
- **NDI** (1 bit): toggle = new transport block
- **RV** (2 bits): 0/2/3/1 typical sequence; RV=0 for initial Tx
- **HARQ process number** (4 bits): 0–15

**DL-specific (DCI 1_1):**
- TB-to-codeword swap indicator
- Antenna port(s) + DMRS config
- PDSCH-to-HARQ-ACK timing (k1 index)
- Downlink assignment index (DAI): HARQ-ACK codebook position

**UL-specific (DCI 0_1):**
- SRS resource indicator (SRI): selects beam / precoder for PUSCH
- Precoding and number of layers
- PUSCH timing offset (k2): slot offset from DCI to PUSCH
- Beta offset indicator (for UCI on PUSCH)

---

## UL Scheduling Flow

```
UE → gNB: PRACH (random access) or SR (Scheduling Request on PUCCH)
gNB → UE: PDCCH [DCI Format 0_0 / 0_1]
              ↓
         PUSCH [1 TB, BSR/PHR MAC CE if triggered, data]
              ↓
         gNB decodes, sends HARQ feedback via PDCCH (DCI with DL assignment)
         or explicit ACK/NACK in DL control
```

---

## PDCCH and Search Spaces (TS 38.213 §10)

### Control Resource Set (CORESET)
- Defined by `controlResourceSet` in RRC: number of RBs, number of OFDM symbols (1–3), CCE-to-REG mapping
- CORESET 0: configured by MIB (initial access)
- CORESET 1–11: configured by RRC for dedicated UE

### Search Space Types
| Type | RNTI Monitored | Content |
|---|---|---|
| Common SS (CSS) | P-RNTI, SI-RNTI, RA-RNTI, TC-RNTI | Paging, SI, RAR, fallback DL/UL grant |
| UE-Specific SS (USS) | C-RNTI, CS-RNTI, MCS-C-RNTI | Dynamic grants, configured scheduling |

### Monitoring Occasions (TS 38.213 §10.1)
- Search spaces define: slot offset, monitoring symbols, aggregation levels (1/2/4/8/16 CCEs), monitored DCI formats
- UE blindly decodes PDCCH candidates at configured aggregation levels

---

## Configured Scheduling (Semi-Persistent, TS 38.321 §5.8)

### Activation / Deactivation
- **CS-RNTI** used in DCI to activate (NDI=1 for DL, specific field for UL) or deactivate (NDI=0)
- Once active, UE uses the periodicity from `configuredGrantConfig` / `semiPersistentOnPUSCH` RRC
- No PDCCH required for each grant — UE transmits/receives autonomously

### DL SPS (Semi-Persistent Scheduling)
- RRC configures: `sps-Config` (periodicity, HARQ process count, nrofHARQ-Processes)
- Activated/deactivated via DCI 1_1/1_2 with CS-RNTI
- HARQ-ACK still reported on PUCCH

### UL Configured Grant
- **Type 1**: fully RRC-configured, immediately active (no DCI activation needed)
- **Type 2**: RRC-configured, DCI 0_1/0_2 activates/deactivates with CS-RNTI
- Applications: VoIP, IoT periodic reporting, NTN uplink (reduces PDCCH overhead)

---

## Logical Channel Prioritization (LCP, TS 38.321 §5.22)

Governs how MAC PDU is assembled from multiple logical channels given a PUSCH grant.

### LCP Algorithm
Each logical channel has:
- **Priority** (1–16, 1 = highest)
- **PBR** (Prioritized Bit Rate): guaranteed bits per TTI interval
- **BSD** (Bucket Size Duration): 5/10/20/50/100/150/300/500/1000 ms → Bucket = PBR × BSD

**Step 1** — Fill buckets: serve each LC in priority order up to its bucket size (PBR)  
**Step 2** — Remaining grant: serve LCs in priority order, unlimited (highest-priority first)  
**Step 3** — Padding BSR (if grant still remaining after all data sent)

### LCP Restrictions (Rel-15+)
LCP is constrained by:
- `allowedSCS-List`: LC only mapped to grants with matching subcarrier spacing
- `maxPUSCH-Duration`: LC excluded if grant exceeds max allowed PUSCH duration
- `configuredGrantType1Allowed`: LC may be restricted to Type 2 configured grants only
- `allowedServingCells`: LC restricted to specific serving cells (CA)

### MAC CE Ordering in UL PDU (§6.1.2)
Priority ordering for inclusion in UL PDU when grant is limited:
1. Extended Power Headroom Report (ePHR) MAC CE
2. Power Headroom Report (PHR) MAC CE
3. C-RNTI MAC CE (if needed post-RACH)
4. Long BSR / Short BSR MAC CE
5. Logical channel data (high priority → low priority)
6. Padding BSR
7. Padding

---

## MCS and Link Adaptation

### MCS Tables (TS 38.214)
| Table | Max Modulation | Used When |
|---|---|---|
| Table 1 (§5.1.3.1-1) | 64QAM | Default |
| Table 2 (§5.1.3.1-2) | 256QAM | Configured via `mcs-Table` in RRC |
| Table 3 (§5.1.3.1-3) | 64QAM (low SE) | MCS-C-RNTI or `mcs-TableTransformPrecoder` |

### RV Sequence
Standard retransmission RV sequence: **0 → 2 → 3 → 1**  
- RV=0: self-decodable (systematic bits)  
- RV=2,3,1: parity bits (incremental redundancy)  
- Adaptive retransmission: gNB may change RV, frequency/time allocation  
- Non-adaptive retransmission: UE uses same resources + cyclic RV increment
