# AI-for-science organisations: data shapes, practice, pain and catalog requirements

Research date: 2026-11-23. Scope: public evidence about how AI-for-science startups, institutes and labs manage data, to inform an open-source multimodal data catalog (dataset = table of samples whose cells are scalars or pointers into files; immutable file versions with checksummed replicas; scientific formats referenced in place; PROV-style provenance; licence/consent propagation; Postgres core).

Not re-described here (covered in sibling notes): Hugging Face Hub, Lance, MLflow, DVC/DataChain, lakeFS, Croissant, Rucio, iRODS, Rubin Butler, MetaCat, SciCat, NOMAD, Zarr/Icechunk/VirtualiZarr in general.

## 0. Evidence labels and access limits

Every claim carries one of these labels.

- **[V]** verified in full this session: the primary file was fetched and read (only `raw.githubusercontent.com` was reachable).
- **[S]** search-snippet only: the text comes from a web-search extract; the page itself could not be opened. Numbers and quotes are as the extract gave them and may be truncated or paraphrased.
- **[P]** prior knowledge (model training, cutoff mid-2026), not re-checked this session.
- **[I]** my inference from the evidence.

Hosts that failed to resolve from this session (so nothing on them was read directly): arxiv.org, arcinstitute.org, technologyreview.com, bvp.com, job-boards.greenhouse.io, jobs.generalcatalyst.com. All job postings, papers, vendor pages and news items are therefore [S] unless stated. Job-board aggregators (Built In, Ladders, Teal, etc.) sometimes paraphrase postings; quoted phrases below are as the search extract returned them.

Things looked for and not found: any startup engineering postmortem that literally says "we could not tell which assay version trained which model"; published headcount of data engineers at named startups; Benchling registry limitations as described by ML teams; current (2026) AnIML/SiLA adoption statistics; the Billion Cells Project's release status; content of the DOE Genesis Mission data RFI.

---

## 1. Data shapes by domain, flagship sizes, formats, channels, reported pain

### 1.1 Summary table

| Dataset | Sample unit | Scale | Format and channel | Licence | Label |
|---|---|---|---|---|---|
| Tahoe-100M (Tahoe/Vevo, in Arc Virtual Cell Atlas) | cell = (plate, cell line, drug, concentration) with sparse count vector | 100,648,790 cells, 1,344 "samples" (unique treatments); 316 GiB on GCS | count matrices h5ad (`.h5ad.gz`, internal gzip), metadata Parquet; public GCS bucket `gs://arc-institute-virtual-cell-atlas/tahoe100M/`; separate Parquet copy on HF Hub for streaming | (Arc atlas CC0 per search) | [V] sizes and format; licence [S] |
| scBaseCount (Arc) | cell from an SRA run, re-quantified with STARsolo under several feature types | >502 M cells, 27 organisms, 75 tissues; 39.5 TiB | h5ad per SRA experiment + Parquet metadata; releases are dated prefixes (`scbasecount/2026-01-12/...`); earlier bucket deprecated 2026-03-31 | CC0 [S] | [V] |
| Virtual Cell Challenge 2025 data (Arc) | cell × CRISPRi perturbation in H1 | ~300k cells, 300 perturbations; 5.3 TiB of which 5.28 TiB is raw FASTQ | h5ad + FASTQ on same bucket | | [V] sizes; counts [S] |
| X-Atlas/Orion (Xaira) | cell × genome-wide CRISPR perturbation | 8 M cells, 2 cell lines; processed ~521 GB on Figshare+; successor X-Atlas/Pisces 25.6 M cells, 7 contexts, only a subset released (Mar 2026) | Figshare DOI, non-commercial licence | non-commercial | [S] |
| CZ CELLxGENE Census | cell (obs) × gene (var) with `is_primary_data` flag for duplicates | Oct 2024: 1,550+ datasets, 169.3 M cells, 93.6 M unique; LTS 2025-11-08 reported as 217 M+ total / 125 M+ unique (third-party figure) | TileDB-SOMA on S3, API in Python/R; LTS releases | CC BY | 2024 figures [S] (NAR paper); 2025 figures [S], unverified third party |
| RxRx3 (Recursion) | well = (experiment, plate, address) with 1 site × 6 channels; perturbation = CRISPR guide or compound×8 doses, HUVEC | 2.2 M wells [S]; images ~83 TB, 2048×2048 16-bit PNG, one tar per experiment-plate; embeddings 1.82 GB Parquet (128-d per well) | own download site; `metadata.csv` keyed by `well_id`; 16,328 of 17,063 genes anonymised "protecting Recursion's business interests"; "less than 1% of Recursion's total dataset" | Recursion non-commercial EULA | [V] |
| RxRx3-core | same, curated | 222,601 wells, 1,335,606 single-channel JP2 files, 18 GB (cut from >10 TB by 512×512 centre crop and uint16→uint8) | HF Hub | | [S] |
| JUMP Cell Painting (cpg0016) | well → sites → 5 channels; ~116k compounds, ~15–22k genetic perturbations, 12 (or 13) data-generating sites | est. 115 TB, 1.6 B cells; whole Cell Painting Gallery 656 TB (Jan 2024) | AWS Open Data S3; per-batch positive-control plate ("Target2") | CC0 [P] | [S] |
| OMol25 (Meta FAIR + LBNL et al.) | single-point DFT of a molecular configuration: atoms, charge, spin → energy, forces | >100 M calculations, ~83 M unique systems, up to 350 atoms; ~6 billion CPU core-hours; raw electronic-structure outputs "O(petabytes)" (LBNL slide: ~6 PB of wavefunctions), stored at Argonne (Eagle), access by approval; only a 4 M subset of raw ORCA outputs released first | ASE-DB-compatible LMDB (`*.aselmdb`), charge and spin in `atoms.info`; gated on HF Hub | CC BY 4.0 | format, licence, counts [V]; PB figures [S] |
| OMat24 (Meta FAIR) | bulk inorganic structure → energy, forces, stress; sub-datasets by generation method (rattled-T, AIMD NVT/NPT, rattled-relax) | paper: 110–118 M calculations, 400 M+ core-hours [S]; current docs card lists 1.07 M train + 1.02 M val structures and 185.8 GB train tarballs [V, and inconsistent with the paper; I did not resolve which subset the docs card describes] | tar.gz of aselmdb on `dl.fbaipublicfiles.com`, HF Hub | CC BY 4.0 | mixed |
| AlphaFold DB | UniProt accession → predicted structure + confidence | v4: 214 M entries; "23 TiB, 3 × 214M files", 1,015,797 sharded proteome tars, ~644 M files after untar; v6: 241 M entries [S] | GCS public bucket + BigQuery metadata; EBI FTP for 48 proteomes; file name = `AF-<UniProt>-F1-model_v4` | CC BY 4.0 | v4 [V], v6 [S] |
| ESM3 training set (EvolutionaryScale) | protein with sequence/structure/function tracks | 2.78 B proteins, 771 B tokens; 236 M structures, 539 M with function annotations | proprietary assembly of public sources | | [S] |
| ERA5 / ARCO-ERA5 | hourly global field: time × level × lat × lon per variable | full ERA5 ~5 PB; one AR store `full_37-1h-0p25deg-chunk-1.zarr-v3` is 2.05 PB, chunk = one time step (721×1440×37), ~154 MB per variable chunk | Zarr (v3) in a public GCS bucket in three tiers `raw/` (GRIB, netCDF), `co/` cloud-optimised, `ar/` analysis-ready; stable data appended monthly with ~3-month lag, ERA5T with ~1 week lag; store attributes `valid_time_start/stop`, `last_updated` | Copernicus | [V] |
| Anemoi training-ready ERA5 (ECMWF) | same, regridded O96, 6-hourly, 1979–2023 | ~0.5 TB, 65,000+ files | Zarr built from a YAML recipe (`anemoi-datasets create recipe.yaml out.zarr`) | CC BY 4.0 [S] | [S] |
| NNJA-AI (Brightband + NOAA) | observation record per sensor/source, daily partitions | ~50 TiB (v1, Aug 2025) | BUFR → Avro → Parquet on GCS, with an SDK "data catalog" | CC BY 4.0 | [V] size/format; pipeline detail [S] |
| Clay v1 training set | image chip referencing a STAC item | 33.8 TB, ~70 M chips; embeddings as GeoParquet on Source Cooperative | `stacchip` "keeps data in its original format and indexes chips" | | [S] |
| The Well (Polymathic) | simulation trajectory on a uniform grid at constant time step, per dataset | 15 TB, 16 datasets, 6.9 GB–5.1 TB each | "self-documenting HDF5", train/valid/test splits, own downloader, most also on HF Hub | per dataset | [V] sizes; HDF5 detail [S] |
| Multimodal Universe (Polymathic et al.) | astronomical object with images, spectra, time series, tabular, keyed for cross-match | ~100 TB (v1 98.34 TB), hundreds of millions of observations | native HDF5 at Flatiron via HTTPS/Globus ("GLOBUS is much preferable"); Parquet mirror on HF for streaming; cross-matching utilities require local HDF5; per-survey download/format scripts contributed by PR; per-subset licences and citations | CC BY 4.0 plus per-survey terms | [V] |
| Aspen Open Jets | jet from CMS 2016 JetHT open data | 178 M jets; CMS Open Data overall >4 PB | derived from MiniAOD/NanoAOD (ROOT) | CC0 | [S] |
| GNoME | crystal structure + formation energy | 2.2 M candidate structures, 381k new stable | | | [S] |
| LeMat-Bulk | material/calculation harmonised across MP, Alexandria, OQMD | 5.3 M materials, 3 functionals; `entalpic_fingerprint` hash for dedup | HF Hub | | [S] |
| MatPES | single-point PBE and r2SCAN of 504,811 structures | | | | [S] |
| Recursion (private) | well/image, transcriptome, in-vivo, ADME, patient record | company pages give 23 → 36 → 50 → 65 PB at different times; "up to 2.2 million experiments every week" | GCP; BioHive-2 on-prem training, inference on cloud | proprietary | PB figures [S]; weekly rate [V] |
| insitro (private) | | "20+ petabytes" | | | [S] |
| Noetik (private) | spatially resolved cell with paired H&E, protein, transcript | >100 M cells, >5,000 patients | | | [S] |

### 1.2 Structural observations (what the shapes have in common)

1. **The sample is a join over an experimental design, not a file.** [I] from the schemas above. RxRx3's `well_id` is `experiment_plate_address`; six channel files make one image; one metadata row carries gene, guide, SMILES, concentration, perturbation type and cell type [V]. Tahoe's obs table carries plate, sample (treatment), drug+concentration, Cellosaurus cell-line ID, sublibrary and a `pass_filter` QC level [V]. A catalog row has to address *n* files (channels) and *m* design factors per sample, and controls (EMPTY_control, CRISPR_control) are first-class rows.
2. **One logical dataset is distributed in two or three physical encodings at once.** Tahoe: h5ad on GCS and Parquet on HF [V/S]. Multimodal Universe: HDF5 at Flatiron and Parquet on HF [V]. ARCO-ERA5: raw GRIB/netCDF, cloud-optimised Zarr and analysis-ready Zarr side by side [V]. WeatherNext: BigQuery, Earth Engine and raw Zarr on GCS [S]. A catalog must model "same content, different encoding/chunking" as related representations, with lineage between them, not as unrelated datasets.
3. **Raw-to-derived size ratios are extreme, and the raw tier often never leaves an HPC site.** OMol25 trains on LMDB records while ~6 PB of wavefunctions sit at Argonne behind an approval process [S]. RxRx3-core is 18 GB cut from >10 TB [S]. The Virtual Cell Challenge prefix is 99.6% raw FASTQ [V]. Anemoi's ERA5 is 0.5 TB out of ~5 PB [S].
4. **Datasets are released as dated, superseding snapshots, and the builders deprecate old ones.** scBaseCount `2026-01-12` "is a revamp and expansion of the initial release… The initial release can generally be considered deprecated" and the old bucket was retired [V]. PLINDER is versioned by RCSB sync month (`2024-04/v1`, `2024-06/v2`, current `2026-09`) with split changes between versions [V]. Materials Project "has a practice of re-running calculations with updated methodologies", so LeMat publishes a method rather than a static set [S]. CELLxGENE Census has LTS releases [S].
5. **File-count explosions are a recurring operational problem.** AFDB warns of ~644 M files after untar and ships 1,015,797 tar shards "because… significantly faster", plus manifests [V]. RxRx3-core is 1.3 M JP2 files [S]. Arc ships 460k files [S]. [I]: per-file catalog rows are needed, but so are container entries (tar shard member, HDF5 dataset, Zarr chunk) addressed by byte range.
6. **Business-driven redaction is part of public datasets.** RxRx3 anonymises 96% of gene labels and may "de-anonymize genes in this dataset in the future" [V]. Xaira releases only a subset of Pisces [S]. A catalog needs column/row-level visibility tiers and the ability to re-release the same samples with more labels unblinded later without changing sample identity.
7. **Duplicates across sources are expected and flagged, not removed.** Census: 169.3 M cells of which 93.6 M unique, with `is_primary_data` [S]. LeMat-Bulk fingerprints structures to dedup MP/Alexandria/OQMD [S].

### 1.3 Pain the builders reported

- **Archive formats are hostile to ML.** Brightband on NNJA: "the archival format in which the data is originally available (BUFR) is cumbersome to work with" [V]. Cell Painting Gallery authors: "proprietary image file formats are a major impediment to the reuse of both images and workflows" [S]. TokaMark authors on fusion: datasets "remain fragmented across institutions, locked behind proprietary interfaces, or stored in domain-specific formats" [S].
- **Chunking decides usability.** ECMWF ARCO documentation: chunking largely determines download performance because partial chunks cannot be fetched [S]. ARCO-ERA5 keeps separately chunked copies [V].
- **Egress and hosting cost.** Arc tells users to scope transfers and to "open an issue with the billing line items" if they see download charges [V]. Multimodal Universe steers bulk users to Globus [V].
- **Loading at scale.** Tahoe-100M is ~300 GB as sparse compressed AnnData and needed dedicated loaders (scDataset paper) [S]; Census loaders moved to TileDB-SOMA-ML [S].
- **Batch effects.** JUMP: variation from "lamp intensity, staining concentration, and cell seeding or growth rate" even within one lab; a positive-control plate in every batch; no systematic comparison of correction methods existed before [S].
- **Incompatible "instrument settings" in computed data.** MPtrj mixes PBE and PBE+U, "two incompatible potential-energy surfaces"; U applied only for some metals with O/F present; forces and stresses "remain unadjusted" [S]. MatPES and MP-ALOE were created to fix this [S].
- **Heterogeneous terms.** Multimodal Universe: CC BY overall, "however when using specific subsets, the license and conditions of utilisation should be respected", and it ships a tool that emits per-dataset citation and acknowledgement text [V].
- **Automated curation at scale.** scBaseCount uses "an AI-driven, hierarchical agent workflow" to discover SRA data and extract metadata [V]. [I]: curated metadata therefore needs a provenance marker distinguishing agent-extracted from submitter-asserted values.

---

## 2. Lab-in-the-loop / closed-loop cycle

### 2.1 What the loop looks like in public descriptions

- Genentech/Prescient "lab-in-the-loop" (Frey et al., bioRxiv Feb 2025): generative models, multi-task property predictors, active-learning ranking and in-vitro testing; >1,800 unique antibody variants across four targets, with each round's assay data ingested for the next [S].
- Recursion: "up to 2.2 million experiments every week" in automated labs [V]; a "Industrialized Workflow Platform" on Prefect and GKE centralising data types in BigQuery (vendor case study on hex.tech) [S].
- Periodic Labs ($300 M seed, Sept 2025): autonomous labs producing "valuable negative results that are seldom published"; a mismatch between model and experiment was traced "to a machine loaded incorrectly"; "furnaces drift and measurements are noisy" [S].
- Lila Sciences, Senior Engineer Drug Discovery Platform posting: pipelines that "capture identity, purity, dose-response curves, and IC50/EC50 values from automated synthesis and bioassay readers"; own "data quality and lineage" for the closed loop; "comfortable with the messiness of experimental measurements, including replicates, censored values, and failed runs"; familiarity with ELN/LIMS and instrument formats [S].
- Cradle, Scientific Software Engineer posting: pull data "from machines (plate readers, sequencers, NanoDSF, LabChip, Carterra) into our data warehouse", and control instruments programmatically [S].
- Asimov, Staff Data Engineer Lab Data Platform: pipelines "from instruments, our ELN/LIMS and bench workflows into our warehouse (BigQuery)… with quality checks, lineage and versioning built in" [S].
- AWS (Amazon Bio Discovery, 2026): "manual handoffs introduce delays, make it harder to reproduce experiments"; an "experimental data registry" for triage; MSK collaboration designed ~300,000 antibodies and sent 100,000 for testing (vendor figures) [S].
- A-Lab (LBNL): robotic solid-state synthesis → XRD → automated phase analysis. Workflow manager AlabOS uses MongoDB and RabbitMQ [V].

### 2.2 What breaks today

1. **Wet/dry integration is the minority case.** Benchling 2026 Biotech AI Report (survey Nov 2025, ~100 AI-using organisations, vendor-run): data quality/availability is the top reason AI pilots fail (55% per Drug Discovery News); 50% cite IP and compliance friction; strong wet–dry integration at 30% of high adopters versus 18% of low adopters [S].
2. **Labels do not propagate.** Kaleidoscope blog ("Beyond algorithms"): "dataset labels don't propagate correctly, breaking traceability between raw data and ML training inputs"; outlier review is slow and manual and causes "dataset contamination"; proposes completion tracking to show which datasets are ready for training [S, vendor].
3. **Automated analysis of instrument data is itself contestable.** A-Lab's automated XRD phase identification was publicly challenged (Palgrave; a PRX Energy perspective); the authors conceded "a human can perform a higher-quality refinement"; a 2025 LBNL talk on the follow-on method still notes poor fit quality [S]. [I]: derived labels need the analysis activity, its software version and a confidence/quality score attached, and must be replaceable by a later re-analysis of the same raw pattern without losing the earlier one.
4. **Assay comparability.** Landrum and Riniker (JCIM 2024): pooling ChEMBL IC50 values for the same target with minimal curation, ~65% of pairs differ by >0.3 log units and 27% by >1 log unit (Kendall τ 0.51); with maximal metadata matching 48% and 13% (τ 0.71); the reliable check is reading the original protocol [S]. MELLODDY treated each assay as a separate task because partners "typically do not share assay protocols" [S]. Ginkgo antibody-developability competition organisers: datasets "too small and heterogeneous"; call for standardisation [S].
5. **Instrument data is siloed by vendor.** TetraScience estimates ">10 million" discrete silos across biopharma [S, vendor]. MIT Technology Review Insights (sponsored, July 2026): instruments "often standalone", data "locked inside closed systems" [S].
6. **Hardware faults look like science.** Periodic's mis-loaded machine [S]; JUMP lamp drift [S]. [I]: instrument identity, calibration state and consumable lots must be queryable alongside results.
7. **Candidate → experiment linkage is mostly bespoke.** Aqemia posting: model "compounds, structures, assays, and predictions so each has a canonical identity, a provenance record, and trustworthy lineage across… messy sources"; "defend an identity key" [S]. Oracle and AWS marketing both pitch "unifying designs, embeddings, metrics, and provenance into one queryable layer" [S]. No open standard for the design→make→test→learn link was found [I].

### 2.3 Self-driving-lab and instrument data standards

| Standard | Layer | Adoption evidence | Label |
|---|---|---|---|
| Allotrope Simple Model (ASM, JSON) | result data | Benchling `allotropy` (MIT) converters; supported-software table lists 53 instrument-software parsers, all "Recommended", in 12 categories (binding affinity, cell counting, electrophoresis, flow cytometry, LC, liquid handler, multi-analyte profiling, plate reader, solution analyzer, spectrophotometry, dPCR, qPCR), each tagged with an ASM schema version (e.g. `REC/2024/09`, `BENCHLING/2025/03`, `WD/2024/12`); 35 instruments as of Nov 2024 per Pistoia proposal, which estimated 20–30 more would cover ~70% of use cases | [V] table; [S] counts |
| Benchling Connect | vendor integration | "160+ out-of-the-box instrument integrations"; 200+ customers on Connect and Insights; 1,300+ Benchling customers overall | [S] |
| TetraScience IDS / "Tetra Data" | vendor schema | claims 48% of top pharma; partnerships with Snowflake (Dec 2024), Databricks (May 2024), NVIDIA (Nov 2024), Microsoft (Jan 2025), Thermo Fisher (Jan 2026) | [S, self-reported] |
| SiLA 2 | instrument control (gRPC) | "most mature open standard for laboratory instrument connectivity"; an older integration study found "almost no off-the-shelf devices offered SiLA 2 servers"; vendor variation remains | [S] |
| OPC UA LADS | instrument control | first spec Nov 2023; "adoption is still in early stages" | [S] |
| AnIML (ASTM XML) | result data | "cannot command an instrument or stream live data; it represents completed measurements"; no adoption figures found | [S] |
| Citrine GEMD | materials process history | graph of material ← process ← ingredient with Template / Spec / Run separation (plan vs. what was done); used in a PARADIM study | [S] |
| Autoprotocol, ChemOS, MaDICES, ORD | protocols / orchestration / reactions | nothing current retrieved; treat as niche | [P] |
| NIST modular-lab effort | consensus on "data interchange formats including priority instrument types and knowledge graphs" | in progress | [S] |

Reading [I]: there is no single dominant instrument-data standard. The ASM schema-version tags show that even the "standard" is a moving target per instrument class. A catalog should store the raw instrument file as the immutable record, store parsed ASM/IDS/AnIML output as a derived artefact with parser name and version, and keep the plan-versus-run distinction that GEMD and SiLA both make.

---

## 3. Simulation and computed data

### 3.1 Level of theory as the "instrument"

- UMA (Meta, 2025) trains one model on five DFT datasets and conditions on a **task embedding** naming the level of theory: `omat` (PBE/PBE+U, VASP), `omol` (ωB97M-V/def2-TZVPD, ORCA 6), `oc20` (RPBE, VASP 5.4 pseudopotentials), `odac`, `omc`; charge and spin are also embedded; datasets are sampled at ratios 4:4:1:1:2; OC20 had to be **recomputed** to give total energies instead of adsorption energies [S].
- OMol25 docs: to reproduce, use `fairchem.data.omol.orca.calc` to write compatible ORCA inputs [V]. MatPES published its VASP settings ("PBE 64" pseudopotentials) so fine-tuning data can be generated consistently [S].
- MPtrj/Alexandria/OMat24 inherit Materials Project's selective +U; a 2026 preprint recommends dropping +U and proposes per-atom shifts as a post-hoc fix [S].

Catalog consequence [I]: every computed label needs a structured, comparable "method" record (code and version, functional, basis or plane-wave cutoff, pseudopotential set, dispersion, +U values, spin treatment, k-mesh, convergence thresholds, energy reference). Two labels are mergeable only if a declared compatibility rule over that record says so. Label *transformations* (referencing, per-atom shifts, normalisation) are provenance activities, not silent edits.

### 3.2 Dedup, filtering and benchmark-aware exclusion

- OMat24 train/val "do not contain any structure that has a protostructure label present in the initial or relaxed structures of the WBM dataset", nor any structure generated from an Alexandria structure with such a label [V]. So training-set construction is defined *relative to a named test set* by a structural equivalence key.
- LeMat-Bulk: selects static calculations per material, filters "non-compatible entries", checks convergence, fingerprints structures [S].
- OMol25: tight SCF thresholds and grid choices in the appendix; only 75 M samples were available when UMA trained ("OMol-preview") [S]. [I]: models are trained on pre-release snapshots that differ from the published dataset, so the exact snapshot must be pinned.

### 3.3 Workflow provenance tools

AiiDA (automatic provenance graph, Materials Cloud Archive with DOIs) [S]; QCArchive ("tens of millions" of QC computations) [S]; atomate2/jobflow, FireWorks, Signac, OPTIMADE [P, not re-checked]. CuspAI's Materials Knowledge Architect posting asks for knowledge of VASP and Quantum ESPRESSO and of ICSD, CSD, NOMAD and Materials Project, and ownership of "data quality, validation, deduplication, lineage and provenance frameworks" [S]. [I]: startups ingest from several of these at once, so the catalog should import an AiiDA or jobflow graph as PROV rather than ask for re-entry.

### 3.4 HPC-to-cloud movement

- OMol25 raw outputs live on Argonne's Eagle with approval-based access; dissemination "still being worked out" [S].
- Multimodal Universe: Globus preferred for bulk; NERSC portal mirror exists [V/S].
- AmSC (DOE): "Globus transfer use through AmSC Data Movement Service" and Globus Auth for federation; Fermilab provides storage via its data platform [S].
- OSDF/Pelican: >100 PB/year, mostly HTC inputs; NCAR RDA integration [S].
- Neocloud friction: vendor-sourced estimates of ~$0.09/GB S3 egress, >$4,000 to move 50 TB, $25–45k for 500 TB; CoreWeave "Zero Egress Migration" (Nov 2025) covers the one-time move only [S, vendor material].

[I]: replicas across {hyperscaler bucket, neocloud object store, HPC POSIX/tape via Globus} with per-replica access protocol and cost class is a baseline need, and "which replica is nearest to this compute" is a routine query.

### 3.5 Mixed fidelity

Besides UMA's task embedding: MatPES ships PBE and r2SCAN for the same structures; MP-ALOE is "completely compatible" with MatPES r2SCAN [S]; a 2026 paper trains multi-fidelity heads [S]. [I]: the same structure with labels at several fidelities is one sample with several label columns, each carrying its own method record; the catalog must make "same structure" an explicit key (hash/fingerprint) separate from "same calculation".

---

## 4. Domain metadata standards: must-have versus niche

Mostly [P]; items checked this session are marked.

**Must respect (users will reject a catalog that cannot round-trip these)**

- **AnnData/h5ad** and **TileDB-SOMA**: obs/var tables plus sparse X and layers. Every flagship single-cell set above uses one of them [V/S]. CELLxGENE schema (ontology-coded obs fields, `is_primary_data`) is the de-facto metadata profile.
- **OME-Zarr (NGFF)** and OME-TIFF: 0.5 is on Zarr v3 with metadata under an `ome` key in `zarr.json`; plate/well/field/channel hierarchy; "not possible to mix collections of images with plates at present" [S]. Biohub postings list OME-Zarr and HDF5 [S].
- **PDBx/mmCIF** (and BinaryCIF), with PDB ID, entity/chain, release date; **UniProt accessions** (AFDB file names embed them [V]); **CCD ligand codes** (Runs N' Poses keys on them [V]).
- **Chemistry identifiers**: SMILES/InChIKey, ChEMBL and PubChem IDs; **Cellosaurus** for cell lines (Tahoe uses it [V]); Ensembl/HGNC genes.
- **CF conventions** with GRIB parameter codes (ARCO-ERA5 maps every variable to an ECMWF param-db entry and unit [V]); **STAC** for imagery (Clay indexes chips against STAC items [S]).
- **ASE atoms / extxyz / LMDB**, **CIF**, and **OPTIMADE** structure fields for materials [V for aselmdb].
- **HDF5** and **ROOT/NanoAOD** as containers.
- **GA4GH DRS** for controlled-access genomics: DRS 1.5 released; ~23 servers on a prototype map; AnVIL/Terra, Seven Bridges CGC and Cavatica in production; Gen3 [S]. **Passports/Visas** for carrying access grants.
- **Units**: UCUM strings at minimum (QUDT optional). The evidence is indirect: RxRx3 puts the unit in the column description ("in uM"), Tahoe concatenates drug, concentration and unit in one string [V]. [I]: units as typed column metadata are a gap in current practice.
- **DOIs / DataCite** for released snapshots (Zenodo, Figshare+, Materials Cloud all used above).

**Bridge where the user segment needs it**

- **Allotrope ASM**, vendor IDS (lab data); **GEMD** (materials process history).
- **DICOM, OMOP** (only for clinical-data companies; Recursion's Tempus/Helix records [S]).
- **FASTQ/BAM/CRAM + htsget**, **mzML** (PNNL's mzPeak proposal, April 2026, signals format churn in mass spec [S]).
- **ISA-Tab, MIAME/MINSEQE, BioSamples**: needed mainly when importing from or depositing to public archives.
- **IMAS data dictionary** for fusion (Fusion Data Platform and IAEA catalogue both map to it [S]).
- **GeoZarr**: status not retrieved; treat as emerging.

**Niche for this audience**: EMMO, Beacon, RRIDs (useful as an optional reagent/instrument identifier field), AnIML, autoprotocol.

---

## 5. Governance and regulation

### 5.1 Human-subject and controlled-access data

- **NIH NOT-OD-25-081 (28 Mar 2025)**: generative models, "including model parameters", built by Approved Users on controlled-access human genomic data are **Data Derivatives**; they may not be shared except with other Approved Users and may not be retained after project close-out; entering such data into public generative tools is prohibited. NOT-OD-25-118 (30 May 2025) sought input; no final guidance found [S].
- **UK Biobank RAP**: default route since 2024; "Researchers cannot download individual-level data… This includes bulk datasets, tabular data, and any variables generated within the platform"; exemptions by application; automated output review being procured from January 2026; UK Biobank states that downloading and offering data for sale are breaches (2026 incidents referenced, details unverified) [S].
- **All of Us**: no participant-level data may leave the Workbench; egress volume alarms disable accounts; no publication of statistics for fewer than 20 participants without an exception [S].
- HIPAA and GDPR basics (de-identification, purpose limitation, erasure, cross-border transfer) [P].

Catalog consequences [I]: (a) a model is a governed derivative, so licence/consent must propagate dataset → model → predictions → any dataset distilled from the model, with expiry dates; (b) some data can only ever be *referenced* inside an enclave, so the catalog must hold entries whose bytes it cannot read, with a "location is an enclave, no egress" property; (c) aggregate-release rules (minimum cell count) are a policy attached to data, not to users; (d) erasure requests require finding every derived artefact containing a subject.

### 5.2 GxP

- 21 CFR Part 11 and ALCOA+ (attributable, legible, contemporaneous, original, accurate, plus complete, consistent, enduring, available): time-stamped audit trails, e-signatures, record immutability [P; not retrieved this session].
- **FDA draft guidance (Jan 2025)** on AI for regulatory decision-making: risk-based credibility framework by context of use; the plan must describe the model and the development, training, tuning and test data; discovery-stage AI is out of scope [S].
- **EU GMP draft Annex 22 (consultation 7 Jul–7 Oct 2025; finalisation expected 2026)**: test data must be kept separate from training and validation data by technical and procedural controls; staff with test-data access generally cannot train the same model; dynamic and generative models excluded from critical GMP uses; revised Chapter 4/Annex 11 strengthen audit trails and e-signatures [S].
- Vendors already sell this: Seqera GxP (audit logging, immutable versioning, RBAC, run tied to pipeline version and checksums), Kaleidoscope (21 CFR Part 11, SOC 2 Type 2), Benchling Validated Cloud [S].

Consequence [I]: discovery-stage startups are mostly outside GxP, but pharma partners and later-stage work are not. Append-only audit of every metadata mutation, immutable versions, and *enforced* split separation (access control on test partitions, logged reads) are cheap to design in and expensive to retrofit.

### 5.3 IP and partner segregation

- **MELLODDY**: 10 pharma, 2.6 B+ activity points, 21 M+ molecules, 40k+ assays; data stayed behind each firewall; each partner prepared data on premise from a shared manual; ledger for traceability [S].
- **AISB / Apheris federated OpenFold3**: AbbVie and J&J (Mar 2025), then BMS, Takeda, Astex (Oct 2025); fine-tuning completed in under ten weeks without sharing data; weights belong to members; reported to use ~5× more proprietary structural data than the PDB (vendor claims) [S].
- Inductive Bio: pre-competitive consortium with contributed anonymised ADMET data "in a secure IP-protected environment" [S]. Periodic Labs: systems "can be trained on partner data" [S]. AWS: "proprietary data stays isolated and is never used to train hosted models" [S].

Consequence [I]: per-partner compartments with hard isolation, plus the ability to answer "which models have seen partner X's data?" and "which artefacts must be destroyed when contract X ends?" are core queries. Federated setups need catalog entries for remote datasets the operator cannot see, described by schema and statistics only.

### 5.4 Biosecurity and export control

- Proposed five-tier **Biosecurity Data Levels** (arXiv 2602.08061, NeurIPS 2025 workshop; >100 researchers after Asilomar 50th endorsed data controls); Evo 2 and ESM3 withheld eukaryote-infecting viral sequences; fine-tuning on the withheld data restored capability. Not adopted as policy [S].
- BIS interim final rule on biotechnology equipment (Jan 2025) reasoned from data generation for AI; S.3741 Biosecurity Modernization and Innovation Act introduced Feb 2026; China proposed export-catalogue additions for 23 AI technologies (Oct 2026) [S].

Consequence [I]: datasets need a hazard/sensitivity classification independent of licence, and "exclusion filters applied during training-set construction" must be recorded so a lab can prove what was withheld.

### 5.5 Mandates and "AI-ready" criteria

- NIH DMS Policy in force since 25 Jan 2023; NIH Public Access Policy effective 1 Jul 2025; several agencies' Nelson-memo policies effective 31 Dec 2025 (status not re-verified) [S]. Horizon Europe DMP/FAIR requirement [P].
- DOE: Genesis Mission executive order (Federal Register 28 Nov 2025) creating the American Science and Security Platform; >$320 M announced Dec 2025; AmSC described as a "federated platform that connects AI models, curated scientific data, workflows, and computing resources across DOE laboratories" with "FAIR, AI-ready datasets"; DOE RFI (5 Dec 2025) on partnerships to curate data; PNNL "AI-Ready Data Pilot Project Report" (27 Feb 2026) proposes data requirements, metadata standards and validation tests; arXiv 2507.23018 proposes Data Readiness Levels × Data Processing Stages [S].
- Google DeepMind "AI data stocktakes" essay (2026) with fusion (JET archive) as the case study [S].
- Bridge2AI and ESIP AI-readiness checklists: not retrieved; [P] they cover provenance, documentation, ethics/consent, pre-model explainability and sustainability.

---

## 6. Infrastructure reality at startups

### 6.1 Stack as seen in postings and case studies (all [S])

| Organisation | Evidence |
|---|---|
| Recursion | Prefect + GKE, BigQuery as the centralised store, on-prem BioHive-2 for training, cloud for inference |
| insitro | Airflow, Argo, Kubernetes, Ray, **redun**, PyTorch Lightning; "full imaging data lineage, from on-microscope acquisition to high-throughput ML pipelines"; "terabytes of data per day"; "petabyte-scale cellular data platform" |
| Biohub/CZI | S3, Parquet, Athena, Kinesis, MongoDB, Airflow, Terraform; "petabyte-scale data pipelines and platform tooling to ingest, transform, catalog, and deliver heterogeneous biological datasets for AI research"; OME-Zarr, HDF5; convert public-repository data to AI-ready and combine with proprietary |
| Lila Sciences | AWS, Kubernetes, Python/SQL, lakehouse, vector DB/RAG; "validation checks and schema-evolution gates"; roles from Senior DE to Director of Data Platform Engineering and Principal Data Architect |
| Asimov | BigQuery warehouse fed from instruments and ELN/LIMS |
| Cradle | instrument → warehouse ETL, programmatic instrument control |
| Aqemia | Bronze → Silver → Gold with lineage; canonical identity for compounds, structures, assays, predictions |
| CuspAI | Data Engineer integrating "materials databases, literature, patent filings, and laboratory instruments"; Materials Knowledge Architect owning schemas and provenance |
| Periodic Labs | Research Engineer – Data: "dataset versioning, lineage tracking, and reproducibility tooling such as DVC, Delta Lake, or custom solutions" (returned by one search extract; a second search of the same posting did not reproduce the phrase, so treat as unconfirmed); combine "text, numerical, molecular, and spectral data"; a role to "source, evaluate, and procure external datasets" |
| Isomorphic Labs | Data Services (Django, PostgreSQL), Bioinformatics Platform ("efficient data stores for machine learning models"), Data Curator (Chemistry), Data Management Specialist |
| Inductive Bio | "harmonize, validate, and version scientific data"; "schemas for integrating heterogeneous data types (chemical, image-based, genomic)" |
| Lumen Bioscience | Benchling API integrations |
| Absentia Labs, Amgen | "provenance, lineage, and versioning suited to scientific and ML workflows"; "document data lineage, assumptions, validation outcomes" |

Pattern [I]: bucket + warehouse (BigQuery/Snowflake/Databricks) + general orchestrator (Airflow, Prefect, Dagster, Argo, redun) + Benchling as ELN/registry + home-grown lineage. "Lineage/provenance" is in almost every posting, and Periodic's "or custom solutions" shows no accepted product fills the slot. Several companies hire dedicated *curators* and *data procurers*, roles with no analogue in a web-scale ML lab.

### 6.2 Build versus buy

Bought: ELN/LIMS (Benchling: 1,300+ customers), instrument integration (Benchling Connect, TetraScience, Ganymede), warehouse, workflow SaaS (Seqera), notebooks/BI (Hex). Built: identity keys across modalities, assay-result models, dataset assembly for ML, lineage, split logic. Recent consolidation: Apprentice.io acquired Ganymede (21 Jan 2026); Benchling "Direct Data Share" to Snowflake and "Science Foundries" (announced 15 Oct 2026 per search extract); Benchling AI Connectors on MCP (GA 2026); Seqera–Benchling integration via MCP (Apr 2026) [S].

### 6.3 Headcount

No public numbers found. Indirect signals [S]: Lila has at least six distinct data-platform role titles open at once; Benchling's own internal data team is "a small, autonomous team"; Aqemia has an Engineering Manager and a Staff Data Engineer above the Senior DE role. [I]: a 50–150 person techbio typically shows 2–6 people on data platform; this is my estimate, not sourced.

### 6.4 Pain statements

- "55%… data quality" as the top cause of failed AI pilots (Benchling survey) [S].
- "wet lab and dry lab work have lived in separate worlds" (Benchling blog) [S].
- "dataset labels don't propagate correctly" (Kaleidoscope) [S].
- "establishing canonical identity, provenance and trustworthy lineage across heterogeneous, often messy sources" (Aqemia) [S].
- "replicates, censored values, and failed runs" (Lila) [S].
- BVP essay and commentary: ~63% of biology AI models trained on UniProt/PDB sequence-structure data (citing Epoch AI); 380+ new biology models per year; the model boom "has not solved the underlying data problem"; build data generation backward from what the model needs [S].
- The literal complaint "cannot find which assay version trained which model" was not found in a public source. The closest are the Kaleidoscope and Aqemia statements. Label: [I] that this is a common failure.

---

## 7. Evaluation, splits and leakage

### 7.1 Documented leakage and split practice

- **PDBbind / CASF**: heavy train–test overlap; LP-PDBBind and CleanSplit re-split by protein and ligand similarity; retrained models drop sharply [S].
- **Co-folding**: AlphaFold 3's structural training cutoff is 30 Sep 2021. **Runs N' Poses** is "a zero-shot benchmark, provided that your method uses a structural training cutoff of 30 September 2021"; it ships `all_similarity_scores.parquet` with similarity of every benchmark system "against the entire PDB up until 5 January 2025", per-system `target_release_date` and the closest training system, and shows how to recompute for another cutoff ("the 1 June 2023 cutoff used by Boltz-2"); computing the similarities "requires an entire copy of the PDB, the PLINDER dataset, and large amounts of memory" [V].
- **PLINDER**: >400k protein–ligand systems; "reusable ligand, pocket, and protein-interface similarities and cluster assignments"; splits "curated to minimize the information leakage based on protein-ligand interaction similarity"; splits changed between dataset versions [V].
- **Matbench Discovery** (Nat. Mach. Intell. 7, 836–847, 2025): compliant models train on MPtrj only; non-compliant use OMat24, sAlex etc.; OMat24 removes WBM protostructures from its splits [V/S].
- **WeatherBench 2**: ERA5 is both training data and ground truth; training periods differ per model (FourCastNetV2 1979–2015, Pangu 1979–2017, Aurora and AIFS to 2020), leaving 2021 as the only year outside all of them [S].
- **Virtual Cell Challenge 2025**: leaderboard metrics were gamed; Arc added a "Generalist Prize" over seven metrics and made 2026 zero-shot with no training set [S].
- **Polaris**: "an immutable dataset… alone doesn't guarantee reproducible model comparisons. How that dataset is split, the metrics used, and other evaluation details also matter"; a benchmark = dataset version + split indices + metrics; Zarr storage; certification and method-comparison guidelines [V/S].
- **ADMET blind challenges** (ASAP × OpenADMET × Polaris, 2025): careful curation plus multitask learning placed second using public data only [S].
- **Antibody developability** (Ginkgo GDPa1, 242–246 antibodies × 10 assays): limited model performance [S].
- CASP16: AF3 a "modest improvement"; ligand affinity "totally unreliable" [S].

### 7.2 What a catalog must record for leakage to be checkable [I, grounded in 7.1]

1. **Per-sample entity keys in domain identifier spaces**: PDB ID + chain + CCD code, UniProt accession, InChIKey/scaffold, structure fingerprint/protostructure, cell line, donor/patient, plate/batch, valid-time for forecasts.
2. **Per-sample event dates**: deposition/release date, acquisition date, valid time. Time splits and training cutoffs are queries over these.
3. **Similarity/cluster assignments as versioned, reusable derived tables** tied to (method, parameters, reference snapshot): MMseqs2/Foldseek clusters at a stated identity, Tanimoto on a stated fingerprint, SuCOS-pocket, protostructure labels. PLINDER and Runs N' Poses both publish exactly this.
4. **Splits as first-class immutable objects**: named, versioned sets of sample IDs with the rule that produced them and the dataset version they index.
5. **Training manifests**: the exact dataset snapshot(s) and row sets each model checkpoint saw, including pretraining ancestors and any distillation source (a model trained on predictions of another model inherits that model's training set).
6. **Exclusion declarations**: "this set excludes everything equivalent to benchmark B under key K" (OMat24–WBM) stored as a checkable assertion.
7. **A leakage query**: for model M and evaluation set E, list E-samples whose entity key, or cluster under similarity S, intersects M's transitive training manifest, or whose date precedes M's cutoff.
8. **Test-partition access control and read logging** (Annex 22 direction).
9. **Benchmark results linked to (model version, benchmark version, split, metric implementation version)**; the Virtual Cell Challenge dispute was partly about metric implementation differences [S].

---

## 8. Products aimed at this market (2025–2026) and the gap

| Product | Covers | Adopters / status | What it does not do [I] |
|---|---|---|---|
| TetraScience | instrument/file capture → vendor schema ("Tetra Data") → lakehouse hand-off | claims 48% of top pharma; Takeda lighthouse; ShapeTX case | not a dataset/model lineage catalog; pharma-priced; closed schema |
| Benchling (ELN/LIMS/registry, Connect, warehouse, AI agents, MCP connectors, Snowflake share) | entities, results, notebooks, 160+ instruments | 1,300+ companies | bulk scientific files and ML datasets/models live elsewhere; lineage stops at the warehouse table |
| Ganymede (now Apprentice.io) | "Lab-as-Code" instrument integration | acquired Jan 2026 | moving toward manufacturing |
| Kaleidoscope | R&D operations: projects, decisions, "full lineage, reasoning, and permissions on every record" | Crossbow Therapeutics named | decision/record level, not byte-level data or training manifests |
| Quilt | versioned S3 "packages" with metadata; Benchling + Nextflow packaging | not retrieved | package granularity; no sample-level tables or pointers into files |
| LatchBio | workflows, Pods, Plots, Registry; white-label portals for kit/instrument makers; released 25–30 M cell spatial atlases | "300+ biopharmas and R&D labs" (self-reported) | omics-centred; hosted |
| Seqera Platform | Nextflow runs, Data Explorer, Studios, Fusion; native lineage IDs per run/task/output since Nextflow 25.04, searchable on objects (Enterprise 26.1, May 2026); "linked datasets" by URL; GxP package; Benchling integration | wide Nextflow base | lineage only for what Nextflow ran; no models/predictions/experiments |
| DNAnexus, Terra/AnVIL, Verily Workbench | genomics enclaves, DRS, controlled access | UK Biobank RAP (DNAnexus), All of Us (Verily/Terra lineage) | enclave-bound; genomics-first |
| Code Ocean, Deep Origin | reproducible "capsules"/compute workbenches | not retrieved | computation-centred |
| TileDB (Carrara, Oct 2025) | arrays + catalog, "omnimodal"; SOMA for single-cell | Census runs on TileDB-SOMA; claims top-10 pharma; waitlist page | requires adopting the TileDB engine/format |
| LaminDB | lineage-aware catalog for biology; mirrors Arc atlas ("2.5B expression profiles… ~600M cells", 460k files, 41 TB) | Arc README links the mirror [V] | bio-only ontologies; closest open-source neighbour to the proposed design |
| earthmover (Arraylake, Icechunk, Flux) | transactional array store + team catalog for weather/geo | >10 paying customers at Sept 2025 seed ($7.2 M); customers with tens–hundreds of TB; ~1 PB migrated; NOAA NWS CIRRUS to use Icechunk | arrays only |
| Polaris Hub | drug-discovery datasets and benchmarks (Zarr), certified splits | industry steering committees | benchmark hub, not an internal catalog |
| CZI Virtual Cells Platform, cz-benchmarks, CELLxGENE | curated models + datasets + benchmarks | public | public-data service |
| Synapse (Sage) | governed sharing of biomedical research data | NIH consortia [P] | hosted portal |
| Globus; Pelican/OSDF | transfer/auth; federated object delivery (>100 PB/yr) | AmSC, NCAR, LIGO | movement, not meaning |
| DOE AmSC | federated models + data + compute across 17 labs | early 2026 build-out | government platform |
| Apheris | federated compute over private pharma data | AISB (5 pharma) | governance of computation, not a catalog |
| AWS Amazon Bio Discovery | hosted design → CRO testing → "experimental data registry" loop | MSK | AWS-bound, antibody-centred |

**Gap** [I]: each product owns one segment of the chain. Instrument capture (Tetra, Connect, Ganymede) ends at a warehouse table. Workflow lineage (Seqera, AiiDA) covers only what that engine ran. ELN/registry (Benchling) knows the sample and the request but not the bytes or the model. Experiment trackers know the model but not the assay protocol. Benchmark hubs (Polaris, Matbench, PLINDER) know splits but not private training sets. No retrieved product links raw instrument or simulation output → curated dataset snapshot → model → prediction/candidate → validating experiment in one queryable, access-controlled graph across modalities (plates, structures, trajectories, tensors) while leaving files in place. The 2026 trend is point-to-point stitching through MCP connectors (Benchling ↔ Seqera, Benchling ↔ Snowflake), which gives agents reach but not a shared identity or provenance model.

---

## 9. Requirements for the catalog, each tied to evidence

"Differs from frontier lab" compares against a web-scale text/image/video setting.

| # | Requirement | Evidence | Differs from frontier lab? |
|---|---|---|---|
| R1 | Sample rows that join several file pointers (channels, sites, modalities) with experimental-design factors; controls are rows | RxRx3 schema, Tahoe obs [V] | Yes. Web data is one blob + caption; here the design matrix is the label |
| R2 | Pointers into containers: tar member, HDF5 dataset, Zarr chunk/time slice, LMDB key, byte range | AFDB tar shards, aselmdb, ARCO chunks [V] | Partly. WebDataset shards are similar, but ND-array slices and LMDB keys are not typical |
| R3 | Multiple representations of one logical dataset (raw / cloud-optimised / analysis-ready / streaming mirror), linked by lineage | ARCO `raw/co/ar`, Tahoe h5ad+Parquet, MMU HDF5+Parquet [V] | Yes in degree |
| R4 | Dated immutable snapshots with supersession/deprecation and continuous append | scBaseCount releases, PLINDER versions, ERA5 monthly append [V] | Similar concept; append-in-time arrays differ |
| R5 | Replicas across cloud, neocloud and HPC (Globus/POSIX/tape) with checksums, cost class and "no-egress" locations | OMol25 at Argonne, MMU Globus, AmSC, egress pricing [S/V] | Yes. Frontier labs own their storage; these users straddle national HPC and rented GPUs |
| R6 | Entries for data the catalog cannot read (enclaves, partner sites, federated nodes), described by schema and statistics | UKB RAP, All of Us, MELLODDY, AISB [S] | Yes |
| R7 | Structured "method" record for computed labels (code, functional, basis, pseudopotentials, +U, spin, thresholds, reference) with declared compatibility | UMA task embedding, MPtrj +U problem, MatPES [S] | Yes. No analogue |
| R8 | Instruments, calibration state, reagent/consumable lots, operator, protocol version as provenance agents/entities; plan versus run | Periodic mis-load, JUMP drift, GEMD, Lila posting [S] | Yes |
| R9 | Assay as a versioned entity; results mergeable only under declared protocol equivalence; censored values, replicates, failed runs and outlier flags representable | Landrum & Riniker, MELLODDY per-assay tasks, Lila posting, Kaleidoscope [S] | Yes |
| R10 | Negative and failed results retained and queryable as first-class data | Periodic, Lila [S] | Yes. Web pipelines discard failures |
| R11 | Batch structure (plate, run, site, day, sublibrary) and control-plate links as queryable fields | JUMP Target2 plates, Tahoe sublibrary [S/V] | Yes |
| R12 | Derived labels carry the analysis activity, software version and quality score; re-analysis adds a new label version without destroying the old | A-Lab XRD dispute, scBaseCount agent curation [S/V] | Partly (relabelling exists) but the scientific stakes differ |
| R13 | Design–make–test–learn linkage: model version → candidate → synthesis/order → sample → assay result → next training snapshot, with canonical identity per entity | Frey et al. loop, Aqemia posting, AWS registry [S] | Yes. Closest analogue is RL/eval logs, without physical samples |
| R14 | Domain identifier columns as typed, validated keys (UniProt, PDB/CCD, InChIKey, ChEMBL, Cellosaurus, Ensembl, GRIB param, STAC item, DOI) | Tahoe, AFDB, RNP, ARCO [V] | Yes |
| R15 | Typed units and physical quantities on columns | implicit units in RxRx3/Tahoe [V]; CF units in ARCO [V] | Yes |
| R16 | Per-sample event dates and entity keys; versioned similarity/cluster tables; splits as immutable objects; training manifests; exclusion assertions; a leakage query | Runs N' Poses, PLINDER, OMat24–WBM, WB2, Polaris [V/S] | Yes in kind. Frontier contamination checks are n-gram/hash; here leakage is defined by scientific similarity under a chosen metric and by time |
| R17 | Licence, consent and contract terms propagate to models, predictions and distilled data, with expiry and destroy-on-closeout | NIH NOT-OD-25-081, MMU per-subset terms, RxRx non-commercial, Xaira non-commercial [S/V] | Partly. Copyright provenance exists at frontier labs; consent scope, close-out destruction and model-as-derivative do not |
| R18 | Per-partner compartments with hard isolation and "which models saw partner X" queries | MELLODDY, AISB, Inductive, Periodic [S] | Yes |
| R19 | Column/row-level redaction and later unblinding without changing sample identity | RxRx3 anonymised genes [V] | Yes |
| R20 | Append-only audit trail of metadata changes, e-signature hooks, enforced and logged test-partition access | Annex 22 draft, Part 11, vendor GxP offerings [S/P] | Yes |
| R21 | Hazard/biosecurity classification on datasets and recorded training-time exclusion filters | BDL proposal, Evo/ESM3 exclusions [S] | Partly (safety filtering exists) but the classification axis is different |
| R22 | Aggregate-release policies attached to data (minimum counts) | All of Us <20 rule [S] | Yes |
| R23 | Import/export bridges: h5ad/SOMA obs-var, OME-Zarr plate layout, mmCIF, ASE/OPTIMADE, CF/STAC, ASM JSON, AiiDA/Nextflow lineage → PROV, DRS URIs | sections 3–4, Seqera lineage [S/V] | Yes |
| R24 | Citation/acknowledgement bundle and DOI minting per snapshot | MMU citation tool, Zenodo/Figshare use [V/S] | Yes |
| R25 | Works for a 2–6 person data team: single Postgres, no cluster to run, scriptable from Airflow/Prefect/Dagster/redun/Nextflow, MCP-addressable for agents | postings in 6.1, MCP trend in 6.2 [S] | Yes. Frontier labs staff platform teams |
| R26 | Scale envelope: 10^8–10^9 sample rows, 10^6–10^9 file or member entries, 10 TB–10s of PB, tens of thousands of dataset snapshots | Tahoe, Census, AFDB, Recursion, ARCO [V/S] | Smaller row counts than web scale, but far more columns, joins and governance per row |
| R27 | Provenance marker for agent-extracted versus human-asserted metadata; predictions and agent conclusions traceable to data and code | scBaseCount agents [V], Kosmos "every conclusion… traced to code or literature" [S] | Emerging in both settings |

### Where the frontier-lab setting is the same

Content-addressed immutable blobs, sharded containers, replica placement, dataset snapshots, embedding/vector indexes, and training manifests are shared needs. Raw byte volume at the largest science organisations (Recursion 65 PB, ARCO-ERA5 PB-scale) is comparable to mid-size web corpora.

### Where it differs most [I]

1. Meaning lives in the experimental design and the measurement method, so metadata per sample is wide, typed and ontology-bound, and it is wrong to treat it as free-text tags.
2. Data arrives from physical processes with batch structure, drift and failure; replicates, controls and negatives are part of the dataset.
3. "Same thing" is defined by scientific similarity (sequence identity, scaffold, protostructure, valid time), so evaluation hygiene needs similarity tables and dates, not hashes.
4. Governance is contractual and regulatory per source (consent, enclave, partner, GxP, biosecurity) and extends to models.
5. Data is spread over cloud, rented GPU clouds and public HPC, and much of the valuable raw tier is never copied.
6. Teams are small and buy most of the stack; the catalog has to sit beside Benchling, a warehouse and a workflow engine and import their identities.
7. The loop closes through the physical world: predictions become samples and experiments, which become training data, so lineage is cyclic across model generations.

---

## 10. Source list (URL, date where known, label)

Verified in full (GitHub raw, fetched 2026-11-23):
- https://github.com/recursionpharma/rxrx-datasets/blob/trunk/rxrx3/README.md (Jan 2023 release)
- https://github.com/ArcInstitute/arc-virtual-cell-atlas (README, tahoe-100M/README.md, scBaseCount/README.md; release 2026-01-12)
- https://github.com/PolymathicAI/the_well (README)
- https://github.com/MultimodalUniverse/MultimodalUniverse (README)
- https://github.com/google-research/arco-era5 (README)
- https://github.com/Benchling-Open-Source/allotropy/blob/main/SUPPORTED_INSTRUMENT_SOFTWARE.adoc
- https://github.com/polaris-hub/polaris (README)
- https://github.com/brightbandtech/nnja-ai (README; v1 Aug 2025)
- https://github.com/chanzuckerberg/cellxgene-census (README)
- https://github.com/CederGroupHub/alab_management (README)
- https://github.com/plinder-org/plinder (README) and https://github.com/plinder-org/runs-n-poses (README)
- https://github.com/janosh/matbench-discovery (readme)
- https://github.com/ecmwf/anemoi-datasets (README)
- https://github.com/facebookresearch/fairchem docs: `docs/molecules/datasets/omol25.md`, `docs/inorganic_materials/datasets/omat24.md`
- https://github.com/google-deepmind/alphafold/blob/main/afdb/README.md

Search-snippet only (not opened):
- Tahoe-100M: https://www.biorxiv.org/content/10.1101/2025.02.20.639398 (Feb 2025); https://huggingface.co/datasets/vevotx/Tahoe-100M ; scDataset https://arxiv.org/abs/2506.01883
- Arc atlas and challenge: https://arcinstitute.org/news/arc-virtual-cell-atlas-launch (Feb 2025); https://arcinstitute.org/news/virtual-cell-challenge-2025-wrap-up ; https://arcinstitute.org/news/virtual-cell-challenge-2026 ; https://blog.lamin.ai/arc-virtual-cell-atlas
- Xaira: https://www.biorxiv.org/content/10.1101/2025.06.11.659105 (Jun 2025); Business Wire 2026-03-17 (Pisces)
- CELLxGENE: https://academic.oup.com/nar/article/53/D1/D886/7912032 (2025)
- RxRx3-core: https://arxiv.org/abs/2503.20158 (Mar 2025); Recursion pages https://www.recursion.com/operating-system , /platform, /mission
- JUMP: https://broadinstitute.github.io/jump_hub/explanations/data_description.html ; https://www.nature.com/articles/s41592-024-02241-6 ; https://pmc.ncbi.nlm.nih.gov/articles/PMC11297288
- OMol25: https://arxiv.org/abs/2505.08762 (May 2025); https://newscenter.lbl.gov/2025/05/14/computational-chemistry-unlocked-a-record-breaking-dataset-to-train-ai-models-has-launched/ ; https://fair-chem.github.io/molecules/datasets/omol25_elec.html
- OMat24: https://arxiv.org/abs/2410.12771 (Oct 2024). UMA: https://arxiv.org/abs/2506.23971 ; https://fair-chem.github.io/core/uma.html
- MatPES: https://arxiv.org/abs/2503.04070 ; "Better without U": https://arxiv.org/abs/2601.21056 ; LeMat-Bulk: https://arxiv.org/abs/2511.05178 ; LeMat-Traj: https://arxiv.org/abs/2508.20875
- GNoME: https://www.nature.com/articles/s41586-023-06735-9 ; A-Lab dispute: https://chemistryworld.com/news/new-analysis-raises-doubts-over-autonomous-labs-materials-discoveries/4018791.article
- AFDB: https://academic.oup.com/nar/article/52/D1/D368/7337620 ; https://www.ebi.ac.uk/pdbe/news/alphafold-database-release-notes
- ESM3: https://huggingface.co/EvolutionaryScale/esm3-sm-open-v1
- Weather: https://www.ecmwf.int/en/about/media-centre/aifs-blog/2025/introducing-anemoi-training-ready-version-era5 ; https://arxiv.org/abs/2308.15560 (WB2); https://sites.research.google/weatherbench/faq/ ; https://developers.google.com/weathernext ; https://nnja-ai.readthedocs.io/en/stable/faq/ ; https://clay-foundation.github.io/model/release-notes/data_sampling.html
- earthmover: https://techcrunch.com/2025/09/22/earthmover-wants-to-become-the-snowflake-of-weather-and-geospatial-data ; https://www.businesswire.com/news/home/20250919639500/en
- Physics: https://arxiv.org/abs/2412.00568 ; https://arxiv.org/abs/2412.02527 ; https://arxiv.org/abs/2412.10504 (Aspen Open Jets); Fusion Data Platform https://archive.aps.org/dpp/2025/pp13/82 ; TokaMark https://arxiv.org/abs/2602.10132
- DOE: https://www.govinfo.gov/content/pkg/FR-2025-11-28/html/2025-21665.htm ; https://amsc.energy.gov ; https://newscenter.lbl.gov/2026/07/15/how-the-genesis-missions-american-science-cloud-advances-innovation/ ; https://www.pnnl.gov/publications/ai-ready-data-pilot-project-report (2026-02-27); https://arxiv.org/abs/2507.23018 ; https://www.govinfo.gov/content/pkg/FR-2025-12-05/html/2025-22127.htm
- Lab loop: Frey et al. bioRxiv Feb 2025 (via https://twistbioscience.com/resources/webinar/Lab-in-the-Loop-Smarter-AI-for-Antibody-Design); https://aws.amazon.com/blogs/industries/introducing-amazon-bio-discovery ; https://techcrunch.com/2025/09/30/former-openai-and-deepmind-researchers-raise-whopping-300m-seed-to-automate-science ; https://a16z.com/announcement/investing-in-periodic-labs/ ; https://hex.tech/blog/recursion-data-team-accelerates-drug-discovery/ ; https://www.technologyreview.com/2026/07/27/1139667/ (sponsored); https://www.preprints.org/manuscript/202608.2107
- Standards: https://www.benchling.com/blog/open-source-data-standards-allotrope ; Pistoia proposal https://www.pistoiaalliance.org/?p=26052 (Nov 2024); https://arxiv.org/abs/2606.03755 (LAP); https://citrineinformatics.github.io/gemd-docs/high-level-overview ; https://ngff.openmicroscopy.org/specifications/0.5/index.html ; https://www.ga4gh.org/wp-content/uploads/2026/04/2025-annual-report-designed-3.pdf
- Assay noise: https://pmc.ncbi.nlm.nih.gov/articles/PMC10934815/ (JCIM 2024)
- Governance: https://grants.nih.gov/grants/guide/notice-files/NOT-OD-25-081.html (2025-03-28); https://grants.nih.gov/grants/guide/notice-files/NOT-OD-25-118.html (2025-05-30); https://community.ukbiobank.ac.uk/hc/en-gb/articles/31972311370013 ; https://support.researchallofus.org/hc/en-us/articles/4407354684052 ; https://www.govinfo.gov/content/pkg/FR-2025-01-07/html/2024-31542.htm ; https://health.ec.europa.eu/consultations/stakeholders-consultation-eudralex-volume-4-good-manufacturing-practice-guidelines-chapter-4-annex_en ; https://arxiv.org/abs/2210.08871 (MELLODDY); https://www.apheris.com/networks/aisb ; https://www.statnews.com/2025/03/27/abbvie-johnson-and-johnson-pharma-data-sharing-openfold3-ai-drug-discovery/ ; https://arxiv.org/abs/2602.08061 (BDL); https://www.nti.org/analysis/articles/aixbio-horizon-scan-spring-2026/
- Benchmarks: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12552120/ (CleanSplit); https://arxiv.org/abs/2308.09639 (LP-PDBBind); https://doi.org/10.1101/2025.02.03.636309 (Runs N' Poses); https://doi.org/10.1038/s42256-025-01055-1 (Matbench Discovery); https://polaris-hub.vercel.app/blog/reproducible-machine-learning-in-drug-discovery-how-polaris-serves-as-a-single-source-of-truth (2025-01-29); https://huggingface.co/datasets/ginkgo-datapoints/GDPa1
- Market: https://www.benchling.com/biotech-ai-report-2026 ; https://www.drugdiscoverynews.com/the-2026-ai-power-shift-17020 ; https://blog.kaleidoscope.bio/beyond-algorithms-software-infrastructure-for-ai-ml-biotechs/ ; https://www.bvp.com/atlas/building-biology-native-data-infrastructure-for-the-ai-era ; https://seqera.io/blog/context-gap-benchling-lineage-gxp/ (2026-04-30); https://seqera.io/blog/enterprise-release-26.1/ (2026-05-29); https://www.newswire.com/news/apprentice-io-acquires-ganymede-to-deliver-the-industrys-first-ai-native (2026-01-21); https://tiledb.com/blog/introducing-carrara (Oct 2025); https://blocksandfiles.com/2025/11/13/coreweave-zero-egress ; https://edisonscientific.com/articles/announcing-kosmos
- Job postings (aggregator extracts, 2026): Lila https://job-boards.greenhouse.io/lila/jobs/7940387002 and https://www.theladders.com/job/senior-engineer-i-ii-drug-discovery-platform-lila-sciences-cambridge-ma_88168586 ; Aqemia https://jobs.lever.co/aqemia.com/a3ef27a3-71a5-4c41-aa09-fd2954b994a6 ; CuspAI https://opportunities.northzone.com/companies/cuspai/jobs/88371735-materials-knowledge-architect ; Periodic Labs https://builtin.com/job/research-engineer-data/9259344 ; insitro https://jobs.a16z.com/jobs/insitro/ff6605ae-4961-4a06-b656-6d7dc665d990--senior-staff-software-engineer-imaging (posted 2026-08-25); Biohub https://www.builtinnyc.com/job/senior-staff-data-platform-engineer/10312623 ; Cradle https://www.indexventures.com/startup-jobs/cradle/scientific-software-engineer/ ; Asimov https://xtalks.com/job/staff-data-engineer-lab-data-platform/ ; Inductive Bio https://builtin.com/job/data-engineer/7637577 ; Isomorphic Labs https://builtin.com/job/software-engineer-data-services-london/9509879
