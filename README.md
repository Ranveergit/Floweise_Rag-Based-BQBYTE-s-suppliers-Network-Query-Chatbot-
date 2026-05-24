# Floweise_Rag-Based-BQBYTE-s-suppliers-Network-Query-Chatbot-
RAG chatbot built on Flowise Cloud that answers natural language questions about a supplier network using a governance policy PDF and 2,000-row performance CSV. Uses LLaMA 3.3 70B via Groq + HuggingFace embeddings + Qdrant vector store.


# SCM Assistant — Flowise RAG Chatbot
### Trinamix Hiring Task | Ref: TX-JrAI-003

## Public Chatbot URL
**https://cloud.flowiseai.com/chatbot/6dd6ddda-b990-4635-845f-6593739719b3**

---

## Chatflow Architecture

![Chatflow Canvas](screenshots/05-chatflow-canvas.png)

| Component | Choice |
|-----------|--------|
| LLM | ChatGroq — LLaMA 3.3 70B Versatile |
| Embeddings | HuggingFace — sentence-transformers/all-MiniLM-L6-v2 (384 dims) |
| Vector Store | Qdrant Cloud — collection: scm-vectors |
| Chain | Conversational Retrieval QA Chain |
| Memory | Buffer Memory |

---

## Why LLaMA 3.3 70B via Groq?

- **Free** — no billing required, Groq free tier sufficient for this task
- **Fast** — Groq LPU hardware delivers 10x faster inference than GPU-based APIs
- **Powerful** — 70B parameters gives strong policy reasoning vs smaller 7B/13B models
- **Better than GPT-3.5** — stronger instruction following and multi-document reasoning
- **Better than Gemini Flash** — more accurate when cross-referencing policy sections with CSV data

---

## Document Store

| File | Loader | Chunks |
|------|--------|--------|
| SupplyChain_Governance_Policy_v3.2.pdf | PDF Loader | 19 |
| supplier_performance_data.csv | CSV Loader | 2,000 |
| supplier_summary.txt | Text File Loader | 12 |

---

## Chunk Configurations Tested

| Config | Chunk Size | Overlap | Observation |
|--------|-----------|---------|-------------|
| A (Final) | 1000 | 100 | Better for policy questions — preserves full section context |
| B (Tested) | 500 | 50 | Better for single-fact lookups — smaller targeted chunks |

Config A selected for final deployment.

---

## Sample Q&A (Verbatim)

**Q1: Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?**

> There are 11 Tier-3 suppliers with active disruption flags — all classified as High Risk. Per Policy Section 9, this triggers **Level 3 — Activate**: immediate CPO escalation, alternate supplier activated at minimum 40% volume within 10 business days, safety stock +50%, and full RCA within 15 business days. Suppliers: Dravex Components India, Plataforma Metales SA, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Quetzal Textiles, Sibertek Molding, Archipelago PCB Corp, Varna Electronics EAD, Deltaforge Vietnam.

---

**Q2: Which suppliers qualify for the annual Volume Rebate Program and how many are there?**

> **19 suppliers** qualify per Policy Section 4.2 (Tier-1 + OTD ≥93% + Defect <0.5% + Sustainability ≥85): Borealis Composites, Crestline Chemical Supply, Fenwick Alloy Solutions, Hanguk Circuit Works, Hokkaido Alloy Tech, Krauss-Polymex GmbH, Lakeshore Components, Lumivex Semiconductor NL, Maplewood Polymer Corp, Norbec Alloy Works, Nordloom Finland Oy, Orrentek Precision Mfg, Ostwind Composites AG, PrecisionForge Taiyuan, Solveig Eco Packaging, Straits Packaging Hub, Tasman Circuit Boards, Toreval Electronics, Valdoro Special Alloys. Annual rebate: 2.5% of total invoice value.

---

**Q3: Which region has the highest total PO value, and does it breach the concentration limit?**

> **EMEA** at **$193,987,179.91** — approximately **48.5%** of total spend ($399,563,494.10). This **breaches the 45% regional concentration cap** (Policy §5.3). A Diversification Plan must be submitted within 60 days.

---

**Q4: Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?**

> **11 suppliers** (Compliance Score < 60 per Policy §3.4): Deltaforge Vietnam, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Varna Electronics EAD, Quetzal Textiles, Plataforma Metales SA, Archipelago PCB Corp, Dravex Components India, Sibertek Molding. **Restriction:** New PO issuance limited to 20% of prior quarter volume.

---

**Q5: Which product category has the highest average defect rate and does it exceed the Tier-2 limit?**

> **Mechanical Components** — average **2.12%** across 360 POs. Below the Tier-2 ceiling of **2.50%** (Policy §3.2) — no breach, but approaching the limit.

---

## What I'd Improve

- **CSV Agent / SQL tool** — vector search samples ~1% of 2000 rows; a pandas agent would enable accurate full-dataset aggregation
- **Metadata filtering** — tag chunks by source (PDF vs CSV) to prevent cross-contamination in retrieval
- **Supplier-level chunking** — group all POs per supplier into one chunk so each retrieved chunk = complete supplier profile
- **Larger LLM** — GPT-4o or LLaMA 3.1 405B for complex multi-step cross-document reasoning
- **Hybrid BM25 + vector search** — combine keyword and semantic search for exact supplier name retrieval

---

## Security

- No API keys committed to this repository
- `.gitignore` excludes: `.env`, `*.env`, `api_keys.txt`, `secrets/`
- All credentials stored in Flowise credential manager only

---

*Trinamix INC · Talent Acquisition · Ref: TX-JrAI-003*

