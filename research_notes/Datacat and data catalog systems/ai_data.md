# AI data: cataloguing, versioning, storage and governance (state as of Nov 2026)

Method note: ~24 web searches on 2026-11-23. Page fetches (WebFetch) were blocked by the session network policy (mlcommons.org, gibsondunn.com unreachable), so findings rest on search-result summaries plus primary URLs. Items marked **[PK]** come from prior knowledge (cutoff mid-2026) with the canonical URL given but NOT re-verified in this session; treat versions/dates on [PK] items as needing a check. Vendor claims are flagged.

## 1. What counts as "AI data" and which catalog entities/relationships does it imply?

### Takeaway
AI data is not one asset type: it is a graph of immutable, versioned artifacts (corpora, splits, shards, samples, labels, embeddings, indexes, checkpoints, adapters, prompts, traces, eval results) joined by "derived-from / trained-on / evaluated-on" edges. Existing tools each model a slice: experiment trackers model run→model→metric; hubs model repo→revision→file; table formats model table→snapshot→fragment. None models the whole chain down to sample level.

### Cited Findings
- MLflow 3 made `LoggedModel` a first-class entity (no longer run-centric), linked to source run, code, config, params, metrics and datasets; traces and evaluation metrics can be grouped under one LoggedModel — [MLflow 3 docs](https://mlflow.org/docs/3.1.3/genai/mlflow-3), [MLflow releases](https://mlflow.org/releases/3)
- MLflow 3.17.0 (changelog dated 2026-10-06) associates aggregate logged-model evaluation metrics with the managed evaluation dataset that produced them, and fixed `EvaluationDataset` hashing — i.e. the dataset→eval→model link is still being tightened in late 2026 — [MLflow CHANGELOG](https://raw.githubusercontent.com/mlflow/mlflow/master/CHANGELOG.md), [3.17.0 announcement](https://groups.google.com/g/mlflow-users/c/LDktHclrHU4)
- Databricks Unity Catalog records model-version→upstream training/eval table lineage only when the model was trained on a table and logged with `mlflow.log_input` (MLflow ≥2.11) — lineage is opt-in and table-level — [Databricks docs](https://docs.databricks.com/aws/en/machine-learning/manage-model-lifecycle/)
- Apache Gravitino's model catalog is a managed (not federated) catalog with namespace catalog→schema→model→version; each version holds a URI plus optional aliases — [Gravitino model catalog](https://gravitino.apache.org/docs/next/model-catalog), [Manage model metadata](https://gravitino.apache.org/docs/next/manage-model-metadata-using-gravitino)
- Robotics data hierarchy in LeRobotDataset v3.0: dataset → chunk → file (Parquet for tabular signals, MP4 shards per camera) with episodes defined by metadata offsets (`meta/episodes/` chunked Parquet; `dataset_from_index`/`dataset_to_index`), not by filenames — many episodes per file — [LeRobot v3 docs](https://huggingface.co/docs/lerobot/lerobot-dataset-v3), [LeRobot paper](https://arxiv.org/pdf/2602.22818)
- Croissant models a dataset as metadata + `FileObject`/`FileSet` resources + `RecordSet`s with typed `Field`s (i.e. a logical record view over physical files), built on schema.org/Dataset — [Croissant paper](https://arxiv.org/pdf/2403.19546)
- ModelPack/KitOps package model weights, datasets, code, config and experiment metadata as one immutable OCI artifact — [KitOps repo](https://github.com/kitops-ml/kitops), [CNCF ModelPack](https://www.cncf.io/projects/modelpack/)
- [PK] MLflow `mlflow.data.Dataset` is a thin record (name, digest, source, schema, profile) logged to a run with a context tag (training/eval) — a pointer + hash, not managed data — [MLflow dataset tracking docs](https://mlflow.org/docs/latest/ml/dataset/)
- [PK] W&B Artifacts: typed, versioned (v0, v1…, aliases) directories with a content-hashed manifest; runs declare `use_artifact`/`log_artifact`, producing a run↔artifact bipartite lineage DAG; "reference artifacts" track external URIs by checksum without copying — [W&B Artifacts docs](https://docs.wandb.ai/guides/artifacts)

### Inferences
- Minimum first-class entity set for an "AI-ready" catalog: **Dataset** (logical), **DatasetVersion** (immutable snapshot/manifest), **Split/Subset/Config**, **View** (query/filter over a version, no copy), **Shard/File** (physical, replica-bearing), **Sample/Record** (addressable by stable ID or (file, offset/row)), **AnnotationSet** (labels versioned separately from raw media, with annotator/tool/guideline provenance), **EmbeddingSet** (must record producing model+version, dimension, metric, source column), **Index** (vector/FTS/scalar; derived, rebuildable), **MaterializedLoaderFormat** (tokenized/shuffled shards: record tokenizer, seed, shuffle buffer, source version), **FeatureView**, **Model/Checkpoint/Adapter** (base-model edge for LoRA/quantized/merged), **Prompt/PromptVersion**, **Trace/Trajectory/Episode**, **Run** (activity), **EvalSuite/Benchmark**, **EvalResult**.
- Core edges: DatasetVersion —contains→ Split —materialized_as→ Shard —contains→ Sample; DatasetVersion —derived_from(activity)→ DatasetVersion; Run —used(role=train|val|test|calibration|RAG)→ DatasetVersion/View; Run —generated→ Checkpoint; Model —fine_tuned_from/quantized_from/merged_from→ Model; EvalResult —of→ Model, —on→ DatasetVersion; Synthetic DatasetVersion —generated_by→ Model (+ seed data). The *role* on the "used" edge is essential (train vs eval is what contamination tracking needs). W3C PROV (entity/activity/agent) fits, and Croissant 1.1 has adopted PROV-O (see §4).
- Embeddings, indexes, tokenized shards and Parquet auto-conversions are *derived materializations* and should be modelled as such (rebuildable, tied to a source version), distinct from independent datasets.
- Scientific "file with versions and replicas across sites" maps cleanly onto the Shard/File layer; AI adds the logical layers above (record sets, views, samples) and the model/eval graph beside it.

### Gaps
- No authoritative, cross-tool ontology of AI data entities was found; the list above is a synthesis. Agent traces / RL trajectories as catalogued datasets are modelled only inside tracing tools (MLflow traces, W&B Weave) — no neutral standard found.
- Prompts as versioned catalog assets: MLflow has a prompt registry [PK, unverified this session]; no open standard found.

## 2. Existing systems: data model, strengths, limits, health

### Takeaway
The ecosystem splits into (a) hubs/registries (HF Hub, OCI/ModelPack, UC/Gravitino/Kubeflow), (b) experiment-lineage trackers (MLflow, W&B), (c) data-versioning layers (DVC, lakeFS, Icechunk), (d) multimodal table formats (Lance, Deep Lake, Iceberg/Delta), (e) loader formats/engines (WebDataset, MDS, LitData, Ray Data, Daft), and (f) curation/annotation tools. 2025–26 saw consolidation (lakeFS took over DVC; CoreWeave absorbed W&B; HF replaced Git LFS with Xet and added mutable buckets) but no convergence on one data model.

### Cited Findings
**Hugging Face Hub**
- Xet storage uses content-defined chunking (~64 KB target chunks, 8–128 KB bounds) for byte-level dedup; an edit only disturbs nearby chunk boundaries — [Hub Xet docs](https://huggingface.co/docs/hub/xet/deduplication), [Xet chunking](https://huggingface.co/docs/xet/chunking)
- Migration Git LFS→Xet moved "well over a dozen PBs" and ~500,000 repos — [HF blog](https://huggingface.co/blog/migrating-the-hub-to-xet); a secondary source gives 20 PB by July 2025 (unverified) — [mind-verse](https://mind-verse.de/news/neues-speichersystem-hugging-face-umstellung-git-lfs-xet)
- The Xet protocol is being written up as an IETF Internet-Draft (draft-denis-xet, -05 in June 2026; individual draft, not a standard) — [IETF datatracker](https://datatracker.ietf.org/doc/html/draft-denis-xet-04)
- Parquet content-defined chunking landed in PyArrow/Pandas (`use_content_defined_chunking`) so rewritten Parquet files dedupe on CAS backends (July 2025) — [HF blog](https://huggingface.co/blog/parquet-cdc)
- Storage Buckets (launched 2026-03-10): mutable, **non-versioned** object storage on Xet for working data, `hf buckets cp/sync/rm`; Common Crawl published its April 2026 crawl in a bucket — [HF changelog](https://huggingface.co/changelog/introducing-storage-buckets), [HF blog](https://huggingface.co/blog/storage-buckets), [Common Crawl](https://blog.commoncrawl.org/blog/april-2026-crawl-archive-now-available-in-a-hugging-face-storage-bucket)
- [PK] Data model: repo (model/dataset/space) = git history of files + README "card" with YAML metadata (license, task, configs/splits); dataset viewer auto-converts to Parquet on `refs/convert/parquet` and exposes rows/search/filter/statistics API and auto-generated Croissant JSON-LD — [dataset-viewer docs](https://huggingface.co/docs/dataset-viewer/parquet). Limits: lineage is free-text/tags (`base_model`, `datasets:`), single-site storage, no replica concept, coarse (repo-level) gating.

**MLflow 3 / W&B** — see §1. W&B was acquired by CoreWeave (closed 2025-05-05) and from 2026-09-30 SaaS accounts sign in through "CoreWeave Forge" — [CoreWeave](https://coreweave.com/blog/coreweave-completes-acquisition-of-weights-biases), [W&B page](https://www.coreweave.com/companies/weights-and-biases). Both treat datasets as pointers+digests; neither manages replicas or sample-level content.

**DVC / DataChain / lakeFS / Pachyderm**
- lakeFS acquired the DVC open-source project from Iterative; lakeFS now stewards DVC, positioned for "data scientists working with smaller datasets"; Iterative's team moved focus to multimodal AI data (DataChain, Inc.) — [lakeFS blog](https://lakefs.io/blog/lakefs-acquires-dvc/), [DVC FAQ](https://dvc.org/blog/dvc-joins-lakefs-your-questions-answered/)
- [PK] DVC: git-tracked `.dvc` pointer files (MD5) + content-addressed cache/remote; file-level dedup only; pipelines via `dvc.yaml` — [dvc.org](https://dvc.org/doc). DataChain: Python "dataset as versioned table of file references + Pydantic-typed signals", metadata in a DB, files stay in object storage — [DataChain repo](https://github.com/iterative/datachain). lakeFS: Git-like branches/commits/merges over an object-store namespace, zero-copy branching, S3-compatible gateway, hooks — [lakeFS docs](https://docs.lakefs.io/).
- Pachyderm: no 2026 status information surfaced in search (see Gaps).

**Lance / LanceDB**
- Lance file format 2.2 (blog March 2026): Blob V2, nested field evolution, native Map types, better compression; selected by `data_storage_version="2.2"`; file-format version is independent of library version (library at 6.x–7.x per docs.rs) — [LanceDB blog](https://lancedb.com/blog/lance-file-format-2-2-taming-complex-data), [benchmarks post (vendor)](https://lancedb.com/blog/lance-format-v2-2-benchmarks-half-the-storage-none-of-the-slowdown), [lance repo](https://github.com/lance-format/lance)
- Lance positions itself as file format + table format + namespace spec + index in one stack — [LanceDB blog](https://lancedb.com/blog/lance-format-v2-2-benchmarks-half-the-storage-none-of-the-slowdown); academic description of its random-access structural encodings — [arXiv 2504.15247](https://arxiv.org/html/2504.15247v1)
- Strength cited by practitioners: adding/backfilling feature columns without rewriting ("data evolution") vs Parquet/Iceberg/Delta — [PyData Seattle 2025](https://cfp.pydata.org/seattle2025/talk/BM8RSH); "multimodal lakehouse" where media + embeddings are core columns — [Gradient Flow](https://gradientflow.com/multimodal-lakehouse-lance/)

**Scientific tensors**
- Icechunk 1.0 (2025-07-10): transactional, versioned Zarr storage with virtual chunk references; Icechunk 2 announced 2026-04-09 — [Icechunk blog index](https://icechunk.io/en/latest/blog-posts/)
- VirtualiZarr 2.5.1 (2026-04-09) supports sharded Zarr v3; 2.6.0 requires Icechunk 2.x (~3x faster virtual-ref writes); 2.8.0 (2026-10-14) reimplements FITS and netCDF3 parsers without kerchunk — [VirtualiZarr releases](https://virtualizarr.readthedocs.io/en/latest/releases.html); existing Zarr v2/v3 can be virtually ingested into Icechunk without rewriting chunks — [Icechunk docs](https://icechunk.io/en/stable/guides/zarr/)

**Registries / packaging**
- ModelPack: CNCF Sandbox (accepted 2025-05-13 per project page), vendor-neutral OCI-based model packaging spec; KitOps implements both ModelKit and ModelPack; Docker Model Runner can emit ModelPack (`--format=cncf`, CNCF blog 2026-08-12) — [CNCF ModelPack](https://www.cncf.io/projects/modelpack/), [CNCF blog](https://www.cncf.io/blog/2026/08/12/advancing-ai-model-interoperability-with-docker-and-modelpack/), [KitOps](https://github.com/kitops-ml/kitops)
- [PK] Kubeflow Model Registry: RegisteredModel→ModelVersion→ModelArtifact REST API with a "model catalog" federation layer; datasets are not first-class — [Kubeflow docs](https://www.kubeflow.org/docs/components/model-registry/)

**Loader formats/engines**
- MDS (Mosaic StreamingDataset): binary shards (commonly 64 MB) + index, deterministic mid-epoch resumption; LitData: `optimize()` into chunks, streams from S3/GCS/Azure; WebDataset: POSIX tar shards, sequential, no fixed-cost random access — [zeroentropy overview](https://zeroentropy.dev/concepts/streaming-datasets/), [mosaicml/streaming](https://www.github.com/mosaicml/streaming), [LitData](https://pypi.org/project/litdata)
- Benchmarks disagree and are mostly vendor/personal (LitData vs MDS vs WebDataset; LitData vs Deep Lake on S3, 2026) — [benchmark repo](https://gh.r-mj.com/Nano1337/benchmark-dataloader), [2026 S3 benchmark](https://tanganke.github.io/blog/2026/benchmarking-dataset-loading-s3)
- Ray Data: streaming-batch execution across heterogeneous CPU/GPU pools reading from object storage (paper claims 31% training-time gain on a Stable Diffusion benchmark) — [arXiv 2501.12407](https://arxiv.org/html/2501.12407v5), [Anyscale blog (vendor)](https://anyscale.com/blog/architecting-multimodal-data-pipelines-that-scale-with-ray)

**Robotics**
- LeRobotDataset v3.0 (lerobot ≥0.4.0; stable 0.6.1 cited in docs) — see §1; conversion script from v2.1 migrates JSONL metadata to Parquet — [LeRobot docs](https://huggingface.co/docs/lerobot/lerobot-dataset-v3), [porting guide](https://huggingface.co/docs/lerobot/porting_datasets_v3)
- [PK] Open X-Embodiment: 1M+ trajectories, 22 embodiments, pooled from 60 datasets in RLDS (TFDS episode format) — [arXiv 2310.08864](https://arxiv.org/abs/2310.08864)

**[PK] Others (not re-verified; one-line models)**: Deep Lake (Activeloop) — tensor-columnar datasets with version control and a query language, chunked for object stores — [deeplake docs](https://docs.deeplake.ai/); Feast — entities/feature views/feature services with a registry and offline+online stores, point-in-time joins — [feast.dev](https://docs.feast.dev/); FiftyOne — sample-centric dataset with fields, labels, embeddings, saved *views* and "brain" similarity/uniqueness — [docs](https://docs.voxel51.com/); Argilla (HF-owned) / Label Studio (HumanSignal) — records + annotation tasks/responses per annotator, export to HF/JSON — [Argilla](https://docs.argilla.io/), [Label Studio](https://labelstud.io/guide/); TFDS — builder code + versioned `dataset_info` + TFRecord/ArrayRecord shards — [TFDS](https://www.tensorflow.org/datasets); NVIDIA DALI — GPU decode/augment pipeline, reads TFRecord/WebDataset/files, no catalog — [DALI](https://docs.nvidia.com/deeplearning/dali/); Daft — distributed dataframe with multimodal types, reads Lance/Iceberg/Delta/HF — [daft.ai](https://www.daft.ai/).

### Inferences
- HF's split into immutable versioned repos vs mutable non-versioned buckets is a design signal: publication and working-data need different semantics; a catalog should support both with an explicit "snapshot/publish" transition.
- Chunk-level CAS (Xet) and file-level CAS (DVC, W&B, OCI digests) are both in production; a catalog should store digests at file level as a minimum and allow optional chunk manifests for dedup-aware replication.
- Model registries (UC, Gravitino, Kubeflow, ModelPack) uniformly treat a model version as "URI + metadata + aliases"; none requires a typed link to the exact dataset version.
- Table-format catalogs cannot express "N episodes inside one MP4/Parquet shard" (LeRobot v3) or "virtual chunk references into archival netCDF/FITS" (Icechunk/VirtualiZarr); both are *indirection manifests* a file+replica catalog must understand to keep referenced bytes alive and co-located.

### Gaps
- Pachyderm (HPE) status in 2026: nothing found; believed low open-source activity since HPE acquisition (2023) — uncertain.
- Deep Lake version/health, Feast release cadence, Kubeflow Model Registry and OSS Unity Catalog model/volume features in 2026: searches returned nothing specific; [PK] descriptions only.
- Exact date of the lakeFS–DVC deal not confirmed (referred to in past tense by Feb 2026; believed Nov 2025).
- Lance Namespace spec: existence confirmed only via vendor description; no spec text retrieved. Crate source mentions a 2.3 format constant — status unknown.
- Hugging Face MCP server and dataset viewer details not re-verified in this session.

## 3. Formats and access patterns a catalog must understand

### Takeaway
A catalog must distinguish *source-of-truth* formats (Parquet, Lance, Zarr v3, HDF5, raw media) from *training materializations* (WebDataset tar, TFRecord, MDS, LitData chunks) and *model formats* (safetensors, GGUF, ONNX), and record for each asset its access pattern (random vs sequential), its container indirection (sample ↔ file/offset), and whether it is a virtual reference set over other files.

### Cited Findings
- Random access vs sequential: Lance targets efficient random access in columnar storage via adaptive structural encodings — [arXiv 2504.15247](https://arxiv.org/html/2504.15247v1); WebDataset tar offers no fixed-cost random access and suffers when shards are too small; MDS shards + index allow deterministic resume — [zeroentropy](https://zeroentropy.dev/concepts/streaming-datasets/)
- MDS and LitData both require a conversion/prepare pass → a second, derived physical copy of the dataset exists per loader format — [zeroentropy](https://zeroentropy.dev/concepts/streaming-datasets/), [LitData](https://pypi.org/project/litdata)
- Virtual datasets over archival files: VirtualiZarr creates Zarr stores of chunk references into netCDF/HDF5/FITS/Zarr without rewriting, committed transactionally in Icechunk — [VirtualiZarr](https://virtualizarr.readthedocs.io/), [Icechunk](https://icechunk.io/en/stable/guides/zarr/)
- Sample→container indirection: LeRobot v3 episode offsets inside shared Parquet/MP4 — [LeRobot docs](https://huggingface.co/docs/lerobot/lerobot-dataset-v3)
- Dedup: Xet CDC chunks (~64 KB) — [Xet docs](https://huggingface.co/docs/xet/chunking); Parquet CDC makes row-group/page boundaries content-defined so appended/edited Parquet dedupes — [HF blog](https://huggingface.co/blog/parquet-cdc)
- Dual packaging is normal for science: Multimodal Universe ships native HDF5 (needed for cross-matching utilities) at Flatiron plus Parquet on HF for streaming — [arXiv 2412.02527](https://arxiv.org/pdf/2412.02527); The Well ships self-documenting HDF5 under a shared spec plus HF streaming — [arXiv 2412.00568](https://arxiv.org/html/2412.00568v2)
- Disaggregated object storage → CPU preprocessing → GPU training without disk materialization is the reference large-scale pattern — [Anyscale (vendor)](https://anyscale.com/blog/architecting-multimodal-data-pipelines-that-scale-with-ray); LanceDB markets global shuffle + cross-cloud caching to GPUs — [LanceDB docs (vendor)](https://docs.lancedb.com/training/why-lancedb)
- [PK] Model formats: safetensors (JSON header + raw tensors, zero-copy, no code execution) — [repo](https://github.com/huggingface/safetensors); GGUF (single-file weights + KV metadata incl. tokenizer/quantization, llama.cpp) — [spec](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md); ONNX (protobuf graph + weights, opsets) — [onnx.ai](https://onnx.ai/). All carry extractable embedded metadata a catalog can harvest.

### Inferences
- Catalog should record per physical representation: format + format version, shard count/size, index presence, random-access capability, sample-count, and a `materialized_from` edge to the logical dataset version with the transform parameters (tokenizer, resize, seed).
- Sample addressing needs a stable logical ID *and* per-representation locators (file digest + row/offset/byte-range/tar member/Zarr chunk key), because the same sample lives in several materializations and at several sites.
- "Views" (filter/slice/join by metadata without copying) exist in FiftyOne views, DataChain, Lance/Iceberg snapshots and Zarr virtual refs, but with incompatible semantics; a neutral view object = (base version(s), predicate/selection or explicit sample-ID list, digest of resolved manifest).
- HPC/GPU-cluster locality is the bridge to scientific replica management: the catalog's replica table should include cache tiers (node-local NVMe, burst buffer, site object store) and loader-format copies as replicas-of-a-derivation, with pre-staging rules driven by scheduled training runs.

### Gaps
- No reliable, neutral 2026 benchmark comparing Parquet/Lance/WebDataset/MDS/LitData under identical conditions was found.
- No source found describing GPU-cluster cache/replica management integrated with a metadata catalog (Rucio-style rules for training shards); appears to be an open niche.

## 4. Standards and governance

### Takeaway
Croissant 1.1 (Feb 2026) is the closest thing to a universal AI-dataset metadata standard and now covers provenance (PROV-O) and usage policies; regulation (EU AI Act GPAI training-content summary, in force since Aug 2025) creates a concrete documentation obligation; integrity (OpenSSF Model Signing) and BOM standards (SPDX 3 AI/Dataset profiles, CycloneDX ML-BOM) exist but stop at file/artifact level and do not cover training-data provenance end-to-end.

### Cited Findings
- Croissant 1.1 (MLCommons, Feb 2026; co-authored with ODI): machine-actionable provenance adopting W3C PROV-O; external vocabulary linking at dataset/field/type level; structured usage policies for consent and licensing using standard policy vocabularies; multi-dimensional array fields, semantic types, example values, validation rules; "agent-ready" framing. MLCommons reports ~700K datasets with Croissant metadata; early adopters named: Common Crawl, Encord, HumanSignal — [MLCommons](https://mlcommons.org/2026/02/croissant-1-1-standard/), [ODI](https://theodi.org/news-and-events/blog/croissant-11/)
- Croissant-RAI extension covers data lifecycle, labeling, safety, fairness, traceability, compliance, inclusion — [arXiv 2407.16883](https://arxiv.org/html/2407.16883v1)
- Croissant metadata is still often missing/poor, motivating auto-generation tools: Croissant Baker (May 2026) and CroissantMiner (Oct 2026) — [arXiv 2605.15079](https://arxiv.org/pdf/2605.15079), [arXiv 2610.07132](https://arxiv.org/pdf/2610.07132)
- EU AI Act: GPAI rules apply since 2025-08-02; Commission published the mandatory template for the public summary of training content in July 2025 (overview of training data, main collections and sources; intended to help rightsholders exercise rights) — [IDPC summary](https://idpc.org.mt/?p=3493), [AI Act Service Desk](https://ai-act-service-desk.ec.europa.eu/en/resources)
- Digital Omnibus on AI: provisional political deal 2026-05-07 defers high-risk obligations (which include Article 10 data governance) to 2027-12-02 for Annex III systems and 2028-08-02 for Annex I products; Art. 50(2) marking of synthetic content postponed to 2026-12-02; extends the legal basis for processing special-category data for bias detection/correction beyond high-risk providers — [White & Case](https://www.whitecase.com/insight-alert/eu-agrees-digital-omnibus-deal-simplify-ai-rules), [Orrick](https://www.orrick.com/de-DE/Insights/2026/05/EUs-Digital-Omnibus-on-AI-7-Key-Changes-You-Need-to-Know), [FPF timeline](https://fpf.org/blog/the-ai-act-implementation-timeline-what-changes-under-the-ai-omnibus)
- [PK] AI Act text: Art. 10 requires documented data-governance for training/validation/test sets (origin, collection, preparation, bias examination, gaps); Art. 53(1)(c) copyright policy honouring TDM opt-outs (DSM Dir. Art. 4(3)); Art. 53(1)(d) public training-content summary; pre-Aug-2025 GPAI models have until 2027-08-02 — [Art. 10](https://artificialintelligenceact.eu/article/10/), [Art. 53](https://artificialintelligenceact.eu/article/53/)
- OpenSSF Model Signing (OMS): detached signature over a manifest of model files, Sigstore-bundle format, supports keyless Sigstore, bare keys or PKI; library/CLI v1.0 April 2025; roadmap to embed metadata such as training-data sources — [OMS spec](https://raw.githubusercontent.com/ossf/model-signing-spec/main/README.md), [OpenSSF blog](https://openssf.org/blog/2025/06/25/), [CoSAI paper](https://www.coalitionforsecureai.org/wp-content/uploads/2026/03/signing-ml-artifacts-1.pdf)
- A March 2026 IETF individual draft observes Sigstore signs model files at rest but does not cover training-data provenance, and CycloneDX/SPDX record lineage only as unverified metadata — [draft-sharif](https://datatracker.ietf.org/doc/html/draft-sharif-ai-model-lifecycle-attestation-00)
- AIBOM carriers: CycloneDX 1.7 ML-BOM and SPDX 3.0 AI profile (secondary source; HF has not made OMS signing default) — [safeguard.sh (secondary)](https://safeguard.sh/resources/blog/openssf-model-signing-v1-0-sigstore-ml-supply-chain)
- Benchmark contamination: canary strings (e.g. BIG-bench GUID) only work if trainers honour them and are trivially stripped; detection uses n-gram/MinHash overlap pre-training or Min-K% post hoc; dynamic benchmarks proposed — [arXiv 2502.17521](https://arxiv.org/pdf/2502.17521), [arXiv 2406.16746](https://arxiv.org/pdf/2406.16746); anecdotal evidence GPT-4-base memorised BIG-bench canary tasks — [Alignment Forum](https://alignmentforum.org/posts/kSmHMoaLKGcGgyWzs/big-bench-canary-contamination-in-gpt-4)
- [PK] Documentation frameworks: Datasheets for Datasets — [arXiv 1803.09010](https://arxiv.org/abs/1803.09010); Model Cards — [arXiv 1810.03993](https://arxiv.org/abs/1810.03993); Data Cards — [arXiv 2204.01075](https://arxiv.org/abs/2204.01075). All prose-first; Croissant/RAI is the machine-readable successor.
- [PK] Data Provenance Initiative audited 1,800+ text datasets and found high rates of missing/incorrect licence metadata on hosting platforms (>70% licence omission, ~50% error) — [arXiv 2310.16787](https://arxiv.org/abs/2310.16787); "Consent in Crisis" documented rapid growth of AI-crawler restrictions in robots.txt/ToS — [arXiv 2407.14933](https://arxiv.org/abs/2407.14933)
- [PK] Data & Trust Alliance Data Provenance Standards v1.0.0 (2024): 22 metadata fields across source, provenance (lineage, method, geography), use (consent, privacy, IP, intended/prohibited use); being standardised at OASIS — [D&TA](https://dataandtrustalliance.org/work/data-provenance-standards)
- [PK] C2PA Content Credentials: signed per-asset manifests; includes a training/data-mining permission assertion (maintained under CAWG in 2.x) — [C2PA specs](https://c2pa.org/specifications/specifications/2.2/index.html)
- [PK] SPDX 3.0 (Apr 2024) adds AI and Dataset profiles (fields e.g. datasetType, dataCollectionProcess, sensitivePersonalInformation, intendedUse, knownBias; relationships trainedOn/testedOn) — [SPDX 3.0.1](https://spdx.github.io/spdx-spec/v3.0.1/); CycloneDX ML-BOM since 1.5 (modelCard, datasets as `data` components) — [CycloneDX ML-BOM](https://cyclonedx.org/capabilities/mlbom/)
- [PK] Takedown precedent: LAION-5B withdrawn Dec 2023 after CSAM findings; Re-LAION-5B (Aug 2024) removed 2,236 links — models already trained on the original were unaffected and untracked — [LAION](https://laion.ai/blog/relaion-5b/)

### Inferences
- A catalog should natively import/export Croissant 1.1 (incl. RAI + PROV-O + usage policies) and emit SPDX 3 Dataset/AI and CycloneDX ML-BOM documents from its lineage graph; these are serialisations of the same graph (trainedOn/testedOn edges), so the internal model should be a superset.
- The GPAI summary template is effectively a *report over the catalog*: list of datasets used per model, sources (public/licensed/scraped/user/synthetic), crawl domains, opt-out handling. Producing it requires dataset-version→model edges plus per-dataset source-category, licence and collection-period fields.
- Licence/consent need to be machine-evaluable terms (ODRL-style policies as Croissant 1.1 suggests) attached at dataset *and* sample/source level, with propagation along derivation edges (most-restrictive-wins + flagged conflicts) — nothing found does this automatically.
- Takedown/deprecation requires: tombstones on samples/versions, forward traversal to every derived dataset, materialization, embedding index, checkpoint and eval; and a "tainted-by" status on models. LAION shows this is manual today.
- Integrity: store digests per file, a signed manifest per dataset version (OMS-style manifest signing generalises from models to datasets), and verify at replication/stage-in — which scientific replica systems already do with checksums.
- Eval-set contamination tracking is a lineage query (does any training-role ancestor of model M intersect eval set E at sample-hash/near-dup level?) — requires sample fingerprints (exact + MinHash) stored in the catalog.

### Gaps
- Whether the Digital Omnibus was formally adopted/published in the OJ (expected before 2026-08-02) could not be confirmed; dates above are from the May 2026 provisional deal. No source found saying the Omnibus changed the GPAI training-summary obligation.
- Croissant 1.1 details (exact policy vocabularies — ODRL/DUO?; MCP/agent features) could not be read from the primary page (host blocked).
- FAIR4ML schema version/status, D&TA→OASIS progress, C2PA spec version and SPDX/CycloneDX 2026 releases: not verified this session ([PK]).
- NeurIPS Datasets & Benchmarks 2025–26 Croissant requirement: believed mandatory since 2025, not verified.
- No standard found for machine-readable dataset deprecation/retraction notices propagating to models.

## 5. AI as a consumer of the catalog

### Takeaway
By 2026 an MCP server is table stakes for catalogs, and standards bodies frame metadata as "agent-ready"; "AI-ready data" guidance from science bodies exists but is fragmented by domain with no cross-agency standard.

### Cited Findings
- DataHub ships an MCP server (managed on DataHub Cloud, self-hostable for Core; PyPI `mcp-server-datahub`) — [DataHub docs](https://docs.datahub.com/docs/features/feature-guides/mcp), [PyPI](https://pypi.org/project/mcp-server-datahub/)
- OpenMetadata has a built-in MCP server and documents registering it in Databricks Unity Catalog so agents call it via the Unity AI Gateway (which also acts as a governed registry of MCP servers) — [OpenMetadata docs](https://docs.open-metadata.org/v2.0.x/how-to-guides/mcp/databricks)
- Google Knowledge Catalog (ex-Dataplex) exposes MCP servers for metadata and lineage — [Google Cloud docs](https://docs.cloud.google.com/dataplex/docs/mcp-overview?hl=en); claim that all major catalogs launched MCP integrations by 2026 (vendor/secondary) — [Atlan](https://atlan.com/know/mcp-connected-data-catalog/)
- Croissant 1.1 explicitly targets agent consumption (semantic types, example values, validation rules, machine-actionable policies) — [MLCommons](https://mlcommons.org/2026/02/croissant-1-1-standard/)
- LLM-generated metadata is an active research line (Croissant Baker; CroissantMiner with validation benchmark) — [arXiv 2605.15079](https://arxiv.org/pdf/2605.15079), [arXiv 2610.07132](https://arxiv.org/pdf/2610.07132)
- ESIP Data Readiness Cluster (NOAA, NASA, USGS participants) AI-readiness checklist v1.0 (last updated 2023-12-20), environmental-data focused — [ESIP](https://www.esipfed.org/collaboration-areas/data-readiness/), [GitHub](https://github.com/esipfed/data-readiness)
- NIH Bridge2AI Standards WG published AI-readiness criteria for biomedical data and an evaluation method — [PMC11526931](https://pmc.ncbi.nlm.nih.gov/articles/PMC11526931/)
- DOE: PNNL AI-Ready Data Pilot report (2026-02-27) proposes an evaluation framework of data requirements, metadata standards and validation tests — [PNNL](https://www.pnnl.gov/publications/ai-ready-data-pilot-project-report); JLab GlueX assessment argues data must be FAIR before AI-ready — [arXiv 2609.34987](https://arxiv.org/pdf/2609.34987)
- NSF workshop report: criteria for AI-ready data are not well understood; different degrees of readiness may be appropriate — [arXiv 2412.10278](https://arxiv.org/pdf/2412.10278)

### Inferences
- Catalog API should be designed MCP-first: search (hybrid keyword+vector over metadata), describe schema with examples/statistics, resolve access (signed URLs/replica selection), lineage traversal, and policy check ("may I train on this?") as tools — with the agent's identity and purpose passed for governance.
- An "AI-readiness score/profile" per dataset (documentation completeness, licence clarity, format accessibility, quality stats, splits defined, checksums) is implementable as computed catalog facets mapped to ESIP/Bridge2AI/DOE criteria.
- LLM-generated documentation must be stored with provenance (generated-by model, human-verified flag) to avoid polluting authoritative metadata.

### Gaps
- Hugging Face MCP server specifics, and any NASA/ESA formal 2026 "AI-ready" criteria: not found/verified.
- No evidence found on measured quality of agent-driven data discovery over catalogs (independent evaluations).

## 6. AI-for-science specifics

### Takeaway
Scientific foundation-model datasets are built by small teams who repackage heterogeneous archives into a uniform spec, then distribute twice: native HDF5/Zarr at an HPC/institutional site and Parquet/streamable copies on Hugging Face. Cross-site access, heterogeneous formats, and cross-matching across sources are the recurring pain points.

### Cited Findings
- Multimodal Universe: ~100 TB, 120M+ galaxy images, 5M+ spectra, plus light curves etc. from DESI, SDSS and others; native HDF5 hosted at Flatiron (full local download needed for cross-matching utilities), mirrored at NERSC; Parquet copies as per-survey HF datasets for streaming; versioned collection (v1.5) — [arXiv 2412.02527](https://arxiv.org/pdf/2412.02527), [NERSC portal](https://portal.nersc.gov/cfs/cosmo/data/mmu/), [HF org](https://huggingface.co/MultimodalUniverse/collections)
- The Well (NeurIPS 2024 D&B): 15 TB, 16 physics-simulation datasets (6.9 GB–5.1 TB each), uniform grids, self-documenting HDF5 under a unified spec with format-checking scripts, PyTorch interface, `the-well` on PyPI, streamable from HF — [arXiv 2412.00568](https://arxiv.org/html/2412.00568v2), [NeurIPS](https://proceedings.neurips.cc/paper_files/paper/2024/hash/4f9a5acd91ac76569f2fe291b1f4772b-Abstract.html)
- Climate/EO: virtualisation of archival netCDF/HDF5/FITS into Zarr via VirtualiZarr + Icechunk is the 2025–26 direction (FITS parser shows astronomy uptake) — [VirtualiZarr releases](https://virtualizarr.readthedocs.io/en/latest/releases.html)
- HEP: GlueX (JLab) FAIR/AI-readiness assessment (2026) — [arXiv 2609.34987](https://arxiv.org/pdf/2609.34987)
- [PK] ARCO-ERA5: Google's analysis-ready cloud-optimised ERA5 as Zarr on GCS, basis for WeatherBench 2 / GraphCast-class models — [repo](https://github.com/google-research/arco-era5)
- [PK] OMat24 (Meta FAIR): 110M+ DFT calculations distributed on HF as ASE-LMDB with CC-BY-4.0; Open Catalyst OC20/OC22 as LMDB — [arXiv 2410.12771](https://arxiv.org/abs/2410.12771), [HF dataset](https://huggingface.co/datasets/fairchem/OMAT24)
- [PK] AlphaFold DB: 200M+ predicted structures, CC-BY-4.0, bulk on GCS + EBI FTP; PDB in mmCIF via wwPDB — [AlphaFold DB](https://alphafold.ebi.ac.uk/download); CERN Open Data portal distributes ROOT/NanoAOD with DOIs — [opendata.cern.ch](https://opendata.cern.ch/)

### Inferences
- Each project invented its own "uniform spec + loader + checker" (MMU HDF5 convention, The Well HDF5 spec, LMDB for materials, RLDS/LeRobot for robots). A catalog that stores a declarative schema (Croissant RecordSets incl. n-D arrays in 1.1) and validates against it would replace bespoke checkers.
- Dual-site, dual-format distribution = replicas of *different representations* of the same logical dataset version; the catalog must model representation equivalence (HDF5@Flatiron ≡ Parquet@HF for version v1.5) distinct from byte-identical replicas.
- Cross-matching (object IDs/sky coordinates across surveys) is a sample-level join across datasets — argues for sample-level identifiers and cross-dataset entity keys in the catalog.
- Upstream licences/provenance are per-survey/per-source; aggregated datasets need per-subset licence and citation metadata that propagates to the aggregate.

### Gaps
- Explicit builder-reported pain points (quotes on scale/licence/cross-site issues) could not be extracted because paper full-text fetch was blocked; the pain points above are inferred from packaging choices.
- Genomics foundation-model data packaging: not researched (budget).

## 7. Gap analysis: what no current system does well

### Takeaway
The unmet need is a single graph that spans physical files/replicas, logical datasets/views/samples, and models/evals, with policy and integrity propagating along edges. Every existing tool covers at most two of those layers.

### Cited Findings
- Sample-level lineage: a targeted search for engineering write-ups found none describing sample-level lineage at scale; lineage in UC/MLflow is table/dataset-level and opt-in — [Databricks docs](https://docs.databricks.com/aws/en/machine-learning/manage-model-lifecycle/); "data portraits" are proposed as an after-the-fact membership-test artifact precisely because training-set membership is otherwise unrecorded — [arXiv 2406.16746](https://arxiv.org/pdf/2406.16746)
- Dataset→eval→model linkage is still being patched within a single tool in late 2026 (MLflow 3.17) — [MLflow CHANGELOG](https://raw.githubusercontent.com/mlflow/mlflow/master/CHANGELOG.md)
- Integrity/provenance stop at the artifact: signing covers model files at rest, not training-data provenance; BOMs carry unverified lineage metadata — [IETF draft](https://datatracker.ietf.org/doc/html/draft-sharif-ai-model-lifecycle-attestation-00); HF has not adopted OMS by default (secondary) — [safeguard.sh](https://safeguard.sh/resources/blog/openssf-model-signing-v1-0-sigstore-ml-supply-chain)
- Contamination control relies on voluntary canaries and post-hoc statistical tests — [arXiv 2502.17521](https://arxiv.org/pdf/2502.17521)
- Metadata completeness remains poor enough that tools to auto-generate/validate Croissant are 2026 research topics — [arXiv 2610.07132](https://arxiv.org/pdf/2610.07132); licence metadata on hubs is frequently missing/wrong [PK] — [arXiv 2310.16787](https://arxiv.org/abs/2310.16787)
- HF's largest hub has versioned repos and unversioned buckets but single-provider storage (no multi-site replica semantics) — [HF buckets docs](https://huggingface.co/docs/hub/en/storage-buckets); scientific projects therefore hand-maintain mirrors (Flatiron/NERSC/HF) — [arXiv 2412.02527](https://arxiv.org/pdf/2412.02527)
- Criteria for "AI-ready" are unsettled and domain-fragmented — [arXiv 2412.10278](https://arxiv.org/pdf/2412.10278)

### Inferences (gap list for the designer)
1. **Unified file + table + tensor + model catalog with replica/site awareness** — Rucio/iRODS know replicas but not record sets/models; HF/Lance/UC know datasets/models but not sites. No system found does both.
2. **Sample-level identity and lineage at 10^9–10^10 samples** — needs compact representations (manifest digests, row-ID ranges, Bloom/MinHash sketches, lineage stored as transform + parent-manifest rather than per-sample edges, materialised on demand).
3. **Representation equivalence** — same logical version as HDF5, Parquet, WebDataset, tokenized MDS at different sites; no tool tracks these as one asset with derivation parameters.
4. **Licence/consent/opt-out propagation** through derivations to models, with machine-evaluable policies and GPAI-summary/AIBOM report generation.
5. **Takedown/deprecation propagation** — tombstones flowing to derived datasets, indexes, checkpoints, evals; nothing found automates it.
6. **Eval-set contamination tracking** — role-typed usage edges + sample fingerprints enabling "was any ancestor of this model trained on this benchmark?" queries.
7. **Cross-tool lineage stitching** — dataset version (HF/lakeFS/Icechunk/Lance snapshot IDs) ↔ run (MLflow/W&B) ↔ model (registry/OCI digest) ↔ eval; today each tool keeps its own IDs. A catalog should accept foreign identifiers/digests as first-class aliases and ingest OpenLineage/MLflow/W&B events.
8. **Derived-asset lifecycle** — embeddings/indexes/loader shards as rebuildable, cache-like replicas with GC and pre-staging policies tied to training schedules on GPU/HPC clusters.
9. **Mutable-working vs immutable-published** semantics with signed snapshot manifests (HF buckets vs repos shows the need).
10. **Agent-facing governance** — MCP tools that enforce policy and record agent access as provenance.

### Gaps
- Evidence for gaps 1–3 and 8 is largely *absence of evidence* from searches plus tool documentation; internal lab systems (OpenAI, Google, Meta, Anthropic) may solve some of these but no public write-ups were found.
- Could not verify whether DataChain, Lance namespace, or lakeFS 2026 releases have added sample-level lineage or multi-site replication; check their current docs before claiming novelty.
