# Controlled data: CUI, ITAR and EAR, and NIST requirements

Research notes, November 2026, in support of `docs/controlled-data.md` in the mudcat repository. This is not legal advice. The primary texts (eCFR, NIST CSRC, the NARA CUI Registry, the DoD memo) could not be fetched from the research environment, so everything below is from search extracts. Each item states what kind of source it rests on.

## CUI

- **Authority.** 32 CFR Part 2002 and the CUI Registry of the National Archives (NARA).
- **Markings (32 CFR 2002.20, Cornell LII extract; FAS copy of the marking guide).** The Registry's markings are the only authorised markings for unclassified information that needs safeguarding or dissemination controls. A banner combines the CUI control marking, category and subcategory markings, and limited dissemination control markings. Dissemination markings are set off by a double slash; subcategories by a single slash.
- **Limited dissemination controls (NARA Registry page).** Only the designating agency may apply them; they can be combined; using them to restrict access unnecessarily runs against the programme's goals. `FED ONLY` supersedes other controls but cannot override dissemination required by law. `NOFORN`, `FED ONLY` and `FEDCON` are examples (DCSA, DoD CUI Marking Guide, December 2024).
- **Export Controlled is a CUI category (NARA Registry page).** Banner `CUI//SP-EXPT` for specified authorities, and plain `CUI` or `CUI//EXPT` for basic ones. Listed authorities include 15 CFR 748.1(c) and 22 CFR 120.21 (specified) and 22 CFR 124.9(a)(5) (basic).
- **Not found:** the decontrol provisions (32 CFR 2002.18) and any official text on whether file names, search indexes or other metadata of CUI are themselves CUI.

## NIST SP 800-171 and CMMC

Sources: secondary compliance blogs, which agree with each other; verify against 32 CFR 170, the DoD class deviation and NIST CSRC.

- **Revision 2:** 110 requirements in 14 families. **Revision 3** (May 2024): 97 requirements in 17 families, with organisation-defined parameters and no basic/derived split. The assessment guide 800-171A grows from 320 to 422 determination statements.
- **CMMC Level 2 is still assessed against Revision 2.** 32 CFR 170.14 states that Level 2 requirements are identical to Revision 2, and DoD class deviation 2024-O0013 (2 May 2024) ties DFARS 252.204-7012 to Revision 2, with no end date.
- **2026 developments are unclear.** One source reports a June 2026 proposed FAR rule that would require Revision 3 government-wide; another, from mid-September 2026, says it had not appeared in the Federal Register. A claim that third-party assessments were suspended from July 2026, and a reference to a class deviation 2026-O0025, could not be confirmed.
- **Validated cryptography (Revision 2, 3.13.11).** FIPS-validated cryptography is required when cryptography protects the confidentiality of CUI. "Using AES-256 is not the same as using a FIPS-validated AES-256 implementation." A module counts only when it is on NIST's list of validated modules. One source says this control tops the DoD assessment centre's list of most often unmet controls.
- **FIPS 140-2 to 140-3.** A secondary source (Wikipedia, through search) reports that FIPS 140-2 validations move to the historical list on 21 September 2026.

## DFARS 252.204-7012

Sources: Cornell LII and eCFR listings, vendor pages (AWS, Google, Microsoft), law-firm summaries (Crowell).

- A contractor that uses an external cloud service provider for covered defense information must ensure that the provider "meets security requirements equivalent to the FedRAMP Moderate baseline".
- A DoD memo defines equivalence for offerings without a FedRAMP authorisation: 100% compliance with the Moderate baseline, validated by a FedRAMP-recognised third-party assessor. A FedRAMP Moderate or High authorisation is sufficient, but only for services inside the authorised boundary.
- Cyber incidents that affect covered defense information must be reported to the DoD within 72 hours. The contractor is responsible for the provider's compliance with the reporting paragraphs.

## ITAR

- **Encryption carve-out, 22 CFR 120.54(a)(5)** (govinfo extract of the 2025 CFR; Federal Register 26 December 2019; effective March 2020). Sending, taking or storing unclassified technical data is not an export when it is secured with end-to-end encryption, "using cryptographic modules (hardware or software) compliant with the Federal Information Processing Standards Publication 140-2 (FIPS 140-2) or its successors", at a strength comparable to AES-128 or more, and "not intentionally sent to a person in or stored in a country proscribed" in §126.1 or Russia.
- **Paragraph (b):** end-to-end means the means of decryption are not provided to any third party. (An older edition also lists who the intended recipient may be: the originator, a US person in the United States, or a person otherwise authorised.)
- **Paragraph (c):** "the ability to access technical data in encrypted form that satisfies the criteria set forth in paragraph (a)(5) of this section does not constitute the release or export of such technical data."
- **Commentary (vendor and law-firm sources):** server-side encryption where the provider holds the keys does not qualify; tokenisation is not covered.
- **Deemed exports.** The searches did not return the text of §120.50, §120.55 (access information) or §120.56 (release). The statement that giving a foreign person access to technical data is an export even inside the United States is from general knowledge of the regulation, not from a source read in this session.

## EAR and AI model weights

Sources: law-firm summaries from early 2025; not checked for later changes.

- The January 2025 "AI Diffusion" rule of BIS created ECCN 4E091 for unpublished weights of models trained with 10^26 or more operations. Published (open) weights are not controlled under it.
- Commentary on models trained on controlled data: weights "inherit a sensitivity presumption from their training corpus"; whether weights "reveal" ITAR technical data "is fact-specific and turns on the extent to which the training data is recoverable from the weights"; one pipeline guide recommends that "derived datasets inherit the most restrictive tag of any input". No official DoD or NARA text was found that states this for CUI.

## University enclaves

Sources: university web pages and two arXiv papers found by search.

- Northwestern offers an HPC option hosted at NCSA (HIPAA and NIST SP 800-171) and a Google Cloud enclave for smaller workloads.
- Colorado State's enclave is hosted at the San Diego Supercomputer Center and uses 800-171 controls for FAR 52.204-21 and DFARS 252.204-7012; access needs a qualifying project and a background check.
- Utah's Citadel is aligned with CMMC 2.0 Level 2, with access per project.
- NC State's SURE is a virtual workspace with no HPC.
- Purdue's earlier REED environment ran on AWS GovCloud, which "restricts access to users within the geographical boundaries of the United States and can support ITAR-regulated data"; Purdue later built a local HPC system (Weber) because of cost and integration drawbacks of the cloud approach.
- Utah State estimates about $2,500 per user seat.
- A 2018 NIST workshop case study from Florida applied the Risk Management Framework to CUI on HPC.

No source confirmed a US-persons-only rule for these enclaves in general; that restriction follows from the export-control determination of each project.

## NIST SP 800-171 Revision 2 requirements that fall on the software

A search confirmed the numbering of these requirements through a university CMMC mapping, a withdrawn NIST handbook and the DoD Level 2 assessment guide. The short descriptions are paraphrases, and some come from memory of the publication. Check the wording in the Revision 2 text before use.

| Number | Subject |
|---|---|
| 3.1.1, 3.1.2, 3.1.5 | Limit access to authorised users and to permitted functions; least privilege |
| 3.1.3 | Control the flow of CUI in accordance with approved authorisations |
| 3.1.4 | Separation of duties |
| 3.1.9 | Privacy and security notices consistent with CUI rules |
| 3.1.10, 3.1.11 | Session lock and session termination |
| 3.1.22 | Control CUI posted or processed on publicly accessible systems |
| 3.3.1, 3.3.2 | Create and retain audit logs; trace actions to individual users |
| 3.3.4 | Alert in the event of an audit logging process failure |
| 3.3.8, 3.3.9 | Protect audit information; limit management of audit logging to a subset of privileged users |
| 3.4.6, 3.4.7 | Least functionality; restrict nonessential functions and services |
| 3.5.1 to 3.5.3 | Identify and authenticate users; multifactor authentication |
| 3.8.3, 3.8.4, 3.8.9 | Sanitise media; mark media; protect backups |
| 3.13.4 | Prevent unauthorised and unintended information transfer through shared system resources |
| 3.13.8, 3.13.16 | Cryptography in transit; confidentiality of CUI at rest |
| 3.13.11 | FIPS-validated cryptography |
| 3.14.1 | Flaw remediation |

A catalog that holds metadata about CUI is itself an asset in the scope of the assessment, even when the data bytes are on other storage: it holds names, descriptions, statistics, indexes, the audit log, backups, and the credentials that give access to the bytes.

Source for the numbering: [University of Hawaii CMMC levels mapping](https://hawaii.edu/infosec/minimum-standards/cmmc/levels-1-3), [NIST SP 800-171 Revision 2](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-171r2.pdf).

## What this means for a catalog

1. The catalog holds controlled information itself: names, descriptions, fields, statistics, embeddings, previews, the search index, the change stream, logs and backups.
2. Access is the controlled event: who the person is and where they read from, not only where the bytes are.
3. Derived data keeps the control, and a model trained on the data is controlled until an authority decides otherwise.
4. Operators of a service are persons with access.
5. Every external service is a possible release: enrichers that call model services, external search engines, public descriptions, federation feeds, agents.
6. Evidence is required: a record of each access, and the scope of an incident within 72 hours for defence contractors.
7. Compliance belongs to an organisation's system. A product can supply functions and evidence for access control, audit, media protection, protection of communications and integrity, but not for personnel, physical or training requirements.
8. The ITAR carve-out maps to custody where the service holds only ciphertext and the customer holds the keys, at the cost of search and computation on that content.

## Sources

- [NARA CUI Registry: limited dissemination controls](https://www.archives.gov/cui/registry/limited-dissemination)
- [NARA CUI category: Export Controlled](https://www.archives.gov/cui/registry/category-detail/export-control)
- [32 CFR 2002.20](https://www.law.cornell.edu/cfr/text/32/2002.20)
- [DoD CUI Marking Guide, December 2024](https://www.dcsa.mil/Portals/128/Documents/CTP/CUI/DoD%20CUI%20Marking%20Guide%20DEC%202024.pdf)
- [NIST SP 800-171 Revision 3](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-171r3.pdf)
- [NIST 800-171 Rev 2 vs Rev 3 (Secureframe)](https://secureframe.com/blog/nist-800-171-rev2-vs-rev3)
- [NIST 800-171 Rev 2 vs Rev 3 (The Defense Compliance Report)](https://thedefensecompliancereport.com/nist-800-171-rev-2-vs-rev-3/)
- [DFARS 252.204-7012](https://www.law.cornell.edu/cfr/text/48/252.204-7012)
- [Crowell: DoD guidance on FedRAMP Moderate equivalency](https://crowell.com/en/insights/client-alerts/no-longer-cloudy-dod-issues-new-guidance-on-fedramp-moderate-equivalency-cloud-security-requirements)
- [22 CFR 120.54, 2025 edition](https://www.govinfo.gov/content/pkg/CFR-2025-title22-vol1/pdf/CFR-2025-title22-vol1-sec120-54.pdf)
- [Federal Register, 26 December 2019: activities that are not exports](https://www.govinfo.gov/content/pkg/FR-2019-12-26/html/2019-27438.htm)
- [Wiley: the encryption carve-out](https://wiley.law/alert-US-State-Department-Opens-the-Door-to-Use-Encryption-Carve-Out-to-Send-and-Store-ITAR-Data)
- [WilmerHale: BIS export controls on AI](https://www.wilmerhale.com/en/insights/publications/20250205-bis-issues-long-awaited-export-controls-on-ai)
- [Orrick: controls on AI model weights](https://www.orrick.com/en/Insights/2025/01/New-Export-Controls-on-Advanced-Computing-Items-and-AI-Model-Weights)
- [Northwestern: options for regulated research](https://www.it.northwestern.edu/departments/it-services-support/research/data-storage/available-options-for-regulated-research.html)
- [Utah CHPC: Citadel](https://chpc.utah.edu/resources/citadel.php)
- [NC State: Secure University Research Environment](https://research.ncsu.edu/administration/compliance/research-compliance/export-controls/cui/secure-university-research-environment-sure)
- [Purdue: from cloud-based to locally sited secured computing](https://ar5iv.labs.arxiv.org/html/2108.03994)
- [NIST workshop: RMF for CUI on HPC](https://www.nist.gov/document/erik-rmf-cui-hpc-lessons-deumenspdf)
