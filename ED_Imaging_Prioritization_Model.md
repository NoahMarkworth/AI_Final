# ED Imaging Queue Prioritization Model
## Hybrid Two-Layer Architecture — Full Specification

**Project:** Nebraska Medicine ED Imaging Queue Simulator  
**Grounded in:** Dr. Zeger & MJ Nielsen empathy interviews | ESI Handbook v4 | NHAMCS 2021 | 54-condition clinical reference | AHA/ASA Stroke Guidelines 2019 | ACEP Chest Pain / Cholangitis / CES guidelines  
**Status:** Model design complete — pre-code specification  
**Version:** 1.0

---

## Part 1: Problem Statement and Design Context

Nebraska Medicine's ED imaging queue operates on a first-come, first-served basis. Desk technicians with limited medical training coordinate patient movement to radiology. A stroke patient — losing 1.9 million neurons per minute without reperfusion — can sit behind a lower-acuity patient simply because they arrived earlier.

This model exists to make that cost visible. It is a proof-of-concept advocacy tool designed to demonstrate to leadership what a smarter queue could look like in practice.

**The model does not replace clinical judgment. It makes the gap between current state and possible future state concrete.**

---

## Part 2: Model Architecture Overview

The model is hybrid — two layers working in sequence.

```
INPUT (structured patient data)
        ↓
LAYER 1: DETERMINISTIC SCORING
  → Produces numerical urgency score
  → Every weight traceable to a source file
  → Fully auditable
        ↓
LAYER 2: AI REASONING (conditional activation)
  → Activates only for:
    (a) Incomplete data — deterministic layer cannot fully evaluate
    (b) Tiebreaking — scores within 10 points
  → Reasons about delay consequence asymmetry at 20/40/60-minute intervals
  → Sourced from clinical chapter time-sensitivity data
        ↓
OUTPUT: Ranked imaging queue + plain-language rationale per patient
```

---

## Part 3: Input Variables

All input variables are drawn directly from the NHAMCS 2021 Emergency Department micro-data file structure (doc21-ed-508.pdf, Section II Codebook).

| Variable | NHAMCS Field | Description | Range / Codes |
|---|---|---|---|
| Triage Level | IMMEDR | ESI immediacy rating | 1=Immediate, 2=Emergent, 3=Urgent, 4=Semi-urgent, 5=Nonurgent |
| Temperature | TEMPF | Triage vital sign | 80.6–106.0°F (implied decimal) |
| Heart Rate | PULSE | Beats per minute | 0–300 |
| Respiratory Rate | RESPR | Breaths per minute | 0–60 |
| Systolic BP | BPSYS | mmHg | 0–300 |
| Diastolic BP | BPDIAS | mmHg | 0–200 |
| SpO2 | POPCT | Pulse oximetry % | 0–100 |
| Pain Scale | PAINSCALE | Self-reported 0–10 | 0–10 |
| Chief Complaint 1–5 | RFV1–RFV5 | Reason for visit | Free text / coded |
| Arrival Mode | ARREMS | Arrived by ambulance | 1=Yes, 2=No |
| Imaging Ordered | CATSCAN, CTAB, CTCHEST, CTHEAD, MRI, ULTRASND | Imaging flags | 1=Yes, 2=No |
| Contrast Required | CTCONTRAST, MRICONTRAST | IV contrast needed | 1=Yes, 2=No |
| Age | AGE | Patient age in years | 0–120 |
| Sex | SEX | Patient sex | 1=Female, 2=Male |

**Source:** NHAMCS 2021 Micro-Data File Documentation (doc21-ed-508.pdf), Section II Codebook, pp. 32–92.

---

## Part 4: Layer 1 — Deterministic Scoring

### 4.1 Score Architecture

The deterministic urgency score is the sum of seven independent components:

```
URGENCY SCORE = C1 + C2 + C3 + C4 + C5 + C6 + C7
```

Where:
- **C1** = ESI Base Score
- **C2** = Condition Time-Sensitivity Modifier
- **C3** = Imaging Modality Urgency Modifier
- **C4** = Vital Sign Danger Zone Modifier
- **C5** = Pain Score Modifier
- **C6** = Arrival Mode Modifier
- **C7** = Age Risk Modifier

**Scoring range:** Minimum 5 (ESI-5, no risk factors) — Maximum ~220 (ESI-1, all modifiers maxed)

**Tiebreaker threshold:** Layer 2 AI reasoning activates when two patients score within 10 points of each other.

---

### 4.2 Component 1: ESI Base Score (C1)

**Source:** ESI Implementation Handbook v4 (Esi_Handbook.pdf), Chapters 2–3. The ESI five-level algorithm is the structural spine of Layer 1.

The ESI assigns patients to one of five levels based on four decision points:
- **Decision Point A:** Does the patient require an immediate life-saving intervention? (ESI 1)
- **Decision Point B:** Is this a high-risk situation, or is the patient confused/lethargic/in severe pain? (ESI 2)
- **Decision Point C:** How many resources will this patient need? (ESI 3/4/5)
- **Decision Point D:** Are vital signs in the danger zone? (consider up-triaging from 3→2)

**Scoring:**

| ESI Level | NHAMCS Code | Clinical Meaning | C1 Score |
|---|---|---|---|
| ESI 1 — Immediate | IMMEDR = 1 | Requires immediate life-saving intervention. Airway, apnea, pulselessness, severe respiratory distress, SpO2 <90%, acute mental status change, unresponsive. | 100 |
| ESI 2 — Emergent | IMMEDR = 2 | Should not wait. High-risk situation, or confused/lethargic, or severe pain/distress ≥7/10. Placement should occur within 10 minutes of arrival. | 75 |
| ESI 3 — Urgent | IMMEDR = 3 | Requires 2+ resources. Vital signs may warrant up-triage. | 45 |
| ESI 4 — Semi-Urgent | IMMEDR = 4 | Requires 1 resource. Stable. | 15 |
| ESI 5 — Nonurgent | IMMEDR = 5 | No resources expected. Exam and prescription only. | 5 |

**ESI Level distribution in the real NHAMCS 2021 dataset** (16,207 total ED visits):
- Immediate: 2.95% of visits
- Emergent: 10.36% of visits
- Urgent: 34.75% of visits
- Semi-urgent: 18.34% of visits
- Nonurgent: 2.50% of visits
- Unknown/no triage: ~31% (not in scoring scope)

**Source:** NHAMCS 2021, Section III Marginal Data, Emergency Department Patient Visits table (doc21-ed-508.pdf, p. 93).

**Key ESI distinctions for queue prioritization:**

The ESI handbook draws a critical operational distinction that is directly applicable to this model: *among ESI-2 patients specifically, the queue model must apply additional differentiation.* ESI-2 encompasses patients ranging from a stable chest pain patient needing an ECG within 10 minutes to a patient with active stroke symptoms for whom every minute of delay permanently destroys brain tissue. The base score treats all ESI-2 patients at 75 points, and C2 (Condition Time-Sensitivity) is the primary mechanism that separates them.

**Reference:** ESI Handbook, Chapter 4 (ESI Level 2), p. 27: "When faced with multiple ESI level-2 patients simultaneously, the triage nurse must evaluate each patient according to the ESI algorithm. Then, the nurse can 'triage' all level-2 patients to determine which patient(s) are at highest risk, in order to facilitate patient placement based on this evaluation."

---

### 4.3 Component 2: Condition Time-Sensitivity Modifier (C2)

**Source:** ER Imaging Optimization Clinical Intelligence Reference, Chunks 1–5 (ER_Imaging_Optimization_Neuro_Chunk1.pdf through ER_Imaging_Optimization_Trauma_Oncology_Chunk5.pdf).

This is the most clinically significant component. It maps chief complaint patterns to one of 54 documented conditions and assigns a modifier based on the clinical consequence window — the time frame within which delay produces irreversible harm.

**Four tiers derived from the clinical reference:**

---

#### TIER 1 — CATASTROPHIC TIME-SENSITIVITY (+40 points)
**Definition:** Death or irreversible harm measured in minutes. Every minute of imaging delay actively worsens outcome. No safe waiting window.

| Condition | Primary Imaging | Time-Sensitivity Evidence | Chief Complaint Patterns |
|---|---|---|---|
| **Tension Pneumothorax** | Clinical diagnosis first; CXR/POCUS confirms | "Death within minutes to hours without decompression." Needle decompression within 60–90 seconds. | Sudden dyspnea, absent breath sounds, hypotension, tracheal deviation |
| **Ruptured AAA (free rupture)** | POCUS → OR; CTA if transiently stable | "Survival measured in minutes without surgery." Free rupture: 100% mortality without OR. | Sudden back/flank/abdominal pain + hypotension + known AAA |
| **Cardiac Tamponade (acute/traumatic)** | POCUS echo | "Penetrating trauma tamponade: minutes from collapse to arrest." PEA inevitable without drainage. | Hypotension + JVD + muffled heart sounds + penetrating trauma |
| **STEMI with cardiogenic shock** | ECG first; echo; angio | "Each 30-minute delay increases 1-year mortality ~7.5%." Door-to-balloon ≤90 min. | Crushing chest pain + diaphoresis + hypotension + ST elevation |
| **Aortic Dissection Type A** | CTA chest | "Mortality 1–2% per hour in first 48 hours." Target OR within 2 hours of confirmed diagnosis. | Sudden tearing chest/back pain + pulse discrepancy + hypertension/hypotension |
| **Massive Hemothorax** | CXR → CTA | "Massive hemothorax: near 100% mortality without drainage and source control." | Trauma + absent breath sounds + dull percussion + hemodynamic instability |
| **Ruptured Ectopic Pregnancy (hemorrhagic shock)** | POCUS FAST → OR | "Leading cause of first-trimester maternal death." OR within minutes if hemodynamically unstable + positive hCG. | Positive hCG + pelvic pain + hemodynamic instability |

**C2 Score: +40**

---

#### TIER 2 — CRITICAL TIME-SENSITIVITY (+30 points)
**Definition:** Irreversible harm begins within 1–6 hours of symptom onset. Established time-locked intervention windows. Imaging delays directly eliminate treatment eligibility.

| Condition | Primary Imaging | Time-Sensitivity Evidence | Chief Complaint Patterns |
|---|---|---|---|
| **Ischemic Stroke (LVO)** | NCCT head → CTA → CT perfusion | "1.9 million neurons die every minute." Door-to-CT ≤25 min; door-to-needle ≤60 min; door-to-groin ≤90 min. tPA window 4.5 hrs; thrombectomy window 6–24 hrs. | Acute focal neuro deficit: facial droop, arm weakness, speech disturbance, gaze deviation |
| **Hemorrhagic Stroke (ICH)** | NCCT head | "Hematoma expands in 20–40% within first 3–6 hours." BP reduction to <140 within 1 hour of imaging. Hemostatic therapy most effective within 3–4 hours. | Sudden severe headache + focal deficits + declining GCS + severe hypertension |
| **Subarachnoid Hemorrhage** | NCCT head | "Rebleed in 4–17% within 24 hours with 50–80% mortality." CT within 25 min; CTA/LP within 1 hour of negative CT. | "Worst headache of life" — thunderclap onset; nuchal rigidity; nausea/vomiting |
| **Acute Limb Ischemia (Grade IIb)** | CTA → OR/angio | "6-hour critical window for limb salvage." Nerves fail at 1 hour; muscle necrosis at 6 hours; OR within 2–6 hours for Grade IIb. | Sudden severe limb pain + pallor + absent Doppler + paresthesia/paralysis — "6 P's" |
| **Testicular/Ovarian Torsion** | Doppler US | "Testicular viability ~100% within 6 hours; <10% after 24 hours." OR within 6 hours (testicular), 4–8 hours (ovarian). | Sudden severe unilateral scrotal/pelvic pain + nausea/vomiting + absent cremasteric reflex |
| **Orbital Compartment Syndrome** | Clinical → CT orbits | "Irreversible vision loss within 60–90 minutes." LCIC within 60 minutes of vision loss onset. | Facial/orbital trauma + proptosis + decreased vision + IOP >40 mmHg |
| **Boerhaave Syndrome** | CT chest + Gastrografin esophagogram | "Primary repair feasible <24 hrs." Mortality <12 hrs: 10–15%. Mortality >24 hrs: >50%. Mortality untreated: ~100%. | Forceful vomiting + severe chest pain + subcutaneous emphysema + Hamman's sign |
| **Ludwig's Angina / Epiglottitis (airway compromise)** | CECT neck | "Airway can close within hours." Early awake fiberoptic intubation far safer than emergent. | Floor-of-mouth induration + trismus + drooling + stridor |
| **Cauda Equina Syndrome (complete)** | MRI lumbar spine | "Surgical decompression within 24 hours; ideally <6 hours." Permanent sphincter dysfunction if untreated >48–72 hrs. | Saddle anesthesia + urinary retention + bilateral leg weakness + back pain |
| **Epidural Hematoma (expanding)** | NCCT head | "Arterial hemorrhage rapidly expanding over 6–8 hours." OR within 60–90 minutes of neurological deterioration. | Trauma + lucid interval + secondary rapid deterioration + blown pupil |
| **Phlegmasia Cerulea Dolens** | Doppler US → CTV | "0–6 hours: limb salvage possible. >12–24 hours: irreversible venous gangrene." CDT within 6 hours. | Massive leg edema + cyanosis + motor/sensory deficits + pain |

**C2 Score: +30**

---

#### TIER 3 — HIGH TIME-SENSITIVITY (+20 points)
**Definition:** Irreversible harm window of 6–24 hours. Delays measurably worsen outcome but a brief queue wait remains acceptable with active monitoring.

| Condition | Primary Imaging | Time-Sensitivity Evidence | Chief Complaint Patterns |
|---|---|---|---|
| **Acute Mesenteric Ischemia** | CTA abdomen/pelvis | "0–6 hours: bowel viability preserved." Mortality doubles once bowel necrosis occurs (20% vs 50–80%). | Severe abdominal pain "out of proportion" + normal exam + AF history + elevated lactate |
| **Ascending Cholangitis Grade III (Reynolds Pentad)** | RUQ US → MRCP/CT | "Grade III: emergent ERCP within 12 hours." Mortality 20–50% without drainage. | Fever + RUQ pain + jaundice + confusion + hypotension (Reynolds Pentad) |
| **Subdural Hematoma (acute with GCS decline)** | NCCT head | "Surgery within 4 hours of deterioration." SDH >10 mm or midline shift >5 mm → emergent craniotomy. | Trauma + declining GCS + hemiparesis + midline shift |
| **Ruptured Spleen/Liver (active arterial hemorrhage)** | FAST → CT abdomen | "Angioembolization within 60–90 minutes of CT blush identification." | Blunt abdominal trauma + LUQ/RUQ pain + hemodynamic instability + FAST positive |
| **Pelvic Fracture with Hemorrhage** | Pelvic X-ray → CTA pelvis | "Exsanguination within 30–60 minutes without hemorrhage control." Angioembolization within 60–90 min of CT blush. | High-energy trauma + pelvic instability + hemodynamic instability + no other bleeding source |
| **Fournier's Gangrene** | CT pelvis/perineum | "Mortality increases 3–9% per hour of delay." OR within 6 hours. | Genital/perineal pain + rapidly spreading erythema + crepitus + sepsis in diabetic |
| **Necrotizing Fasciitis** | CT affected area | "Fascia spreads 1–2 cm/hour." Mortality doubles if surgery >12 hours. | Rapidly spreading erythema beyond apparent cellulitis + pain disproportionate to findings + sepsis |
| **Pulmonary Embolism (massive)** | CTPA | "Massive PE without treatment: ~80% mortality." Systemic thrombolysis within minutes of confirmed diagnosis. | Sudden dyspnea + syncope + tachycardia + hypotension + known DVT/PE risk |
| **Aortic Dissection Type B (complicated)** | CTA chest | "Complications develop within 24–72 hours." TEVAR for complicated Type B. | Tearing back/interscapular pain + hypertension + pulse discrepancy (less dramatic than Type A) |
| **Ruptured Viscus (free perforation)** | CT abdomen | "Peritonitis worsens every hour. OR within 2–6 hours." Mortality doubles at >24-hour delay. | Sudden abdominal pain → board-like abdomen → absent bowel sounds + free air on X-ray |
| **Traumatic Aortic Injury Grade II–III** | CTA chest | "30% die within 6 hours without treatment." TEVAR within 24 hours; anti-impulse therapy immediately. | High-velocity deceleration + chest pain + mediastinal widening on CXR |
| **CNS Infection (HSV Encephalitis)** | MRI brain with gadolinium | "IV acyclovir should begin immediately upon clinical suspicion — do not wait for imaging or LP." Mortality without acyclovir: 70–80%. | Fever + acute confusion + behavioral change + temporal lobe seizures |
| **Urosepsis with Obstructive Uropathy** | CT abdomen (non-contrast) | "Decompression within 6–12 hours." Mortality 20–40% with delayed drainage vs <5% with prompt drainage. | Flank pain + high fever + rigors + CVA tenderness + sepsis |

**C2 Score: +20**

---

#### TIER 4 — MODERATE-HIGH TIME-SENSITIVITY (+10 points)
**Definition:** Clinical deterioration expected within 24–72 hours without imaging. Meaningful but not catastrophic if imaging is briefly delayed.

| Condition | Primary Imaging | Time-Sensitivity Evidence |
|---|---|---|
| Appendicitis (non-perforated) | CT abdomen | Perforation risk ~20% at 24 hrs, ~65% at 48 hrs. |
| Bowel Obstruction (complete, no strangulation) | CT abdomen | Strangulation develops within 24–72 hrs in complete obstruction. |
| Ascending Cholangitis Grade II | RUQ US/MRCP | ERCP within 24–48 hours. |
| Spinal Cord Compression (MSCC, ambulatory) | MRI spine | Ambulatory status at treatment is primary predictor of recovery. |
| Epidural/Psoas Abscess (no motor deficit) | MRI spine | Can progress to motor deficit within hours; surgery if deficit develops. |
| Retroperitoneal Abscess | CT abdomen | Sepsis progression if undrained. |
| Carotid/Vertebral Artery Dissection | CTA head/neck | Embolic stroke risk highest in first 24–72 hours. |
| Pulmonary Embolism (submassive, hemodynamically stable) | CTPA | RV dysfunction may progress; catheter-directed therapy window. |
| Sigmoid Volvulus (no ischemia) | CT abdomen | Ischemia can develop within hours without decompression. |
| Ascending Cholangitis Grade I | RUQ US/MRCP | ERCP within 72 hours. |

**C2 Score: +10**

---

#### TIER 5 — MODERATE TIME-SENSITIVITY (+5 points)
**Definition:** Clinical management is improved by imaging, but brief delays (hours) are unlikely to change ultimate outcome in the absence of deterioration.

Examples:
- Appendicitis (early, no perforation signs)
- Uncomplicated pneumonia requiring CT characterization
- Non-emergent biliary pathology
- Stable pancreatitis staging (CT after 48–72 hours per guidelines)
- Stable diverticulitis

**C2 Score: +5**

---

#### TIER 6 — LOW TIME-SENSITIVITY (+0 points)
**Definition:** Imaging is clinically indicated but no meaningful time-pressure exists.

Examples:
- Incidental findings requiring workup
- Chronic condition follow-up imaging
- Low-risk chest pain with negative troponin (HEART score ≤3)
- Elective or non-urgent clinical questions

**C2 Score: +0**

---

#### Chief Complaint to Condition Mapping

The model maps chief complaints to conditions using a pattern-matching approach. The following patterns are derived from the clinical reference presentations:

**Stroke presentation:** FAST criteria (Facial droop, Arm weakness, Speech difficulty, Time) → Ischemic stroke / ICH / SAH algorithm → Tier 1–2 depending on hemodynamics. *Source: Neuro Chunk 1, Ischemic Stroke: "Sudden onset focal neurological deficit in a defined vascular territory."*

**Chest pain + hemodynamic instability:** → STEMI with shock (Tier 1) or Aortic Dissection Type A (Tier 1)

**Chest pain + hemodynamic stability:** → STEMI (Tier 2, requires ECG first) or Aortic Dissection evaluation

**"Worst headache of life" / thunderclap headache:** → SAH (Tier 2) — *Source: Neuro Chunk 1, SAH: "Thunderclap headache reaching maximum intensity within seconds."*

**Abdominal pain + AF + elevated lactate:** → Mesenteric Ischemia (Tier 3)

**Abdominal pain + fever + jaundice + confusion:** → Cholangitis Grade III (Tier 3)

**Back pain + saddle anesthesia + urinary retention:** → Cauda Equina Syndrome (Tier 2)

**Known malignancy + new back pain + leg weakness:** → MSCC (Tier 4)

**Scrotal pain + absent cremasteric reflex:** → Testicular torsion (Tier 2)

**Pelvic pain + positive hCG + hemodynamic instability:** → Ruptured ectopic (Tier 1)

**Genital/perineal pain + crepitus + sepsis:** → Fournier's Gangrene (Tier 3)

**Rapidly spreading erythema + sepsis + pain disproportionate:** → Necrotizing Fasciitis (Tier 3)

**Tearing back/flank pain + known AAA + hypotension:** → Ruptured AAA (Tier 1 if unstable, Tier 2 if transiently stable)

*This mapping is not exhaustive. The AI layer (Layer 2) handles ambiguous chief complaint presentations.*

---

### 4.4 Component 3: Imaging Modality Urgency Modifier (C3)

**Source:** ER Imaging Optimization Clinical Reference (all chunks) — Imaging and Time-Sensitivity sections. Imaging modality selection and urgency is condition-specific and directly determined by the clinical literature.

**Rationale:** Not all imaging carries equal time pressure. A head CT for a stroke patient has a 25-minute door-to-CT target per AHA/ASA guidelines. An abdominal CT for appendicitis has a 3–6 hour target per ACEP. This component captures that distinction.

| Imaging Modality | Clinical Context | C3 Score | Source Citation |
|---|---|---|---|
| **CT Head (non-contrast)** | Suspected stroke, ICH, SAH, TBI | +15 | AHA/ASA Stroke Guidelines 2019: "Door-to-CT ≤25 minutes." Powers et al. 2019. |
| **CT Head + CTA (stroke LVO protocol)** | Confirmed stroke symptoms + LVO screening | +15 | Powers et al. 2019: "Noninvasive vessel imaging of intracranial arteries is recommended during initial imaging evaluation." |
| **CTA Chest/Abdomen (aortic, vascular emergency)** | Aortic dissection, AAA, mesenteric ischemia, traumatic aortic injury | +12 | Cardio Chunk 2: "CTA within 30 minutes of presentation" for aortic dissection. |
| **CTPA (pulmonary embolism)** | Suspected massive or submassive PE | +12 | Cardio Chunk 2: "CTPA should be obtained as the priority investigation in suspected high-risk PE." |
| **MRI Brain (stroke/encephalitis)** | Wake-up stroke, posterior circulation, HSV encephalitis | +12 | Neuro Chunk 1: "MRI DWI sensitivity >95% within 3–6 hours of onset." |
| **MRI Spine (with gadolinium)** | Suspected CES, MSCC, epidural abscess | +12 | Neuro Chunk 1, Pulm Chunk 4: "MRI within 4 hours of CES presentation." |
| **CT Abdomen/Pelvis with contrast** | Appendicitis, bowel obstruction, mesenteric ischemia, pelvic fracture | +8 | GI Chunk 3: "CT within 3–6 hours of suspected appendicitis." |
| **Scrotal/Pelvic Doppler Ultrasound** | Testicular or ovarian torsion | +12 | Pulm Chunk 4: "OR within 6 hours of onset (testicular)." Doppler US is the gating study. |
| **CT Neck with contrast** | Retropharyngeal abscess, Ludwig's, epiglottitis, Lemierre's | +10 | Cardio Chunk 2: "Airway should be assessed and secured before imaging." |
| **FAST / POCUS** | Trauma, tamponade, ectopic | +0 (done in ER, not in imaging queue) | ATLS guidelines: bedside, not in radiology queue. |
| **CT Orbits** | Orbital compartment syndrome | +12 | Cardio Chunk 2: "LCIC within 60 minutes of vision loss onset." |

**Contrast modifier:**
- IV contrast required: C3 score is as listed above (contrast cases already factored in)
- Note: Contrast requirement does not reduce priority — it is part of the appropriate imaging protocol

---

### 4.5 Component 4: Vital Sign Danger Zone Modifier (C4)

**Source:** ESI Implementation Handbook v4 (Esi_Handbook.pdf), Chapter 6 "The Role of Vital Signs in ESI Triage," Figure 6-1 (Danger Zone Vital Signs), p. 41–46.

The ESI danger zone vital signs are the evidence base for this component. These are the validated thresholds that trigger consideration of up-triage from ESI 3 to ESI 2.

**From Figure 6-1 of the ESI Handbook (Decision Point D):**

| Age Group | HR Threshold | RR Threshold | SpO2 |
|---|---|---|---|
| <3 months | >180 | >50 | <92% |
| 3 months – 3 years | >160 | >40 | <92% |
| 3–8 years | >140 | >30 | <92% |
| >8 years (adults) | >100 | >20 | <92% |

**Vital Sign Scoring:**

| Vital Sign Finding | C4 Score | Source |
|---|---|---|
| SpO2 <92% (any age) | +15 | ESI Handbook Ch. 6: "Uptriage to ESI 2." Danger zone threshold. |
| SpO2 <90% (any age) | +20 | ESI Handbook Ch. 3: ESI Level 1 criterion — "SpO2 <90%." |
| Systolic BP <90 mmHg | +20 | Clinical chapters: hemodynamic instability = highest priority across all conditions. |
| HR >120 (adult) | +15 | Clinical chapters: severe tachycardia correlates with hemodynamic compromise. |
| HR >100 (adult, per ESI danger zone) | +10 | ESI Handbook Ch. 6, Figure 6-1. |
| RR >30 (adult) | +10 | Clinical chapters: respiratory failure threshold. |
| RR >20 (adult, per ESI danger zone) | +5 | ESI Handbook Ch. 6, Figure 6-1. |
| Temperature >39.5°C / 103.1°F | +8 | Clinical chapters: high fever suggests serious infection (sepsis, encephalitis). |
| Temperature >38.5°C / 101.3°F | +5 | Clinical chapters: fever threshold for serious infection in at-risk populations. |
| HR >160 (child 3mo–3yr) or >140 (child 3–8yr) | +10 | ESI Handbook Ch. 6, Figure 6-1, pediatric danger zones. |
| Cushing's Triad (bradycardia + hypertension + irregular RR) | +25 | Neuro Chunk 1, Trauma Chunk 5: "Impending herniation." |

**Vital sign modifier cap:** Maximum C4 contribution = 35 points (to avoid over-weighting a single deteriorating vital in isolation from clinical context).

**Important ESI caveat:** The ESI handbook explicitly states that blood pressure is NOT included in the ESI algorithm's vital sign danger zone. BP is included here as a clinical modifier derived from the condition-specific literature, not from the ESI algorithm itself. *Source: ESI Handbook Ch. 6, p. 44: "It is important to note that when considering abnormal vital signs, blood pressure is not included in the ESI algorithm."*

**Vital sign interaction flag:** When SpO2 <92% AND HR >100 AND RR >20 all appear simultaneously, this triggers Layer 2 AI reasoning regardless of score proximity, because this triad indicates physiological decompensation that may not be fully captured by additive scoring.

---

### 4.6 Component 5: Pain Score Modifier (C5)

**Source:** ESI Implementation Handbook v4 (Esi_Handbook.pdf), Chapter 3, p. 20–21; Chapter 4, p. 28–31.

| Pain Score | C5 Score | Source |
|---|---|---|
| 9–10 (severe, with objective signs) | +8 | ESI Handbook Ch. 3: Pain ≥7 may triage as ESI 2. Objective signs of distress (diaphoresis, tachycardia, body posture) support. |
| 7–8 (severe, self-reported) | +5 | ESI Handbook: "Self-reported pain rating of 7 or higher on scale of 0–10" — may triage as ESI 2. |
| 5–6 (moderate) | +3 | Not an ESI trigger but relevant queue modifier. |
| ≤4 (mild to none) | +0 | ESI Handbook: "A patient with a sprained ankle and pain of 8/10 is a good example of an ESI level-4 patient. It is not necessary to rate this patient as a level 2 based on pain alone." Pain score modifies, but does not override clinical context. |

**Key limitation from the ESI handbook:** "Pain is one of the most common reasons for an ED visit and clearly all patients reporting pain 7/10 or greater do not need to be assigned an ESI level-2 triage rating." *Source: ESI Handbook Ch. 3, p. 20.* Therefore, C5 is a supplementary modifier and is deliberately capped at +8 to prevent pain score alone from overriding clinical assessment.

---

### 4.7 Component 6: Arrival Mode Modifier (C6)

**Source:** NHAMCS 2021 (ARREMS variable); AHA/ASA Stroke Guidelines 2019 (Powers et al.).

| Arrival Mode | C6 Score | Source |
|---|---|---|
| Arrived by ambulance (ARREMS = 1) | +5 | Ambulance arrival is associated with higher acuity and shorter door-to-imaging times in stroke (AHA/ASA 2019: "EMS use independently associated with quicker ED evaluation — more patients with door-to-imaging time ≤25 minutes"). |
| Private vehicle or walk-in (ARREMS = 2) | +0 | ESI Handbook Ch. 3, p. 24: "Arriving by ambulance is not a criterion to assign a patient ESI level 1 or 2. The ESI criteria should always be used to determine triage level without regard to method of arrival." Arrival mode is a weak modifier only — the ESI handbook explicitly cautions against over-weighting it. |

---

### 4.8 Component 7: Age Risk Modifier (C7)

**Source:** ESI Implementation Handbook v4 (Esi_Handbook.pdf), Chapters 4 and 6; NHAMCS 2021 (doc21-ed-508.pdf) — pediatric and elderly ED utilization data; clinical reference chapters throughout.

| Age Group | C7 Score | Source |
|---|---|---|
| Neonatal (1–28 days) | +10 | ESI Handbook Ch. 6: "Infant <28 days with fever >38.0°C: assign at least ESI 2." Highest vulnerability. |
| Infant (1–3 months) | +8 | ESI Handbook Ch. 6: "Consider assigning ESI 2 if temp >38.0°C." |
| Young child (3 months – 3 years) | +5 | ESI Handbook Ch. 6: Pediatric fever guidelines; higher triage concern. |
| Child (3–12 years) | +3 | Standard pediatric concern for rapid deterioration. |
| Adult (13–64 years) | +0 | Baseline. |
| Elderly (65–79 years) | +5 | ESI Handbook Ch. 4: "A frail elderly patient with severe abdominal pain is at a much higher risk of morbidity and mortality than a 20-year-old." Clinical chapters note atypical presentations, reduced physiological reserve, and higher risk of rapid deterioration. |
| Very elderly (≥80 years) | +8 | NHAMCS 2021: "Highest rate of ED visits is by persons age 75 and older." Clinical chapters (GI, ICH, stroke): mortality significantly higher in oldest-old. |

---

### 4.9 Scoring Example

**Patient A:** 68-year-old male, arrived by ambulance. ESI 2. Chief complaint: sudden left arm weakness and facial droop, onset 45 minutes ago. Vitals: HR 88, RR 18, BP 195/110, SpO2 97%. Pain 2/10. CT Head ordered.

| Component | Value | Score |
|---|---|---|
| C1 — ESI 2 | Emergent | 75 |
| C2 — Ischemic Stroke (LVO likely) | Tier 2: Critical | +30 |
| C3 — CT Head (stroke protocol) | CT Head + CTA | +15 |
| C4 — Vital signs | BP 195/110 (elevated, not <90; HR 88 normal; RR 18 normal; SpO2 97%) | +0 |
| C5 — Pain 2/10 | Low | +0 |
| C6 — Ambulance | Yes | +5 |
| C7 — Age 68 | Elderly | +5 |
| **TOTAL** | | **130** |

**Patient B:** 34-year-old female, walk-in. ESI 3. Chief complaint: right lower quadrant pain, 8/10 pain, nausea, anorexia x 18 hours. Vitals: HR 104, RR 18, BP 118/72, SpO2 99%. CT Abdomen with contrast ordered.

| Component | Value | Score |
|---|---|---|
| C1 — ESI 3 | Urgent | 45 |
| C2 — Appendicitis (non-perforated) | Tier 4: Moderate-High | +10 |
| C3 — CT Abdomen | Abdominal CT | +8 |
| C4 — Vital signs | HR 104 (adult danger zone) | +10 |
| C5 — Pain 8/10 | Severe | +5 |
| C6 — Walk-in | No | +0 |
| C7 — Age 34 | Adult | +0 |
| **TOTAL** | | **78** |

**Result:** Patient A (Score 130) is prioritized ahead of Patient B (Score 78) in the imaging queue. The 52-point gap significantly exceeds the 10-point Layer 2 activation threshold, so no AI reasoning is needed — the score is determinative.

**Plain-language rationale (auto-generated):** *"Patient A (stroke symptoms, 45 min onset) is prioritized ahead of Patient B (appendicitis). Patient A is losing approximately 1.9 million neurons per minute without reperfusion. The door-to-CT target for suspected stroke is 25 minutes (AHA/ASA guidelines). Patient B's appendicitis workup has a 3–6 hour guideline target; 30 additional minutes will not meaningfully change outcomes. Patient A has a closing treatment window measured in minutes."*

---

## Part 5: Layer 2 — AI Reasoning

### 5.1 Activation Conditions

Layer 2 activates in exactly two scenarios:

**Scenario A — Incomplete Data**
Patient data is insufficient for the deterministic layer to fully evaluate urgency. This includes:
- Missing vital signs (TEMPF, PULSE, RESPR, BPSYS, BPDIAS, POPCT all blank)
- Missing triage level (IMMEDR = -9 or -8)
- Chief complaint too vague to map to a condition tier (e.g., "pain" without location or context)
- Imaging type ordered but no chief complaint provided

When activated for incomplete data, the AI reasons from available inputs and flags the confidence level of the resulting score.

**Scenario B — Tiebreaking**
Two patients have deterministic scores within **10 points** of each other. The 10-point threshold is derived from the accumulated uncertainty in individual component estimates — a gap smaller than 10 points is below the model's reliable resolution.

---

### 5.2 Delay Consequence Asymmetry Reasoning

When Layer 2 activates, the AI reasons about what happens to each patient specifically if imaging is delayed by 20, 40, and 60 minutes. This reasoning is drawn directly from the clinical chapter time-sensitivity windows and consequence-of-delay sections.

**Framework:**

For each patient in a tiebreaker pair, the AI produces:

```
Patient [X] at +20 minutes of imaging delay:
→ [Specific consequence drawn from clinical chapter]
→ Treatment window affected: [Yes/No] — [how]

Patient [X] at +40 minutes of imaging delay:
→ [Specific consequence]
→ Treatment eligibility change: [description]

Patient [X] at +60 minutes of imaging delay:
→ [Specific consequence]
→ Point of no return: [reached / approaching / not yet]
```

**Example application (tiebreaker):**

Patient A — ESI 2, chest pain, suspected NSTEMI, troponin pending. Score: 92.
Patient B — ESI 2, headache with sudden onset 2 hours ago, mild nausea, no focal deficits. Score: 88.

*Layer 2 activates (gap = 4 points, below 10-point threshold).*

**AI reasoning:**

*Patient A (chest pain, suspected NSTEMI):*
- +20 min: Second troponin draw completes. No direct treatment window closes with CT delay for NSTEMI without ST elevation.
- +40 min: Still within 2-hour ECG/troponin evaluation window. No intervention eligibility changes.
- +60 min: Patient remains stable. NSTEMI risk-stratification window (HEART Pathway) is hours, not minutes.

*Patient B (sudden-onset headache):*
- +20 min: If this is SAH — aneurysm remains unsecured. Rebleed risk begins at ~3–4% daily. CT within 25 min is the AHA target.
- +40 min: Rebleed probability is the dominant risk. "Up to 50% of aneurysmal SAH is initially misdiagnosed as migraine." Delayed CT = delayed diagnosis = delayed neurosurgical consultation.
- +60 min: If subarachnoid blood is present, each hour without aneurysm securing increases rebleed risk. Any thunderclap headache must be imaged emergently.

**AI decision:** Patient B is prioritized. The consequence asymmetry is decisive: Patient A's 60-minute delay changes very little; Patient B's 60-minute delay could represent the difference between a diagnosed and unsecured aneurysm with rebleed potential vs. an aneurysm identified and en route to neurosurgical management.

**Source:** Neuro Chunk 1, SAH: "Untreated aneurysmal SAH: 30-day mortality 45%. Rebleed increases mortality to 60–80%. Every hour of delay in aneurysm securing increases rebleed risk by approximately 3–4% daily."

---

### 5.3 Confidence Flagging (Incomplete Data)

When Layer 2 activates for incomplete data, the output includes a confidence flag:

| Flag | Meaning |
|---|---|
| 🟢 HIGH CONFIDENCE | Complete data; deterministic score reliable |
| 🟡 MODERATE CONFIDENCE | Vitals missing but chief complaint fully mapped; AI reasoning applied |
| 🔴 LOW CONFIDENCE | Chief complaint vague; vitals missing; score may not reflect true urgency |

**Clinical implication:** Low-confidence patients should be flagged for immediate re-assessment by clinical staff. The model does not override clinical judgment — it surfaces gaps for human review.

---

## Part 6: Output Format

### 6.1 Dual Queue View

**Left Column (Current State):** Timestamp order. First-in, first-out. No clinical weighting.

**Right Column (AI-Reordered Queue):** Sorted by urgency score, highest to lowest. Each entry includes:

```
PRIORITY RANK: #1
Patient ID: [Anonymous]
Urgency Score: 130
ESI Level: 2 (Emergent)
Imaging Ordered: CT Head + CTA (no contrast for NCCT; contrast for CTA)
Condition: Suspected Ischemic Stroke / LVO

RATIONALE:
This patient has experienced sudden onset left arm weakness and facial droop for 45 minutes. 
Based on AHA/ASA guidelines, the door-to-CT target for suspected stroke is 25 minutes. 
The treatment window for IV tPA closes at 4.5 hours from symptom onset; for mechanical 
thrombectomy, at 6 hours. Every 15-minute delay in tPA administration reduces the number 
of patients who benefit. Approximately 1.9 million neurons are being lost every minute 
without reperfusion.

This patient's imaging cannot safely wait behind lower-acuity patients in a 
first-come, first-served queue.

Confidence: 🟢 HIGH CONFIDENCE
```

### 6.2 Rationale Language Standards

All rationale must meet three criteria:
1. **Clinically accurate** — All numbers and windows drawn directly from source documents
2. **Plain-language accessible** — Legible to both clinical staff and non-clinical hospital leadership
3. **Specific, not generic** — Rationale must reference this patient's specific condition, imaging type, and time window — not a generic "this patient is sicker"

---

## Part 7: Hard Constraints

1. **Every scoring weight is traceable to a project file.** The citation appears in this document next to each component.

2. **The deterministic layer makes the primary call.** Layer 2 is a resolution mechanism, not a replacement.

3. **No clinical decision is a black box.** Every output includes plain-language rationale that any staff member can audit.

4. **This is a proof-of-concept advocacy tool.** It is not a clinical decision support system and does not replace physician or nursing judgment. Its purpose is to demonstrate what a smarter queue could look like.

5. **The model does not deprioritize any patient.** It reorders a fixed set of patients. No patient is moved to "never" — only to a more clinically appropriate position relative to others.

---

## Part 8: Data Structure and Technical Notes

### 8.1 Input Schema (per patient)

```json
{
  "patient_id": "string (anonymized)",
  "timestamp_arrived": "ISO 8601",
  "timestamp_imaging_ordered": "ISO 8601",
  "esi_level": 1-5,
  "age": 0-120,
  "sex": "M/F",
  "arrived_by_ambulance": true/false,
  "chief_complaints": ["string", "string"],
  "vitals": {
    "temp_f": 98.6,
    "hr": 88,
    "rr": 18,
    "sbp": 195,
    "dbp": 110,
    "spo2": 97,
    "pain": 2
  },
  "imaging_ordered": {
    "ct_any": true/false,
    "ct_head": true/false,
    "ct_chest": true/false,
    "ct_ab_pelvis": true/false,
    "ct_contrast": true/false,
    "mri": true/false,
    "mri_contrast": true/false,
    "ultrasound": true/false,
    "xray": true/false
  },
  "data_completeness": "complete/partial/incomplete"
}
```

### 8.2 Score Calculation (pseudocode)

```
function calculateScore(patient):
    c1 = ESI_BASE_SCORES[patient.esi_level]
    
    conditions = mapChiefComplaintToCondition(patient.chief_complaints)
    c2 = max([CONDITION_TIERS[c] for c in conditions])  // highest applicable tier
    
    c3 = max([IMAGING_SCORES[img] for img in patient.imaging_ordered if true])
    
    c4 = calculateVitalSignScore(patient.vitals, patient.age)
    c4 = min(c4, 35)  // cap at 35
    
    c5 = PAIN_SCORES[patient.vitals.pain]
    
    c6 = 5 if patient.arrived_by_ambulance else 0
    
    c7 = AGE_SCORES[patient.age]
    
    return c1 + c2 + c3 + c4 + c5 + c6 + c7

function rankQueue(patients):
    scored = [(p, calculateScore(p)) for p in patients]
    
    // Check for tiebreakers
    sorted_scored = sort(scored, key=score, descending=True)
    
    for i in range(len(sorted_scored) - 1):
        if abs(sorted_scored[i].score - sorted_scored[i+1].score) <= 10:
            // Activate Layer 2 AI reasoning for this pair
            decision = AIReasoningLayer(sorted_scored[i], sorted_scored[i+1])
            // Swap if AI decides lower-scored patient should come first
    
    return sorted_scored
```

---

## Part 9: Source Citation Index

Every scoring decision in this model is traceable to one of the following project files:

| Decision | Source Document |
|---|---|
| ESI base scores (C1) | Esi_Handbook.pdf — Chapters 2, 3, 5 |
| ESI decision points A, B, C, D | Esi_Handbook.pdf — Chapters 3, 4, 6 |
| ESI danger zone vital signs (C4 thresholds) | Esi_Handbook.pdf — Chapter 6, Figure 6-1 |
| ESI pain score guidance (C5) | Esi_Handbook.pdf — Chapters 3, 4 |
| ESI arrival mode guidance (C6) | Esi_Handbook.pdf — Chapter 3, p. 24 |
| ESI pediatric fever / age rules (C7, pediatric) | Esi_Handbook.pdf — Chapter 6, Table 6-3 |
| Input variable structure | doc21-ed-508.pdf — Section II Codebook |
| NHAMCS triage distribution | doc21-ed-508.pdf — Section III, p. 93 |
| Neurological condition time-sensitivity (C2) | ER_Imaging_Optimization_Neuro_Chunk1.pdf |
| ENT/ophthalmology condition time-sensitivity (C2) | ER_Imaging_Optimization_ENT_Cardio_Chunk2.pdf |
| Cardiovascular condition time-sensitivity (C2) | ER_Imaging_Optimization_ENT_Cardio_Chunk2.pdf |
| GI condition time-sensitivity (C2) | ER_Imaging_Optimization_GI_Chunk3.pdf |
| Pulmonary condition time-sensitivity (C2) | ER_Imaging_Optimization_Pulm_Repro_Infect_Chunk4.pdf |
| Reproductive/urological condition time-sensitivity (C2) | ER_Imaging_Optimization_Pulm_Repro_Infect_Chunk4.pdf |
| Infectious/septic condition time-sensitivity (C2) | ER_Imaging_Optimization_Pulm_Repro_Infect_Chunk4.pdf |
| Trauma/oncological condition time-sensitivity (C2) | ER_Imaging_Optimization_Trauma_Oncology_Chunk5.pdf |
| Stroke door-to-CT / door-to-needle benchmarks | powersetal2019guidelinesfortheearlymanagementofpatientswithacuteischemicstroke2019updatetothe2018.pdf |
| Stroke imaging urgency / CTA/NCCT protocols | powersetal2019guidelines... + prabhakaran-et-al-2026... |
| HEART Pathway / chest pain risk stratification | mahleretal2018safelyidentifyingemergencydepartmentpatientswithacutechestpainforearlydischarge.pdf |
| ED boarding / crowding context | swartz2016emergencydepartmentboardingnowhereelsetogo.pdf; SISTER.pdf; Factors_affecting_emergency_department_crowding.pdf |

---

## Part 10: What This Model Does Not Do

**It does not diagnose.** It does not assign a diagnosis. It reasons about time-sensitivity based on the most clinically probable interpretation of available inputs.

**It does not replace triage.** Triage (ESI assignment) has already happened. This model works downstream of triage, in the imaging queue specifically.

**It does not address imaging appropriateness.** The model assumes imaging has already been ordered by a physician. It does not evaluate whether imaging is indicated — only who should receive their already-ordered imaging first.

**It does not account for contrast prep time.** Patients requiring contrast take longer. This is a logistical factor for the technical implementation, not a scoring factor.

**It does not incorporate scanner availability.** The model outputs a single ranked queue. Parallel scanners and logistics are implementation-layer decisions.

---

*This model is complete and ready for technical specification and frontend build.*  
*All clinical reasoning is sourced from project files. No general knowledge was used where the files provided an answer.*
