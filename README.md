# Flowise_Rag_based_BQBYTE_SUPPLIERS_NETWORK_QUERY_CHATBOT— Full Project Report
### Trinamix Hiring Task | Junior AI Engineer | Ref: TX-JrAI-003

---

## 1. Project Overview

This project builds a Retrieval-Augmented Generation (RAG) chatbot using Flowise Cloud that
answers natural language questions about BQBYTE Technologies' supplier network. The chatbot
combines a governance policy document (PDF) and supplier performance data (CSV) to provide
accurate, policy-cited answers to procurement queries.

**Public Chatbot URL:**
https://cloud.flowiseai.com/chatbot/6dd6ddda-b990-4635-845f-6593739719b3

**GitHub Repository:**
https://github.com/Ranveergit/Floweise_Rag-Based-BQBYTE-s-suppliers-Network-Query-Chatbot-

---

## 2. Data Sources

| File | Description |
|------|-------------|
| SupplyChain_Governance_Policy_v3.2.pdf | BQBYTE Technologies 10-section governance policy covering tier thresholds, SLAs, penalties, audit rules, and disruption procedures |
| supplier_performance_data.csv | 2,000 purchase orders across 116 suppliers with 27 columns including OTD rate, defect rate, compliance score, risk level, and PO value |
| supplier_summary.txt | Pre-computed supplier analysis document created to improve retrieval accuracy for aggregation queries |

---

## 3. System Architecture

```
User Query
    |
    v
Conversational Retrieval QA Chain
    |              |              |
    v              v              v
Qdrant Vector   Groq LLM    Buffer Memory
Store (RAG)     (LLaMA 3.3)  (Chat History)
    |
    v
HuggingFace Embeddings
(all-MiniLM-L6-v2)
    |
    v
Qdrant Cloud
Collection: scm-vectors
384 dimensions
```

### How RAG Works in This System
1. User sends a question
2. The question is converted to a 384-dimensional embedding vector using HuggingFace
3. Qdrant performs Approximate Nearest Neighbour (ANN) search across all indexed chunks
4. Top K most semantically similar chunks are retrieved
5. Retrieved chunks + chat history + user question are sent to LLaMA 3.3 70B
6. LLM generates an answer grounded in the retrieved context
7. Buffer Memory stores the exchange for follow-up questions

---

## 4. Technology Choices & Justification

### 4.1 LLM — LLaMA 3.3 70B via Groq

**What it is:** Meta's open-source large language model with 70 billion parameters, served via
Groq's Language Processing Unit (LPU) cloud infrastructure.

**Why LLaMA 3.3 70B:**
- 70B parameters provides strong multi-document reasoning and instruction following
- Outperforms smaller models (7B, 13B) on complex policy interpretation tasks
- Open-source — no vendor lock-in, full transparency

**Why Groq specifically:**
- Groq LPUs deliver 10x faster token generation than GPU-based inference
- Free tier is generous enough for development and testing
- Low latency critical for interactive chatbot experience

**Comparison with alternatives:**

| Model | Cost | Speed | Quality | Verdict |
|-------|------|-------|---------|---------|
| LLaMA 3.3 70B (Groq) | Free | Very Fast | High | ✅ Selected |
| GPT-3.5-turbo | Paid | Fast | Medium | ❌ Costs money, weaker reasoning |
| GPT-4o | Paid | Medium | Very High | ❌ Costs money, overkill for this task |
| Gemini 1.5 Flash | Free | Fast | Medium | ❌ Less accurate policy reasoning |
| LLaMA 3.1 7B | Free | Fast | Low | ❌ Too small for complex multi-doc RAG |

### 4.2 Embeddings — sentence-transformers/all-MiniLM-L6-v2

**What it is:** A lightweight sentence transformer model producing 384-dimensional dense vectors,
served via HuggingFace Inference API.

**Why this model:**
- Free via HuggingFace free tier
- 384 dimensions — small enough for fast ANN search, large enough for semantic accuracy
- Trained specifically for semantic similarity — ideal for document retrieval
- Works well for English procurement and policy text
- Consistent with Qdrant collection dimensions (no mismatch errors)

### 4.3 Vector Store — Qdrant Cloud

**What it is:** Open-source vector database with a free cloud tier.

**Why Qdrant:**
- Free cloud tier with no credit card required
- Supports 384-dimension collections (compatible with all-MiniLM-L6-v2)
- Fast ANN search via HNSW indexing
- Native Flowise integration
- Reliable — no dimension mismatch issues after proper setup

---

## 5. Document Store Setup

### Step 1 — Create Document Store
In Flowise Dashboard → Document Stores → New Store → Named: `knowledge_base`

### Step 2 — Load PDF
- Loader: PDF File loader
- File: SupplyChain_Governance_Policy_v3.2.pdf
- Result: 19 chunks

### Step 3 — Load CSV
- Loader: CSV File loader
- File: supplier_performance_data.csv
- Result: 2,000 chunks (one chunk per PO row)

### Step 4 — Load Summary
- Loader: Text File loader
- File: supplier_summary.txt (pre-computed aggregation results)
- Result: 12 chunks

### Step 5 — Configure Embeddings
- Model: HuggingFace sentence-transformers/all-MiniLM-L6-v2
- Endpoint: HuggingFace Inference API

### Step 6 — Configure Vector Store
- Provider: Qdrant Cloud
- Collection: scm-vectors
- Dimensions: 384

### Step 7 — Upsert
All documents embedded and stored in Qdrant. Total: ~2,031 chunks.

---

## 6. Chunk Configuration Experiments

### Config A (Final — Used for Deployment)
| Parameter | Value |
|-----------|-------|
| Chunk Size | 1000 characters |
| Chunk Overlap | 100 characters |
| PDF Chunks | 19 |
| CSV Chunks | 2,000 |

**Observation:** Larger chunks preserve more context per policy section. Better for answering
questions that reference multiple policy rules within the same section (e.g., Section 9 disruption
levels require reading multiple paragraphs together).

### Config B (Tested)
| Parameter | Value |
|-----------|-------|
| Chunk Size | 500 characters |
| Chunk Overlap | 50 characters |
| PDF Chunks | 35 |
| CSV Chunks | 2,000+ |

**Observation:** Smaller chunks split policy sections into fragments, reducing context quality for
complex policy questions. However, they improved retrieval precision for single-fact lookups.

**Decision:** Config A selected for final deployment.

---

## 7. Chatflow Architecture (Node Connections)

```
HuggingFace Inference Embeddings
        |
        | (Embeddings input)
        v
    Qdrant Node
    - Server URL: [Qdrant Cloud URL]
    - Collection: scm-vectors
    - Top K: 20
        |
        | (QdrantRetriever output)
        v
Conversational Retrieval QA Chain  <-- Groq (Chat Model)
        |                          <-- Buffer Memory (Memory)
        v
    Chat Output
```

**Key configuration:**
- Top K = 20 (retrieves 20 most relevant chunks per query)
- Buffer Memory = maintains last N conversation turns
- System Prompt = injects key policy thresholds for accurate rule application

---

## 8. Sample Q&A Results

### Q1: Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?

**Chatbot Answer:**
There are 11 Tier-3 suppliers with active disruption flags. All are classified as High Risk.
Per Policy Section 9 (Disruption Response Procedures), this triggers a Level 3 — Activate response:
- Immediate escalation to the Chief Procurement Officer (CPO)
- Alternate supplier activated for a minimum of 40% of volume within 10 business days
- Safety stock adjusted by +50%
- Full Root Cause Analysis (RCA) required within 15 business days

The 11 suppliers are: Dravex Components India, Plataforma Metales SA, Maghreb Castworks,
Helios Pack Greece, Cerromax Mineria, Orinoco Pack SAPI, Quetzal Textiles, Sibertek Molding,
Archipelago PCB Corp, Varna Electronics EAD, and Deltaforge Vietnam.

---

### Q2: Which suppliers qualify for the annual Volume Rebate Program and how many are there?

**Chatbot Answer:**
19 suppliers qualify for the annual Volume Rebate Program per Policy Section 4.2.
Criteria: Tier-1 status + OTD >= 93% + Defect Rate < 0.5% + Sustainability Score >= 85.

Qualifying suppliers: Borealis Composites, Crestline Chemical Supply, Fenwick Alloy Solutions,
Hanguk Circuit Works, Hokkaido Alloy Tech, Krauss-Polymex GmbH, Lakeshore Components,
Lumivex Semiconductor NL, Maplewood Polymer Corp, Norbec Alloy Works, Nordloom Finland Oy,
Orrentek Precision Mfg, Ostwind Composites AG, PrecisionForge Taiyuan, Solveig Eco Packaging,
Straits Packaging Hub, Tasman Circuit Boards, Toreval Electronics, and Valdoro Special Alloys.

Each qualifying supplier receives an annual rebate of 2.5% of total annual invoice value.

---

### Q3: Which region has the highest total PO value, and does it breach the concentration limit?

**Chatbot Answer:**
EMEA has the highest total PO value at $193,987,179.91, approximately 48.5% of total
procurement spend ($399,563,494.10).

This breaches the 45% regional concentration cap defined in Policy Section 5.3.
The Procurement team must submit a Diversification Plan within 60 days.

---

### Q4: Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?

**Chatbot Answer:**
11 suppliers are on Supplier Watch List (SWL) status. Per Policy Section 3.4, any supplier
with a Compliance Score below 60 is placed on SWL regardless of tier designation.

SWL suppliers: Deltaforge Vietnam, Maghreb Castworks, Helios Pack Greece, Cerromax Mineria,
Orinoco Pack SAPI, Varna Electronics EAD, Quetzal Textiles, Plataforma Metales SA,
Archipelago PCB Corp, Dravex Components India, and Sibertek Molding.

Restriction: New purchase order issuance is limited to 20% of prior quarter volume.

---

### Q5: Which product category has the highest average defect rate and does it exceed the Tier-2 limit?

**Chatbot Answer:**
Mechanical Components has the highest average defect rate at 2.12% across 360 purchase orders.

The Tier-2 maximum permissible defect rate is 2.50% per Policy Section 3.2.
At 2.12%, Mechanical Components is below the Tier-2 ceiling — no breach. However, the
category is approaching the limit and warrants close monitoring.

---

## 9. Challenges & Solutions

| Challenge | Root Cause | Solution Applied |
|-----------|-----------|-----------------|
| Pinecone AggregateError | Gemini embeddings dimension mismatch | Switched to HuggingFace all-MiniLM-L6-v2 |
| Qdrant "Forbidden" error | Expired API key | Created new API key with Read/Write permissions |
| Qdrant dimension mismatch | Old collection had wrong dims | Deleted collection, recreated with 384 dims |
| ChatGroq not connecting to QA Chain | Tool Agent loop | Switched to direct Conversational Retrieval QA Chain |
| RAG missing supplier names | Top K too low (4 chunks from 2000) | Increased Top K to 20 + added supplier_summary.txt |
| CSV aggregation inaccurate | Vector search samples data, can't do full-table scans | Added pre-computed summary document to document store |

---

## 10. Limitations & What I'd Improve

### Current Limitations
1. **RAG cannot aggregate across all 2000 rows** — vector similarity retrieves ~20 chunks (1% of data). Cannot reliably compute totals, counts, or averages across the full dataset.
2. **No metadata filtering** — PDF and CSV chunks are mixed in the same collection. Policy queries sometimes retrieve CSV rows and vice versa.
3. **Compliance score variation per PO** — the CSV stores per-PO metrics that vary per quarter. RAG retrieves individual PO rows, not supplier-level aggregates.

### Improvements
1. **CSV Agent / SQL Tool** — replace CSV RAG with a pandas execution agent for accurate full-dataset aggregation (ALL rows scanned, not sampled)
2. **Metadata filtering** — tag chunks with source type (pdf/csv) and filter by source during retrieval
3. **Supplier-level chunking** — pre-aggregate CSV to one chunk per supplier (all POs merged) so Top K=20 = 20 complete supplier profiles
4. **GPT-4o or LLaMA 3.1 405B** — more powerful LLM for complex multi-document cross-referencing
5. **Hybrid BM25 + vector search** — combine keyword search for exact supplier names with semantic search for policy concepts
*
