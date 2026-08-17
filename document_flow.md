# Build Blueprint: AI-Based Early Detection and Prevention of Drug Allergic Reactions

## TL;DR
- **Build a deterministic, data-driven rule engine as the core**, wrap it in a hard-coded anaphylaxis red-flag layer, and use a small local LLM (Llama 3.1 8B / Qwen 2.5 7B via Ollama) ONLY to translate the engine's structured output into plain-language explanations. The engine's clinical logic should be a simplified, consumer-facing adaptation of the Naranjo ADR Probability Scale and WHO-UMC causality categories — this is your strongest academic defensibility hook.
- **Your knowledge base is the product.** A ~40-drug JSON/SQLite KB (RxNorm code + reaction patterns with onset windows + severity flags + cross-reactivity class links), curated ONCE via a Python pipeline from openFDA `drug/label.json`, FAERS `drug/event.json`, and RxNav/RxClass, makes the "add a drug = add a data record, not code" scalability constraint literally true.
- **Uniqueness comes from temporal-plausibility reasoning + proactive cross-reactivity prevention**, not from another symptom logger. Existing consumer allergy apps (Allergy Tracker, MyTherapy, Zyrtec AllergyCast) only log symptoms/pollen; none reason over drug-intake timing vs. onset windows or warn about penicillin→cephalosporin cross-reactivity when a new drug is logged.

## Key Findings

1. **All three core data sources are free, keyless (or free-key), REST/JSON, and well-suited to one-time curation.** openFDA needs no key (per open.fda.gov's authentication page, no key gives 240 requests/minute per IP but is capped at **1,000 requests/day**; a free key raises the daily cap to 120,000/day). DailyMed and RxNav need no authentication at all.
2. **The clinical literature gives you clean, citable numbers for the rule engine**: immediate reactions <1–6 h (majority within 1 h), delayed >6 h to days/weeks; penicillin→cephalosporin cross-reactivity ~1% (side-chain driven, not the debunked "10% rule"); sulfonamide antibiotic→non-antibiotic cross-reactivity is not established (shared-predisposition, not structural); carbamazepine SJS/TEN onset 4–28 days.
3. **Anaphylaxis red-flags are formally specified** by NIAID/FAAN (2006) and WAO (2020) criteria — you can hard-code these directly.
4. **The Naranjo scale and WHO-UMC system are ready-made scoring skeletons.** Naranjo already weights temporal sequence (+2), prior conclusive reports (+1), and alternative causes (−1) — you adapt these into a consumer-facing score.
5. **This exact architecture (rule engine + LLM explanation, patient-facing) is genuinely novel.** Existing systems are clinician-facing CPOE/CDSS (South Korea's K-CDS; HELIOT); patient-facing tools are symptom loggers. Honest novelty claim: *consumer-facing temporal + cross-reactivity reasoning with LLM explanation.*

## Details

### 1. Data Source Access (verified endpoints)

**openFDA — FAERS adverse events** (`https://api.fda.gov/drug/event.json`)
- No key required. Per openFDA's official authentication page: with no API key you get 240 requests/minute and **1,000 requests/day** per IP; with a free key (register at https://open.fda.gov/apis/authentication/) you get 240/minute and **120,000/day** per key. Max `limit=1000` per call. The daily cap (not a per-minute throttle) is the real constraint for bulk curation — get a free key.
- Example — count reaction terms for amoxicillin:
  `https://api.fda.gov/drug/event.json?search=patient.drug.medicinalproduct:"amoxicillin"&count=patient.reaction.reactionmeddrapt.exact`
- This `count` pattern is the key trick: it returns a ranked frequency table of MedDRA reaction terms — exactly what you need to populate "documented reaction patterns" per drug, empirically weighted.

**openFDA — Drug labels / SPL** (`https://api.fda.gov/drug/label.json`)
- openFDA pre-parses SPL XML into named JSON arrays. The fields you need: `adverse_reactions`, `boxed_warning`, `warnings`, `warnings_and_cautions` (note the exact openFDA field name is `warnings_and_cautions`, NOT `warnings_and_precautions`), `contraindications`. Companion `_table` fields hold HTML tables. Nested `openfda` object gives `rxcui`, `generic_name`, `brand_name`, `unii`, `product_ndc`.
- Example: `https://api.fda.gov/drug/label.json?search=openfda.generic_name:"amoxicillin"&limit=1`
- Parse both `warnings` (older label format) and `warnings_and_cautions` (current PLR format) to cover both.

**DailyMed v2** (`https://dailymed.nlm.nih.gov/dailymed/services/v2/`) — no auth, GET only
- Discover SETID by name: `/spls.json?drug_name=amoxicillin&name_type=generic` (also supports `boxed_warning=true` filter, `rxcui=`, `pagesize` max 100). Returns `data[]` with `setid`, `title`, `published_date`, plus pagination metadata.
- Full label: `/spls/{SETID}.xml` (XML only — HL7 v3 SPL, root `<document>`). Inside, each `<section>` carries a LOINC `<code>` (codeSystem OID `2.16.840.1.113883.6.1`). **Verified LOINC section codes:** Boxed Warning **34066-1**, Adverse Reactions **34084-4**, Warnings (old format) **34071-1**, Warnings & Precautions (PLR format) **43685-7**, Contraindications **34070-3**. (Verified against FDA's LOINC-for-labeling document at fda.gov/media/114007/download and the openFDA loinc.csv lookup table.)
- Recommended workflow: `/spls.json?drug_name=X` → grab `setid` → `/spls/{setid}.xml`.
- **Recommendation:** use openFDA `drug/label.json` for parsing (named JSON fields, no XPath); use DailyMed as the authoritative ground-truth source and for version history. openFDA lags DailyMed by ~1 week.

**RxNorm / RxNav** (`https://rxnav.nlm.nih.gov/REST/`) — no auth
- Normalize free-text drug name → RxCUI: `/rxcui.json?name=amoxicillin` (findRxcuiByString; `search=2` for exact-or-normalized; normalized search expands abbreviations, e.g. "hctz"→hydrochlorothiazide, and ignores salt forms). For typos, `/approximateTerm.json?term=amoxicilin&maxEntries=4`.
- Drug → classes (for cross-reactivity): `/rxclass/class/byRxcui.json?rxcui={RXCUI}&relaSource=ATC` (getClassByRxNormDrugId) and RxClass `getClassByRxNormDrugName`. ATC groups beta-lactams, sulfonamides, NSAIDs, anticonvulsants — use ATC class membership to drive cross-reactivity linking, then override with the curated side-chain rules below. `getClassMembers` enumerates all drugs in a class.

### 2. Clinical Grounding for the Rule Engine

**Onset-time windows (Gell-Coombs / ICON classification):**
- **Immediate (Type I, IgE-mediated):** symptoms within 1 h, up to 6 h (majority within the first hour). Presentations: urticaria, angioedema, bronchospasm, anaphylaxis. Culprits: NSAIDs, beta-lactams, NMBAs.
- **Delayed / non-immediate (Type IV, T-cell):** >6 h, typically days to weeks. Presentations: maculopapular exanthem (MPE), fixed drug eruption, and severe cutaneous adverse reactions (SJS/TEN, DRESS, AGEP). Contact/delayed peak commonly 24–72 h.

**Drug-class specifics:**
- **Aminopenicillins (amoxicillin/ampicillin):** delayed MPE typically appears days into a course — one study documented rashes 7–20 days (median 8 days) after starting amoxicillin; extended-challenge studies found delayed reactions at a mean of ~6 days. In EBV/infectious mononucleosis, the rash rate is elevated: Chovel-Sella et al. (238 children, *Pediatrics*) found a **29.5%** rash incidence with amoxicillin vs 23% without antibiotics, and a 2025 systematic review/meta-analysis (*Eur J Clin Microbiol Infect Dis*, 15 studies/3,153 patients) reported a **pooled estimate of 43% (95% CI 18–72%)** after aminopenicillins vs 14% with no antibiotic — mostly NOT true allergy. This is the flagship demo scenario.
- **Sulfonamide antibiotics (TMP-SMX):** immediate IgE reactions plus notable delayed SCARs (SJS/TEN, DRESS). ~3% of TMP-SMX recipients react.
- **NSAIDs:** cross-reactive phenotypes are pharmacological (COX-1 inhibition), not IgE — NERD, NECD, NIUA. Cross-reactivity applies across chemically unrelated COX-1 inhibitors; COX-2 selective (celecoxib) and acetaminophen usually tolerated.
- **Aromatic anticonvulsants (carbamazepine, lamotrigine, phenytoin):** SJS/TEN onset 4–28 days (often within the first 8 weeks). HLA-B*15:02 (Asian) and HLA-A*31:01 (European) risk alleles. HLA-B*15:02 carriers in Asian populations who receive carbamazepine have approximately a **5–10% risk of SJS/TEN vs <0.1% in non-carriers** (meta-analytic OR 79.84, 95% CI 28.45–224.06; Tangamornsuksan et al., PMID 23884208).

**Cross-reactivity relationships to encode:**
- **Penicillin → cephalosporin: ~1% overall** (first-gen or shared R1 side chain), <1% for 3rd/4th gen. Per Campagna et al., *J Emerg Med* 2012 (PMID 21742459): "the overall cross-reactivity rate is approximately 1% when using first-generation cephalosporins or cephalosporins with similar R1 side chains," while "a single study reported the prevalence of cross reactivity with cefadroxil as high as 27%." The "10% myth" came from 1960s–70s contaminated preparations. Encode side-chain matches: amoxicillin/ampicillin share R1 with cefadroxil, cefprozil, cephalexin, cefaclor (higher risk — a Spanish cohort found amoxicillin→cefadroxil cross-reactivity of **35.2%** among 54 confirmed amoxicillin-allergic patients vs only **1.8%** for cefuroxime, which has a dissimilar R1); dissimilar side chains = negligible.
- **Sulfonamide antibiotic → non-antibiotic sulfonamide (furosemide, acetazolamide, celecoxib, thiazides): NO established cross-reactivity.** Strom BL et al., *NEJM* 2003;349(17):1628–1635 (PMID 14573734): "this association appears to be due to a predisposition to allergic reactions rather than to cross-reactivity with sulfonamide-based drugs"; notably "a history of penicillin allergy is at least as strong a risk factor." Endorsed by the AAAAI 2022 Drug Allergy Practice Parameter. Encode as a "predisposition flag," NOT a hard cross-reaction.
- **NSAID → NSAID:** cross-reactivity across COX-1 inhibitors for pharmacological phenotypes.

**Anaphylaxis red-flags (NIAID/FAAN + WAO 2020):**
NIAID/FAAN (2006): anaphylaxis likely if any ONE of 3 criteria. WAO 2020 simplified to 2 criteria and added isolated respiratory/cardiovascular compromise after a known/probable allergen. Hard-code these symptom triggers (see Emergency Layer below).

### 3. Scoring Frameworks to Borrow

**Naranjo ADR Probability Scale** — 10 questions, scored, categorized: ≥9 definite, 5–8 probable, 1–4 possible, ≤0 doubtful. Directly relevant weighted items:
- Previous conclusive reports on this reaction: +1
- Event appeared after drug administered (temporal): +2
- Improved on dechallenge: +1
- Reappeared on rechallenge: +2
- Alternative causes present: −1

**WHO-UMC categories** — Certain / Probable / Possible / Unlikely, based on: plausible time relationship, whether explainable by disease/other drugs, dechallenge response. Qualitative but maps cleanly to a temporal + alternative-cause reasoning structure.

Your consumer adaptation keeps the *defensible skeleton* (temporal weight highest, prior history, alternative causes, known reaction pattern) while dropping clinician-only items (blood levels, placebo, rechallenge — unethical/unavailable to consumers).

### 4. Recommended MVP Feature List

**BUILD (Phase 1 MVP):**
1. Guided symptom checklist (grouped: skin, respiratory, GI, cardiovascular/systemic) + optional free-text.
2. Medication logging: drug name (RxNorm-autocompleted), dose, date/time of intake.
3. Prior allergy profile: known drug allergies + reaction type.
4. **Emergency red-flag screen** (runs first, on every symptom submission).
5. **Deterministic rule engine** → Low/Moderate/High risk with a transparent factor breakdown.
6. **LLM explanation layer** (local Ollama) — plain-language explanation + safety recommendations, grounded strictly in the engine's structured output.
7. **Prevention check:** when a new drug is logged, check against stored allergy profile + cross-reactive classes; warn proactively.
8. Doctor-shareable summary report (printable/PDF).
9. ~40-drug curated KB (beta-lactams, sulfonamides, NSAIDs, anticonvulsants).

**CUT (defer to Phase 2):** SHAP/LIME ML explainability, EHR/FHIR integration, multi-language, photo-based rash analysis, native mobile app, live API calls at runtime, user accounts/cloud sync (use local storage for MVP).

### 5. System Architecture & Data Flow

```
[React 18/Vite mobile-responsive UI]
   │  (symptom checklist + free text, med history, allergy profile)
   ▼
[FastAPI backend]  ── input validation / RxNorm normalization
   ▼
① EMERGENCY RED-FLAG LAYER  ── hard-coded, deterministic
   │  if anaphylaxis pattern → URGENT alert, BYPASS everything else
   ▼ (no red flag)
② RULE ENGINE  ── deterministic scoring over the KB
   │  temporal plausibility + symptom-pattern match + prior history
   │  + cross-reactivity  →  {risk: Low/Mod/High, factors:[…], drug, pattern}
   ▼
③ LLM EXPLANATION LAYER (Ollama, local)  ── input = structured engine output ONLY
   │  constrained/structured prompt → plain-language explanation + recs
   ▼
④ REPORT GENERATOR  ── doctor-shareable summary
   ▼
[Back to UI]
```

Parallel flow — **Prevention loop:** logging a new medication → normalize to RxCUI → look up class + cross-reactive classes → compare to allergy profile → proactive warning card (does not need the symptom engine).

### 6. Tech Stack Recommendation

| Layer | Choice | Justification |
|---|---|---|
| Frontend | **React 18 + Vite** | Team's known stack; mobile-responsive web is right for a capstone (no app-store friction). |
| Backend | **Python + FastAPI** | The rule engine, KB curation, and Ollama calls are all Python. FastAPI gives typed Pydantic models (ideal for the structured engine→LLM contract) and auto OpenAPI docs. Node is fine for the API shell, but keeping engine + LLM + pipeline in one Python service is simpler for a small team. |
| Knowledge base | **SQLite** (with a JSON column per drug) | Zero-config, file-based, version-controllable, "add a drug = INSERT a row." Postgres is overkill for ~40 records; plain JSON files also acceptable but SQLite gives querying for free. |
| LLM | **Ollama + Llama 3.1 8B or Qwen 2.5 7B**, local | Free, private (health data never leaves the machine — a real ethics point for the report), runs on a laptop/lab GPU. Both have excellent JSON/instruction-following. Ollama's `format` parameter (JSON-schema constrained decoding, ≥v0.3.0) mechanically prevents malformed output. Fallback: free/cheap API tier (e.g., Groq free tier) if no local GPU. |
| Data pipeline | **Python (requests + pandas)**, run once | Produces the SQLite/JSON KB as a build artifact committed to the repo. |

**Why local LLM over API:** privacy of health data, $0 cost, reproducibility for the academic report, and the explanation task is easy enough for an 8B model when fully grounded.

### 7. Knowledge Base Schema (example: amoxicillin)

```json
{
  "rxcui": "723",
  "drug_name": "amoxicillin",
  "display_name": "Amoxicillin",
  "drug_class": "aminopenicillin",
  "atc_class": "J01CA04",
  "class_tags": ["beta-lactam", "penicillin", "aminopenicillin"],
  "reaction_patterns": [
    {
      "pattern_id": "amox_immediate_urticaria",
      "type": "immediate",
      "gell_coombs": "I",
      "onset_min_hours": 0,
      "onset_max_hours": 6,
      "symptoms": ["urticaria", "angioedema", "pruritus", "wheeze"],
      "severity": "moderate",
      "source": "SPL adverse_reactions / FAERS"
    },
    {
      "pattern_id": "amox_delayed_mpe",
      "type": "delayed",
      "gell_coombs": "IV",
      "onset_min_hours": 24,
      "onset_max_hours": 480,
      "typical_onset_days": 8,
      "symptoms": ["maculopapular_rash", "morbilliform_rash"],
      "severity": "low_to_moderate",
      "notes": "Common; often non-allergic, esp. with concurrent EBV.",
      "source": "PMID 12818909 / extended-challenge studies"
    },
    {
      "pattern_id": "amox_scar_dress",
      "type": "delayed",
      "gell_coombs": "IV",
      "onset_min_hours": 168,
      "onset_max_hours": 1008,
      "symptoms": ["rash", "fever", "facial_swelling", "lymphadenopathy"],
      "severity": "high",
      "red_flag_escalate": true,
      "source": "SPL warnings_and_cautions"
    }
  ],
  "cross_reactivity": [
    {
      "target_class": "cephalosporin_shared_R1",
      "target_examples": ["cefadroxil", "cefprozil", "cephalexin", "cefaclor"],
      "rate": "~1% overall; up to 27-35% for cefadroxil (shared R1)",
      "mechanism": "R1 side-chain similarity",
      "action": "warn",
      "source": "PMID 21742459 / PMC7716594"
    },
    {
      "target_class": "cephalosporin_dissimilar_R1",
      "target_examples": ["cefuroxime", "ceftriaxone", "cefixime"],
      "rate": "<2% (cefuroxime 1.8%)",
      "action": "low_priority_note"
    }
  ],
  "boxed_warning": false,
  "spl_setid": "<from DailyMed>",
  "last_curated": "2026-08-17"
}
```

### 8. Rule Engine Scoring Design

The engine computes, for each recently-taken drug, a **plausibility score** that the reported symptoms are an allergic reaction to that drug. Factors (Naranjo/WHO-UMC-grounded):

| Factor | Logic | Example points |
|---|---|---|
| **Temporal plausibility** | Does symptom onset fall inside a documented onset window for a KB reaction pattern of that drug? Perfect fit (e.g., rash at day 8 for amoxicillin delayed MPE) = full weight; plausible but wide = partial; outside all windows = 0/negative. | +3 in-window / +1 borderline / −2 implausible |
| **Symptom-pattern match** | Overlap between reported symptoms and the pattern's documented symptom set (Jaccard/weighted). | +3 strong / +1 partial |
| **Prior allergy history** | Reported prior allergy to this drug or same class. | +2 same drug / +1 same class |
| **Cross-reactivity** | Prior allergy to a cross-reactive class (e.g., penicillin allergy + took cephalosporin). | +1 to +2 by rate |
| **Alternative cause** | User indicates concurrent illness/other new drugs (Naranjo item 5). | −1 |
| **Known/documented reaction** | Pattern exists in KB from SPL/FAERS (Naranjo item 1). | +1 |

**Thresholds (tunable, transparent):**
- **Low:** total ≤ 2 — symptoms unlikely related; monitor.
- **Moderate:** 3–6 — plausible reaction; contact provider/pharmacist, consider stopping if advised.
- **High:** ≥ 7 — likely reaction; seek medical advice promptly (High is NOT the emergency path — that's the red-flag layer).

Every score returns the itemized factor breakdown → this structured object is what the LLM explains and what the doctor report shows. Determinism means identical inputs always yield identical risk (essential for testing + defensibility).

### 9. Emergency Red-Flag Layer (hard-coded, grounded in NIAID/FAAN + WAO 2020)

Trigger an **URGENT — call emergency services** alert (bypassing the engine and LLM) if the user reports any of:
- **Airway/breathing:** difficulty breathing, wheeze, stridor, throat tightness, hoarse voice/vocal change, difficulty swallowing.
- **Angioedema of danger zones:** swelling of lips, tongue, throat/uvula.
- **Circulatory:** faintness/collapse, syncope, sudden dizziness (possible hypotension).
- **Rapidly progressing:** widespread/rapidly spreading hives + any of the above; or symptoms in ≥2 body systems rapidly after a drug (skin + respiratory / skin + GI + cardiovascular) — the NIAID/FAAN multi-system criterion.
- **SCAR warning signs (delayed but severe):** skin blistering/peeling, mucosal/mouth sores, skin pain + high fever — route to urgent care (SJS/TEN/DRESS).

This layer is a simple deterministic rules table over the symptom checklist; it must be evaluated FIRST and must never depend on the LLM. Design it to favor sensitivity: NIAID/FAAN criteria were prospectively validated at **95.1% sensitivity (95% CI 85.4–98.7) and 70.8% specificity** (Campbell et al., *J Allergy Clin Immunol Pract* 2016, PMID 27406968) — deliberately over-triggering rather than missing a true anaphylaxis.

### 10. LLM Integration (explanation-only, faithful)

- **Input contract:** the LLM receives ONLY the engine's structured JSON (risk level, factor breakdown, drug, matched pattern, onset math) — never the raw symptoms alone, never the decision.
- **Grounding technique:** structured prompt with an explicit system instruction: "You are an explainer. Do NOT change the risk level. Explain the following assessment in plain language and give general safety steps. If risk is High or emergency, tell the user to seek medical care." Use Ollama's schema-constrained `format` to force a JSON response with fields like `explanation`, `why_this_risk`, `recommended_actions`, `disclaimer`.
- **No heavy RAG needed** — the KB is already structured. A light optional retrieval step: pull the 1–2 relevant SPL `adverse_reactions` sentences for the matched drug into the prompt so explanations quote the label. Use Sentence-Transformers + a tiny FAISS/Chroma index over label snippets only if time allows (Phase 2).
- **Temperature 0** for reproducibility; validate output against a Pydantic model, retry once on failure.

### 11. Data Pipeline Plan (one-time curation, not live)

**Decision: one-time build script, NOT runtime API calls.** Rationale: reliability (no runtime dependency on external uptime/rate limits), reproducibility for the report, speed, and the KB rarely changes. Live calls are only for RxNorm autocomplete of user drug entry (optional).

Pipeline (`build_kb.py`):
1. Curate a seed list of ~40 drugs across the 4 target classes.
2. For each: RxNav `/rxcui.json?name=` → RxCUI; RxClass `byRxcui` → ATC/class.
3. openFDA `drug/label.json?search=openfda.generic_name:"X"` → pull `adverse_reactions`, `boxed_warning`, `warnings_and_cautions`, `contraindications`.
4. openFDA `drug/event.json?...&count=patient.reaction.reactionmeddrapt.exact` → top reaction terms + frequencies.
5. **Manual curation step (the academic value-add):** a pharmacology-literate team member maps raw label/FAERS text into structured `reaction_patterns` with onset windows drawn from the literature (this blueprint's Section 2 tables), assigns severity, and encodes cross-reactivity per the side-chain/predisposition rules. Document every mapping decision with its source — this becomes your methods section.
6. Write to SQLite. Commit the DB + a CSV audit trail.

Mind the openFDA no-key daily cap (1,000 requests/day) — for 40 drugs × a few calls each you're fine, but get a free key to be safe if you iterate.

### 12. 8–12 Week Roadmap

- **Weeks 1–2 — Foundations & data:** finalize 40-drug list; write `build_kb.py`; curate KB v1; define the engine↔LLM JSON contract (Pydantic).
- **Weeks 3–4 — Rule engine + emergency layer:** implement deterministic scoring + red-flag table; unit tests against documented profiles.
- **Weeks 5–6 — Frontend + integration:** React symptom checklist, med log, allergy profile; wire to FastAPI; RxNorm autocomplete.
- **Weeks 7–8 — LLM explanation + prevention loop:** Ollama integration, constrained prompt, cross-reactivity warning on new-drug logging.
- **Weeks 9–10 — Report generator + evaluation:** doctor summary PDF; build the test-case suite; validate engine outputs.
- **Weeks 11–12 — Polish, demo, write-up:** UX refinement, disclaimers, demo scenarios, academic report. **Phase 2 stretch (if ahead):** SHAP/LIME on a small classifier, SPL-snippet retrieval, multi-language, rash photo triage.

### 13. Demo Plan (4 scenarios showcasing uniqueness)

1. **Delayed amoxicillin rash detection.** User: took amoxicillin 8 days ago, now maculopapular rash, no breathing issues. Engine matches `amox_delayed_mpe` (onset in-window day 8) → Moderate; LLM explains delayed Type IV reaction, notes it's often non-allergic (esp. with viral illness) but advises contacting provider. *Shows temporal reasoning no logger app does.*
2. **Anaphylaxis red-flag bypass.** User: took a drug 20 min ago, now lip/tongue swelling + difficulty breathing. Emergency layer fires FIRST → URGENT call-emergency alert; engine + LLM never run. *Shows the safety-critical deterministic bypass.*
3. **Cross-reactivity prevention warning.** User has penicillin allergy on file, logs a new prescription for cephalexin. Prevention loop: cephalexin shares R1 side chain with amoxicillin → proactive warning card ("shared side chain elevates risk above the ~1% baseline; confirm with prescriber"). *Shows proactive prevention + honest, non-alarmist rate.*
4. **Myth-avoidance / specificity.** User with "sulfa allergy" (TMP-SMX) logs furosemide. System does NOT hard-warn — shows a low-priority predisposition note citing that antibiotic→non-antibiotic sulfonamide cross-reactivity is not established (Strom et al., NEJM 2003). *Shows the KB encodes evidence, not folklore — a differentiator and a great report talking point.*

### 14. Evaluation Approach (for the academic report)

- **Test-case suite:** author 25–40 synthetic vignettes (drug + onset time + symptoms + history) with an expected risk band derived from documented reaction profiles/guidelines. Because the engine is deterministic, assert exact outputs (regression tests).
- **Validation against documented profiles:** for each KB drug, verify the encoded onset windows and symptoms match SPL/literature (traceability table: KB field → source).
- **Red-flag sensitivity:** a battery of anaphylaxis-positive vignettes must ALL trigger the emergency bypass (target 100% sensitivity — the layer favors sensitivity over specificity by design, mirroring NIAID/FAAN's validated ~95% sensitivity).
- **LLM faithfulness check:** verify the LLM never changes the risk level vs. the engine output across all vignettes (automated string/level comparison); rate explanation quality on a rubric (accuracy, no fabrication, appropriate urgency).
- **Ablation for the report:** show risk changes appropriately as you vary onset time alone (demonstrates temporal reasoning is doing real work).
- **Limitations to state honestly:** not a diagnostic device; KB limited to ~40 drugs; FAERS is spontaneous-report data (reporting bias, no denominator); consumer self-report is noisy; simplified Naranjo adaptation drops rechallenge/lab items.

## Recommendations

1. **Start with the KB + rule engine (Weeks 1–4) before any UI.** The engine is the differentiator and the highest-risk component. Get deterministic scoring + red-flag tests green first.
2. **Adopt the Naranjo/WHO-UMC framing explicitly in your report and code comments.** Name your factors after their Naranjo analogues — this converts "we made up weights" into "we adapted a validated causality instrument," which is the single biggest defensibility win.
3. **Use openFDA `drug/label.json` for curation parsing, DailyMed as ground-truth reference.** Don't parse SPL XML unless you need version history — the named JSON fields save days.
4. **Keep the LLM on a tight leash:** structured input only, schema-constrained output, temperature 0, explicit "never change the risk" instruction, Pydantic validation. Demonstrate in the report that the LLM cannot alter risk — this is your safety story.
5. **Encode evidence, not folklore.** The penicillin-cephalosporin ~1% (not 10%) and sulfonamide non-cross-reactivity findings are both differentiators and defensible design decisions — feature them.
6. **Run the pipeline once and commit the artifact.** Live external calls at runtime add fragility for zero capstone benefit.

**Benchmarks that would change the plan:** if you can't run a local 8B model (no GPU), switch to a free API tier (Groq/Gemini free) — architecture unchanged. If KB curation for 40 drugs exceeds ~2 weeks, cut to 25 drugs concentrated in beta-lactams + NSAIDs (richest demo value). If red-flag sensitivity testing shows any miss, widen triggers before adding features — safety layer must be perfect before polish.

## Caveats
- **Regulatory framing:** keep it strictly a decision-support/education tool, not a diagnostic. The FDA's Software-as-a-Medical-Device criteria turn on patient-specific *diagnostic* recommendations; a graded-risk educational tool with prominent "seek professional care" disclaimers stays on the safe side, but state this explicitly in the report.
- **FAERS data quality:** spontaneous reports have no denominator, duplicate reports, and reporting bias — use FAERS for *which* reactions are documented, not for computing incidence rates.
- **Cross-reactivity numbers are population estimates**, not individual risk; the app must communicate them as such (Scenarios 3/4 handle this).
- **HLA pharmacogenomic risk** (carbamazepine/HLA-B*15:02) is out of MVP scope (requires genetic data) but is a strong Phase 2 / report "future work" item.
- **Onset windows overlap** (immediate up to 6 h, delayed from 6 h) — the engine should handle boundary cases gracefully (a 4-hour rash could be either); reflect this uncertainty in the score rather than forcing a binary.
- **Source recency note:** anaphylaxis criteria have evolved (NIAID/FAAN 2006 → WAO 2020 → GA²LEN 2024 consensus). The MVP can safely implement NIAID/FAAN + WAO 2020 triggers; note the 2024 consensus as a refinement path.