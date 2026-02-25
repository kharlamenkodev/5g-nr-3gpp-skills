# 5G NR 3GPP Skills

A collection of 3GPP-grounded knowledge skills for AI agents working on 5G NR protocol stack implementation. Each skill encodes the normative content of the relevant 3GPP specification as structured context, enabling agents to answer questions, generate code, and validate implementations against the standard.

## Structure

Each skill lives in its own directory and contains two files:

| File | Purpose |
|---|---|
| `SKILL.md` | Trigger conditions, instructions, and the full spec summary the agent uses at inference time |
| `CLAUDE.md` | Project context loaded automatically by Claude Code when working inside that directory |

## Available Skills

### `5g-nr-rlc/` — RLC Layer (TS 38.322 v19.1.0)

Radio Link Control — the layer between PDCP and MAC.

**Covers:**
- TM / UM / AM entity architecture and state machines
- PDU formats: TMD, UMD (6-bit and 12-bit SN), AMD (12-bit and 18-bit SN), STATUS PDU
- All state variables, constants, timers, and configurable RRC parameters
- Key procedures: segmentation, reassembly, ARQ polling, status reporting, SDU discard
- BSR data volume reporting
- NTN-specific considerations (LEO/GEO timer scaling, extended SN fields)
- Sidelink (V2X, ProSe) and MBS (multicast/broadcast) applicability
- Mode selection guide and implementation checklist

**Source:** 3GPP TS 38.322 v19.1.0, Release 19, February 2026

---

## Planned Skills

The following layers of the 5G NR user-plane and control-plane stack are planned:

| Layer | Spec | Description |
|---|---|---|
| **MAC** | TS 38.321 | Scheduling, logical channels, HARQ, BSR, SR |
| **PDCP** | TS 38.323 | Header compression (ROHC), ciphering, integrity, SN, reordering, discard |
| **RRC** | TS 38.331 | Radio resource control, configuration, measurement, mobility |
| **PHY** | TS 38.211 / 38.212 / 38.213 | Channels, waveforms, numerology, link adaptation |
| **SDAP** | TS 37.324 | QoS flow to DRB mapping |

## How to Use

### With Claude Code

Place the relevant `CLAUDE.md` in or above your working directory, or reference the `SKILL.md` directly as context. Claude will use the loaded skill to answer spec questions, cite clause numbers, and map normative language to implementation patterns.

### With Other Agents

Load `SKILL.md` as a system prompt or knowledge document. The skill includes explicit trigger conditions, source citations, and structured reference tables that work well with RAG or direct context injection.

## Spec Cross-References

The 5G NR stack specs are tightly coupled. Each skill notes where to look for adjacent behaviour:

| Spec | Layer | Key topics |
|---|---|---|
| TS 38.321 | MAC | Logical channels, HARQ, BSR, scheduling |
| TS 38.322 | RLC | TM/UM/AM, ARQ, segmentation |
| TS 38.323 | PDCP | ROHC, ciphering, SN management, discard |
| TS 38.331 | RRC | All radio configuration, measurement, mobility |
| TS 37.324 | SDAP | QoS flow mapping |
| TS 38.340 | BAP | IAB backhaul adaptation |
| TS 38.351 | SRAP | Sidelink relay adaptation |
| TR 21.905 | — | Vocabulary and abbreviations |
