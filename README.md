# Cognitive Bullwhip Diagnostics

**Deterministic middleware for detecting and correcting reasoning failures in LLM-based agent systems.**

[![MCP Compatible](https://img.shields.io/badge/MCP-compatible-blue)](https://modelcontextprotocol.io/)
[![Tests](https://img.shields.io/badge/tests-58%20passing-green)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## The Problem

In supply chain management, the **Bullwhip Effect** describes how small demand fluctuations at the retail level amplify into massive production swings upstream — a 5% change in consumer demand can cause a 40% production variance at the manufacturer (Lee, Padmanabhan & Whang, 1997).

The same amplification dynamic exists in **multi-layer AI agent systems**. A minor misclassification at the input layer propagates through reasoning, execution, and output — compounding at each stage. By the time the failure is visible, the root cause is buried 3-4 layers back. Most debugging targets the symptom layer (output), not the origin layer (input or reasoning).

This project provides **6 deterministic MCP tools** that diagnose where amplification starts, classify signals before execution, enforce reasoning structure, simulate downstream impact, and validate against governance rules — all without using an LLM inside the tools themselves.

## Theoretical Framework

### Cognitive Bullwhip Effect

The Cognitive Bullwhip Effect maps the supply chain amplification model onto AI agent decision pipelines:

| Layer | Supply Chain Analog | Agent System | Failure Mode |
|-------|-------------------|--------------|--------------|
| **Input** | Retail demand signal | Raw prompt / event / data | Noise treated as signal |
| **Reasoning** | Distributor forecasting | Context retrieval + analysis | Step-skipping, inconsistent logic |
| **Execution** | Manufacturer production | API calls, DB writes, tool use | Local optimization, side effects |
| **Output** | Supplier raw materials | Final decision / response | Governance violations, drift |

**Amplification ratio** = output variance / input variance at each layer. A ratio > 3.0x at any single layer indicates active bullwhip.

### Four Pattern Types

| Pattern | Origin Layer | Mechanism | Intervention |
|---------|-------------|-----------|-------------|
| **Noise Sensitivity** | Input | Agent reacts to every fluctuation as a command | Signal classification (anchor_classify) |
| **Reasoning Drift** | Reasoning | Non-deterministic logic produces different outputs per run | Sequence enforcement (logic_sequence) |
| **Myopic Optimization** | Execution | Local task completion breaks downstream systems | Impact simulation (mesh_simulate) |
| **Misaligned Autonomy** | Output | Decisions violate governance rules without audit trail | Rule validation (gate_validate) |

## Architecture

```
Raw Input
    │
    ▼
┌─────────────────┐
│  anchor_classify │  Signal Classification
│  Action / Obs /  │  Noise isolation, confidence scoring
│  Ambiguous       │  → blocks ambiguous signals
└────────┬────────┘
         │ (if Action)
         ▼
┌─────────────────┐
│  logic_sequence  │  Reasoning Enforcement
│  Context →       │  4-step fixed sequence
│  Retrieval →     │  Historical consistency check
│  Analysis →      │  → blocks on step-skip
│  Action          │
└────────┬────────┘
         │ (if pass)
         ▼
┌─────────────────┐
│  mesh_simulate   │  Impact Simulation
│  Risk 0-100      │  Maps all downstream system nodes
│  Horizon analysis│  → adjusts action if risk > threshold
└────────┬────────┘
         │ (if safe)
         ▼
┌─────────────────┐
│  gate_validate   │  Governance Validation
│  Principles check│  Keyword matching (morphology-aware)
│  Audit trail     │  → blocks/escalates on violation
└────────┬────────┘
         │
         ▼
    Final Decision
```

The **sc_pipeline** tool chains all four stages with automatic gating — if any stage returns `block` or `flag`, downstream stages are skipped and the blocking reason is propagated.

The **bullwhip_diagnose** tool operates independently as a diagnostic layer, scanning historical decision logs to detect active amplification patterns and recommend which intervention tool to apply.

## Tools

| Tool | Function | Output |
|------|----------|--------|
| `bullwhip_diagnose` | Scans decision history for amplification patterns across 4 layers | Severity 0-100, amplification map, pattern type, recommended fix |
| `anchor_classify` | Classifies input as Action / Observation / Ambiguous before execution | Signal type, confidence score, noise isolation log |
| `logic_sequence` | Enforces Context → Retrieval → Analysis → Action sequence | Completed steps, risk horizon, historical consistency check |
| `mesh_simulate` | Simulates downstream impact across all connected system nodes | Risk score 0-100, impact map, adjusted recommendation |
| `gate_validate` | Validates against user-defined governance rules with morphology-aware keyword matching | Approve / Escalate / Block + timestamped audit trail |
| `sc_pipeline` | Chains all 4 core tools with auto-gating | Per-stage results, final decision, blocking reason if any |

### Design Principles

- **100% Deterministic**: All scoring uses pure threshold-based analysis — no LLM calls inside any tool
- **Dual-block Output**: Each tool returns (1) human-readable diagnostic report + (2) structured JSON for programmatic use
- **Auto-gating Pipeline**: If any stage blocks, downstream stages are skipped — no wasted computation
- **Morphology-aware Matching**: Keyword detection catches inflected forms (e.g., "deletion" from "delete", "overwritten" from "overwrite")

## Quick Start

Add to your MCP client configuration (Claude Desktop, Cursor, etc.):

```json
{
  "mcpServers": {
    "structured-cognition": {
      "command": "npx",
      "args": ["@agdp/structured-cognition"]
    }
  }
}
```

Or run directly:

```bash
npx @agdp/structured-cognition
```

## Case Study: Kalshi Crypto Trading Bot

A 4-agent trading pipeline operating on Kalshi 15-minute binary crypto contracts experienced an 80% capital loss ($2,000 → $396) despite maintaining a 56% win rate. The loss severity consistently exceeded win magnitude — the system was profitable in frequency but destructive in amplitude.

**Diagnosis** (`bullwhip_diagnose`):

```
Status:      ACTIVE (Severity 100/100, immediate)
Origin:      input — noise_sensitivity
Ratio:       18.75x amplification at input layer

Amplification Chain:
  input:     0.021 → 0.193  (18.75x)
  reasoning: 0.496 → 1.474  (5.33x)
  execution: 0.353 → 2.650  (7.62x)
```

**Finding**: The input layer was amplifying minor price fluctuations (±0.3%) into high-confidence trade signals. The 18.75x amplification ratio at the input layer confirmed a classic noise sensitivity pattern — the bot was reacting to market noise as if it were actionable signal.

**Prescribed intervention**: `anchor_classify` — classify each market event as Action / Observation / Ambiguous before allowing downstream execution.

Full case study: [`test/kalshi-case-study.ts`](test/kalshi-case-study.ts)

## Testing

```bash
npm test     # 58 E2E tests across all 6 tools
```

Test coverage includes:
- Empty, clean, and amplified decision logs (bullwhip_diagnose)
- Ambiguous, clear, spike, and dangerous inputs (anchor_classify)
- Full context, missing context, and structural risk scenarios (logic_sequence)
- Low risk, batch operations, and API calls (mesh_simulate)
- Pass, escalate, block, low confidence, and keyword matching (gate_validate)
- Clean pass, ambiguous stop, and gate escalation pipelines (sc_pipeline)

## Development

```bash
git clone https://github.com/JIPRO-AI/Cognitive-Bullwhip-Diagnotics.git
cd Cognitive-Bullwhip-Diagnotics
npm install
npm run dev     # Development mode (tsx)
npm run build   # TypeScript compile
npm test        # 58 E2E tests
```

## References

- Lee, H.L., Padmanabhan, V., & Whang, S. (1997). "Information Distortion in a Supply Chain: The Bullwhip Effect." *Management Science*, 43(4), 546-558.
- Forrester, J.W. (1961). *Industrial Dynamics*. MIT Press.
- Model Context Protocol — [modelcontextprotocol.io](https://modelcontextprotocol.io/)

## License

MIT — [JIPRO-AI](https://github.com/JIPRO-AI)
