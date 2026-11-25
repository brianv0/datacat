# Dataset search: behaviour, evaluation and benchmarks

Reading notes, November 2026, in support of `docs/search.md` in the mudcat repository. Sources were read from their arXiv or publisher copies where reachable; the Kacprzak papers and the NTCIR overview were read only through search extracts.

## How people search for datasets

- **Kacprzak, Koesten, Ibáñez, Blount, Tennison, Simperl, "Characterising dataset search: an analysis of search logs and data requests", Journal of Web Semantics, 2019.** Query logs of four national open data portals plus a qualitative analysis of written data requests to one of them. Conclusion: portal search is used "in an exploratory manner, rather than to retrieve a specific resource". In data requests, "geospatial and temporal attributes, as well as information on the required granularity of the data are the most common features". An earlier ICWE 2017 version reports that portal queries differ from web queries in length and structure. A related WWW 2018 companion paper used crowdsourced queries and found them much longer than portal queries, with seven times more mentions of place and time, and more often phrased as questions.
- **Chapman, Simperl, Koesten, Konstantinidis, Ibáñez, Kacprzak, Groth, "Dataset search: a survey", VLDB Journal, 2020 (arXiv 1901.00735).** Dataset search "shows characteristics of exploratory search tasks, involving multiple queries, iterations and refinement of the original information need". It "relies on limited metadata instead of looking at the dataset itself", and contextual or personalised results "are practically non-existent". "There is no widely used classification of dataset information seeking tasks yet." On evaluation: "One of the most widely recognized problems of dataset search is the lack of benchmarks." Candidate online measures named: session abandonment rate, session success rate, zero result rate.
- **Goods (SIGMOD 2016), section 5.2 and 6.** Internal search is often for known things: re-finding one's own experimental datasets, finding all datasets that conform to a sensitive protocol buffer, finding data by the jobs that read or write it. Qualified tokens: `path:`, `proto:`, `read_by:`, `written_by:`, `downstream_of:`, `upstream_of:`, `kind:`, `owner:`. Paths are indexed as positional tokens so that `x/y` matches `a/x/y/b` but not `a/y/x/b`. Ranking signals: dataset type, match section, lineage fan-out (with the caveat that widely consumed datasets such as the web crawl get over-ranked), owner-written description. The authors state that ranking still needed significant improvement.
- **"User Experience in Dataset Search Platform Interfaces", 2024 (arXiv 2403.15861).** Think-aloud study on Google Dataset Search and data.europa.eu. Recommendations include a dataset preview panel and "simplified filters with a display of the remaining dataset count" to "prevent over-filtering". Participants valued auto-complete, result speed and relevance.
- **Russell and Dugall, "Discovering Datasets on the Web Scale: Challenges and Recommendations for Google Dataset Search", Harvard Data Science Review, 2024.** Abstract only: 45 million datasets from 13,000 sources; the open approach makes it harder for users to form a mental model of the tool and to make sense of heterogeneous datasets.

## Test collections and measures

| Collection | What it is | Measure and reference scores |
|---|---|---|
| ACORDAR 1.0 (SIGIR 2022) | 31,589 RDF datasets from 543 portals; queries and judgments by human experts | nDCG@10: TF-IDF about 0.55, BM25 about 0.59, FSDM about 0.62, LMD about 0.58 |
| ACORDAR 2.0 (SIGIR 2024) | Densely pooled, with question-style queries and TREC ad hoc topics; evaluates sparse and dense models | nDCG@10 |
| NTCIR-15 Data Search, NTCIR-16 Data Search 2 | Ad hoc retrieval over government statistical data (e-Stat, Data.gov) | Primary nDCG@10; also nERR@10 and Q-measure. One of the best NTCIR-16 runs: nDCG@10 of 0.153 |
| LakeBench (PVLDB 2024, Deng and others) | Table join and union search; over 16 million real tables; more than 10,000 queries | Precision@k and Recall@k; offline index time, online query time, memory |
| LakeBench (arXiv 2307.04217, Srinivas and others, IBM) | A different benchmark with the same name: unionability, joinability and subset tasks | F1 in later work that uses it |

Scores on different collections are not comparable, which is why mudcat's topic-search targets are relative to BM25 on the same collection.

LakeBench (PVLDB) findings relevant to cost: HNSW-based indexes are the fastest online but "not memory efficient", because the whole graph is loaded into memory; product quantisation is the suggested alternative. One method needed 15 days of offline indexing on the largest lake.

## Filtered vector search

**"Benchmarking Filtered Approximate Nearest Neighbor Search Algorithms on Transformer-based Embedding Vectors", 2025 (arXiv 2507.21989).**

- Selectivity is the fraction of items that pass the filter (some works use the inverse definition).
- Pre-filtering "is most efficient when selectivity is low, meaning only a few items remain". Post-filtering and in-filtering "are most efficient when selectivity is high". "If selectivity is too low, ANNS may not return enough valid candidates for post-filtering."
- Primary evaluation: recall@k against QPS.
- Eleven methods compared across filter types, k, scale and selectivity. Filtered-DiskANN and UNG do well at medium scale and fail on the largest dataset. "No universally best method exists."

A unified benchmark from the same year classifies methods as filter-then-search, search-then-filter and hybrid, and scores them on recall and QPS.

Consequence for mudcat: a permission filter is a filter of unknown, caller-dependent selectivity. A caller who can read a small part of the catalog is the low-selectivity case, where post-filtering returns short pages. The engine must choose its plan from the caller's readable share.

## What mudcat took from this

- Seventeen search use cases, split into ranked answers (index, can lag) and complete answers (catalog database, exact).
- Thirteen query classes, twelve of which have machine-derivable ground truth.
- Measures: Success@1, MRR@10, nDCG@10 relative to BM25, Recall@100, recall@10 against exact, exactness, facet count error, zero-result rate; latency percentiles; index lag; leaks, short pages and latency ratio by readable share; index size, rebuild time.
- Online measures from the survey: zero-result rate, abandonment, reformulation, position of the selected result, search-to-resolution rate.
- Not adopted for the first release: joinable and unionable table search, the main research line after Goods (JOSIE, table union search, SANTOS, Starmie, D3L, BLEND).

## Not verified

- The exact query-length statistics in the Kacprzak papers.
- The recommendations section of the Harvard Data Science Review article.
- Whether the licences of ACORDAR and the NTCIR collections allow use in a project test suite.
- The response-time limits used for the latency targets (about 0.1 s feels immediate, about 1 s keeps attention) are a common usability rule cited from memory.
