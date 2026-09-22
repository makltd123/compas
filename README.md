# Alqira — semantic search for Russian companies

**English** | [Русский](README.ru.md)

A B2B search engine that finds Russian companies by the meaning of a request instead of keywords. You write the way you would ask a colleague — "metal structure manufacturers in Yekaterinburg" or "IT outsourcing for banks, Kazan" — and get matching companies with contacts, a description and financial data.

Built in 2026 as a startup (first as Compas, then renamed Alqira). The product code and data are private; this repository describes how it works and how well it works.

## What it does

- Natural-language search over ~147,000 companies, with the city detected from the query or set explicitly.
- A company card: what the company does, products, clients, geography, contacts, and financials for about half of the companies.
- "Similar companies" for any result.
- Saved lists, search history and export to Excel.
- From a description of your product: generate an ideal customer profile (ICP) and a cold email.
- Sign-in by magic link, free and paid plans with limits.

## Data

| Source | Companies |
|---|---|
| 2GIS directory, 122 cities | ~85,600 |
| Orgpage.ru, 30 categories × 40 cities | ~14,300 |
| .ru domain zone: 5M domains scanned and filtered to business sites | ~42,200 |
| Earlier industry sets (IT, manufacturing, construction) | ~5,900 |
| **Total in the vector index** | **~146,800** |

Financial data from public Russian financial statements (FNS, 2023) is attached to ~76,500 companies.

## How it works

```
company directories, domain zone scan
        │
        ▼
liveness check of millions of URLs (200 parallel threads)
        │
        ▼
crawl sites: plain HTTP, contacts parsed from HTML, legacy Windows-1251 pages handled
        │
        ▼
LLM extraction (Mistral): site text → structured company profile (JSON)
        │
        ▼
embeddings with a fine-tuned BGE-M3 model (1024-d) → vector database (Zilliz)
        │
        ▼
financial enrichment: company coordinates → nearest FNS record → tax ID → SME registry → headcount
```

A search request is embedded by the same model, the city is taken from the request or a filter, and the nearest companies in the vector index are returned with their financials attached.

The embedding model was fine-tuned on company data for Russian B2B queries and is public: [`makltd1/compas-bge-m3-ru`](https://huggingface.co/makltd1/compas-bge-m3-ru).

## Search quality

The search was evaluated automatically, not by eye. Results of the last evaluation (June 2026):

**Coverage.** 1,000 queries = 40 industries × 25 cities, scored by the similarity of the best matches:

| Result | Queries |
|---|---|
| good (≥ 40%) | 691 |
| medium (30–39%) | 155 |
| gap (< 30%) | 154 |

**Relevance.** An LLM judge rated the top 30 results of each query from 1 to 10 — 63,600 query–company pairs in total:

- 14,718 pairs (23%) are relevant (≥ 7), 22,151 (35%) are not (≤ 3);
- among results the search was confident about (similarity ≥ 56%), 20.7% were still judged irrelevant;
- the judged relevance grows with the search score — from 4.0 on average below 50% to 6.1 above 60% — so the score is meaningful.

**Robustness** — how results change when the same intent is phrased differently:

| Query style | Score change | Overlap of top 5 with the standard phrasing |
|---|---|---|
| misspelled | −1.2% | 73% |
| as a question | −2.3% | 67% |
| as requirements | −3.3% | 62% |
| descriptive | −4.7% | 55% |
| English | −5.6% | 7% |
| short | −6.0% | 16% |

**What this says.** The main limit is coverage, not the model: some industries (food production, medical clinics, recruitment agencies) are thin in the data, and the search cannot find what is not there. Short and English queries change the results the most.

## Stack

Node.js + Express · Python + Flask embedding service · sentence-transformers (fine-tuned BGE-M3) · Zilliz Cloud · Turso (libSQL) · Mistral API · Docker · Kaggle/Colab GPUs for embedding and fine-tuning.

## Status

The live site is offline. The code is proprietary and kept in a private repository; this repository contains documentation only.
