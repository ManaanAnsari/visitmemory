# VisitMemory — Legal & Regulatory Concerns (Malaysia)

> **Status:** Working notes / risk register. Not legal advice.
> **Scope:** Malaysia (PDPA 2010 as amended). Cross-border and expansion markets are out of scope for now.
> **Last updated:** 18 October 2026

This document captures the legal concerns identified for the VisitMemory design as currently specified in `tech.md` and `business.md`. It is intended to be handed to Malaysian privacy counsel for review. It does **not** resolve the issues; it records them.

---

## 1. Executive Summary

The current product design is built around **covert facial recognition of walk-in shoppers** who have not consented. Under Malaysian law as amended in 2024, that is the single largest legal exposure in the plan.

Key conclusions:

1. **Face data is now "sensitive personal data"** under the PDPA. Processing it requires **explicit consent** in nearly all commercial cases.
2. **Deleting data after 24 hours does not fix the problem.** The offence is the *processing* (creating the template), not the *retention*.
3. **"We only store VIPs" is not a defence.** To identify a VIP you must process the face of every visitor, including non-VIPs.
4. **There is no reliable technical trick to reclassify face templates as non-biometric.** Hashing, vectorisation, and "anonymisation" do not remove the classification while the data remains identifying.
5. **The cloud stack as designed involves cross-border transfers** of biometric data, which have their own amended compliance requirements.
6. **Additional obligations** (DPO appointment, breach notification, controller registration) are already in force and are not addressed in the current docs.
7. **An AI-specific framework is being consulted on right now** (October 2026) and will add further expectations.

The compliant paths are (a) consent-gated biometrics, (b) a non-biometric identifier, or (c) a hybrid. These are detailed in Section 8.

---

## 2. Regulatory Framework

### 2.1 Primary statute

- **Personal Data Protection Act 2010 (Act 709)** ("PDPA").
- **Personal Data Protection (Amendment) Act 2024 (Act A1727)**, Royal Assent 9 October 2024, gazetted 17 October 2024.
- Commencement (P.U. (B) 522/2024):
  - 1 January 2025 — ss. 7, 11, 13, 14
  - **1 April 2025 — ss. 2, 3, 4, 5, 8, 10, 12** (includes the biometric definition change)
  - 1 June 2025 — ss. 6, 9 (DPO + breach notification)

### 2.2 Key definitions (as amended)

- **"biometric data"** (Act 709 s.4, inserted by Act A1727 s.3):
  > "any personal data resulting from technical processing relating to the physical, physiological or behavioural characteristics of a person"
- **"sensitive personal data"** now expressly includes **biometric data**.
- Commentary from Malaysian counsel and the Ministry of Digital expressly cites **facial recognition** as a covered category.

### 2.3 Relevant principles and provisions

| Provision | Effect | Relevance to VisitMemory |
| :--- | :--- | :--- |
| s.5(1) / s.5(1a) | Data controller and data processor must comply with the 7 PDP Principles; processors now bound by the Security Principle. | Both VisitMemory and the merchant are in scope. |
| s.5(2) | Penalty for breach of the Principles: **up to RM1,000,000 and/or 3 years** (raised from RM300,000 / 2 years). | Material risk. |
| s.6(1)(b) | Sensitive personal data may only be processed **in accordance with s.40**. | Bars reliance on ordinary consent/bases. |
| s.6(2) | Non-consent bases (contract, legal obligation, vital interests, etc.) apply **only to ordinary personal data, not sensitive data**. | No fallback for face data. |
| s.7 | Notice and Choice Principle — data subject must be informed of purpose, source, rights, etc. | Covert operation fails this. |
| s.38 | Withdrawal of consent — controller must cease processing. | Need a functioning withdrawal path. |
| s.40(1) | Sensitive data may be processed only with **explicit consent**, or on narrow grounds (employment-law obligations, vital interests, medical, legal proceedings/advice, legal rights, administration of justice, functions under written law, or Minister-prescribed purposes). | **Retail clienteling is not a listed ground.** |
| s.40(3) | Penalty: **up to RM200,000 and/or 2 years**. | Direct exposure for unconsented recognition. |
| s.42 | Right to prevent processing likely to cause damage/distress. | A customer can demand cessation. |
| s.43 | Right to prevent processing for direct marketing. | Relevant to personalised outreach. |
| s.44 | Record-keeping obligation. | Consent records must be retained. |
| s.129 | Cross-border transfer — whitelist removed; controller must ensure substantially similar law / adequate protection **or** rely on s.129(3) grounds (consent is the practical one). | Cloud stack sends face data overseas. |
| s.12a (new) | Mandatory appointment of one or more **Data Protection Officers**; notify the Commissioner. | Applies to controller and processor. |
| s.12b (new) | Mandatory **personal data breach notification** to the Commissioner, and to affected subjects where significant harm. | Incident response required. |
| Registration regime | Data controllers must register with JPDP (active compliance push in 2026). | Merchant and/or VisitMemory may need to register. |

> **Note on "legitimate interest":** Malaysia did **not** introduce a GDPR-style legitimate-interest basis in the 2024 amendment. There is therefore no legitimate-interest route to unconsented face recognition.

---

## 3. Core Concern: Unconsented Recognition

### 3.1 The design as written

The current architecture (`tech.md` §3, §4, §5) does the following:

1. Samples the entrance camera at ~2 FPS.
2. Runs SCRFD face detection and extracts a face crop.
3. Transmits a ~50 KB crop to the cloud.
4. Performs a vector search (AWS Rekognition / Qdrant) against the customer database.
5. Creates a **provisional profile** for unrecognised faces (`is_vip = false`, `display_name = null`).
6. Purges untagged provisional profiles after 24 hours.

### 3.2 Why this is a problem

- Step 2–4 is **technical processing of facial characteristics** → creation of **biometric data** → **sensitive personal data**.
- The subject has **not given explicit consent** (there is no enrollment step in the flow; the Hospitality Protocol in `business.md` §7.2 actively instructs staff *not* to mention the camera).
- None of the s.40 exceptions apply to retail clienteling.
- Therefore each walked-in visitor whose face is templated is a likely **s.40 contravention**, regardless of whether the profile is later deleted.

### 3.3 The two proposed mitigations do not work

**"We delete in 24 hours."**
The 24-hour purge addresses the **Retention Principle**. It does not address the **lawful basis** for processing. The sensitive data is created at the moment of templating; deletion afterwards does not retroactively create consent. It may reduce the severity/scope but not the existence of the breach.

**"We only store VIPs."**
False in practice. Determining whether a face belongs to a VIP requires processing the face of every entrant. Non-VIP visitors are templated and matched (and, in the current design, stored provisionally) before being discarded. The law regulates *processing*, not just *storage*.

---

## 4. Concern: "Can we reclassify it as non-biometric?"

Recorded here so the question is not re-litigated. The following do **not** remove the biometric/sensitive classification:

- **Hashing / encryption of the template** — still derived from facial characteristics; still biometric data.
- **Storing only the 512-d vector, not the image** — the vector *is* the biometric data ("resulting from technical processing").
- **On-device-only processing with no transmission** — still processing; still sensitive.
- **"Anonymisation" / de-identification** — face templates are at best pseudonymous and inherently linkable/re-identifiable. Once linked to a name or purchase, clearly personal data.
- **Calling it a generated "customer ID"** — substance over label. Misclassification is an aggravating factor, not a defence.
- **Aggregating counts / heatmaps only** — would genuinely avoid the issue, but that is a different product (no clienteling).

A defensible "not personal data" position requires that individuals are genuinely **not identifiable** from the data. That condition is incompatible with the product's purpose.

---

## 5. Concern: Cross-Border Data Transfer

The specified stack sends face crops and embeddings to services that are not guaranteed to be Malaysia-resident:

- **AWS Rekognition** (Phase 1) — likely US processing.
- **Cloudflare R2** — no guaranteed Malaysian region; global replication.
- **Supabase / managed PostgreSQL** — region-dependent.
- **Redis / Celery workers** — region-dependent.

Under amended **s.129**:

- The whitelist has been removed; the controller must itself determine that the destination has substantially similar law or adequate protection.
- Alternatively rely on a s.129(3) ground — for biometric data, **explicit consent to the transfer** is the practical route.
- JPDP's Cross-Border Personal Data Transfer Guidelines require a **Transfer Impact Assessment (TIA)**, reassessed at least every 3 years, and notice to the data subject.

**Implication:** the `$4.18/month` COGS in `business.md` may need to change to host in a Malaysian region or self-host (the Phase 2 ArcFace + Qdrant plan is legally preferable if hosted in Malaysia).

---

## 6. Concern: Additional Compliance Obligations Not Currently Addressed

1. **DPO appointment** (s.12a) — required for both controller and processor, with notification to the Commissioner. The docs do not mention staffing for this.
2. **Breach notification** (s.12b) — a defined process is required; biometric breach is especially sensitive.
3. **Data controller registration** with JPDP — verify which entity must register and in which class.
4. **DPIAs** — a Data Protection Impact Assessment should be produced for the AI/biometric system (also expected by the incoming AI framework).
5. **Employee biometrics** — the "staff ignore list" (`tech.md` §10, Caveat 2) processes employees' face data. Employment-law ground may partially apply, but explicit consent / a documented basis is still needed.
6. **Automated decision-making / profiling** — the "Jev" decision engine (`tech.md` §8) profiles customers and drives staff actions; an ADMP guideline has been consulted on and should be reviewed.
7. **Withdrawal & access rights** — s.38/s.42/s.43 and access/correction/portability requests must have an operational path.
8. **Data processor agreement** — the merchant (controller) and VisitMemory (processor) need a written processing agreement, including Security Principle obligations.

---

## 7. Concern: Incoming AI Framework (Active Consultation)

- JPDP **Public Consultation Paper No. 1/2026 — Artificial Intelligence (AI) and Personal Data Protection Framework**.
- Consultation open **5 October 2026 to 23 October 2026**.
- Key propositions relevant to VisitMemory:
  - **AI does not create a separate legal basis** for processing.
  - Must identify and document the legal basis; **sensitive personal data still requires consent**.
  - Fresh consent where data is used for a materially different purpose.
  - Data Protection by Design; testing/validation; monitoring.
  - Third-party AI due diligence and cross-border safeguards.
  - DPIAs; governance; documentation and audit; demonstrable compliance.
- VisitMemory is squarely in scope (an AI system embedded in a SaaS platform processing personal data).
- **Action:** consider submitting feedback before 23 October 2026 and tracking the final framework.

---

## 8. Possible Compliant Paths (For Decision Later)

Recorded for future scoping; not yet chosen.

### Option A — Consent-gated biometrics
- Enroll customers with **explicit consent** at a real touchpoint (signage + explicit opt-in), for a specified purpose.
- Record and retain consent evidence (s.44).
- Provide withdrawal (s.38) and honour it.
- Only recognition of opted-in customers is permissible; matching non-enrolled visitors remains a problem and needs counsel review.
- Prominent in-store notice; abandon the "never mention the camera" protocol.

### Option B — Non-biometric identifier
- Replace face recognition with a QR code, loyalty card, app tap, BLE proximity, or staff-entered phone/name.
- **s.40 does not apply** because no biometric data is processed.
- Preserves the entire POS-spooler, receipt-OCR, and messaging backend.
- Loses the "automatic alert when a stranger walks in" capability.

### Option C — Hybrid
- Biometrics only for enrolled, consenting VIPs.
- No FRT on the general floor / no templating of non-enrolled visitors.
- Non-biometric identifiers for everyone else.

### Option D — Malaysia-resident data path
- Host biometric data in a Malaysian region or self-host (ArcFace + Qdrant) in Malaysia.
- Removes most s.129 friction (still keep a TIA).
- Likely raises COGS above the current `$4.18/month` assumption.

---

## 9. Open Questions for Malaysian Counsel

1. Is any transient, on-device, zero-retention template extraction (used only to check for enrolled matches) defensible without explicit consent, or is it still s.40 processing?
2. Does the merchant (retailer) or VisitMemory (vendor) — or both — act as data controller for each data flow? Does multi-branch "VIP roaming" make VisitMemory a controller/joint controller?
3. What is the minimum valid "explicit consent" mechanism in a physical store (signage + QR opt-in, digital signature, etc.)?
4. Which entity must register with JPDP, and in which data-controller class?
5. What does a compliant TIA look like for AWS Rekognition / Cloudflare / Supabase, and is Malaysia-resident hosting required?
6. How should employee face data (staff ignore list) be handled?
7. Does the Jev decision engine trigger ADMP obligations, and what transparency is required?
8. What is the enforcement posture of JPDP toward retail FRT in practice (compounds, investigations)?
9. What is the exposure for the "Hospitality Protocol" instruction to conceal the use of FRT (misrepresentation / notice failure)?
10. Is there any sector-specific guidance for CCTV/biometrics in retail beyond the PDPA?

---

## 10. Sources

- Personal Data Protection (Amendment) Act 2024 (Act A1727):
  `https://www.pdp.gov.my/ppdpv1/wp-content/uploads/2024/10/Act-A1727.pdf`
- Consolidated Act 709 (2026 reprint, English):
  `https://www.pdp.gov.my/ppdpv1/wp-content/uploads/2026/10/Akta-709-cetakan-semula-2026-BI.pdf`
- JPDP — Principles of Personal Data Protection:
  `https://www.pdp.gov.my/ppdpv1/en/principles-of-personal-data-protection/`
- JPDP — Cross Border Personal Data Transfer Guidelines (3/2025):
  `https://www.pdp.gov.my/ppdpv1/wp-content/uploads/2025/08/GP_CBPDT_EN-1.pdf`
- JPDP — Public Consultation Paper No. 1/2026, AI and Personal Data Protection Framework:
  `https://www.pdp.gov.my/ppdpv1/wp-content/uploads/2026/10/Public_Consultation_Paper_No_1_2026_Personal_Data_Protection_Framework.pdf`
- Christopher & Lee Ong — Analysis of the PDP Amendment Bill 2024 (biometric data as sensitive personal data):
  `https://www.christopherleeong.com/viewpoints/the-personal-data-protection-amendment-bill-2024-an-analysis-and-upcoming-developments/`
- Baker McKenzie — Malaysia consultation on proposed AI and PDP Framework (Oct 2026):
  `https://www.bakermckenzie.com/en/insight/publications/2026/10/malaysia-consultation-on-proposed-ai-and-personal-data-protection-framework`

---

## 11. Disclaimer

This is an engineering and business risk inventory compiled from public sources. It is **not legal advice** and does not create a lawyer-client relationship. No deployment decision should be made without a written opinion from qualified Malaysian privacy counsel.
