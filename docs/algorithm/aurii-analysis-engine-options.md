# Aurii Face + Body Analysis Engine Options

> **For Hermes:** Use `subagent-driven-development` if this moves from parked strategy into implementation. Treat this as the canonical repo parking doc for the Aurii algorithm/engine direction.

**Goal:** Define how Aurii can produce a premium ~30-page private face + body appearance strategy report from user-uploaded photos.

**Architecture:** Aurii should be a multi-stage analysis pipeline, not one giant AI prompt. The engine should combine photo-quality validation, deterministic computer-vision measurements, multimodal reasoning, safety/medical guardrails, a recommendation-ranking model, and a designed report composer.

**Tech Stack Candidates:** MediaPipe Face Landmarker, MediaPipe Pose/BlazePose, OpenCV, segmentation models, multimodal LLMs, a metric registry, JSON report schemas, and a PDF/report renderer.

**Status:** Parked for future build. This doc captures the current algorithm dive and viable engine options so the direction is not lost.

---

## 1. Product Thesis

Aurii is not “AI rates your face.” That positioning is shallow, risky, and easy to copy.

Aurii should be framed as:

> **A private appearance strategy engine that ranks the highest-ROI improvements across face, body, posture, grooming, style, and photo presence.**

The investor-facing wedge is:

- QOVES-like analytical seriousness for face.
- Expanded body/posture/frame analysis.
- Practical transformation plan rather than beauty scoring.
- Non-medical, private, confidence-building tone.
- Premium 30-page report output.

We cannot know or claim QOVES' proprietary algorithm. We can build from public appearance-science concepts and create a differentiated face + body engine.

---

## 2. Likely QOVES-Style Algorithm Pattern — Public-Inferred, Not Proprietary

A QOVES-like system likely feels strong because it turns subjective appearance into a structured analytical flow:

1. Standardised photo capture.
2. Landmark/measurement extraction.
3. Proportion and harmony interpretation.
4. Feature-specific commentary.
5. Aesthetic frameworks such as facial harmony, symmetry, averageness, sexual dimorphism/presence, youth/health cues, and fixability.
6. Personalised recommendations.
7. Report presentation that feels clinical and ordered.

Aurii should replicate the *shape* of that intelligence without claiming access to their internals.

Aurii improves on it by adding:

- body and posture analysis;
- clothing/fit analysis;
- photo/dating/professional presence analysis;
- ROI-ranked recommendation logic;
- progress tracking over time;
- clearer non-medical guardrails.

---

## 3. Core Pipeline

```text
Photo upload
  -> capture-quality gate
  -> face detection + landmarks
  -> body pose + silhouette analysis
  -> visual cue classifiers
  -> context/persona intake
  -> feature metric registry
  -> reasoning layer
  -> recommendation-ranking engine
  -> report schema
  -> designed PDF / web report
```

The important engineering principle: every module should emit structured JSON with confidence values and limitations. The report writer should never invent certainty that the extraction layer did not provide.

---

## 4. Required Photo Set

### Minimum Face Set

- Front neutral, eye-level, no filter.
- Front natural smile.
- Left profile.
- Right profile.
- 45-degree angle.
- Natural light / window shade.
- Camera 1.5–2m away to reduce lens distortion.

### Minimum Body Set

- Full-body front, neutral stance.
- Full-body side.
- Optional full-body back.
- Optional relaxed candid/dating/photo examples.
- Clothing should show silhouette without being revealing: fitted tee, fitted pants/shorts, neutral shoes.

### Optional Context Inputs

- Goal: dating, professional, social, fitness, general glow-up.
- Desired impression: masculine, polished, approachable, athletic, refined, softer, sharper, etc.
- Constraints: budget, time, no procedures, skin sensitivity, religious/modesty/style boundaries.

---

## 5. Module A — Capture Quality Gate

This gate protects Aurii from bad photos creating fake flaws.

### Checks

- Face detected.
- Body detected.
- Multiple people present.
- Head yaw/pitch/roll.
- Camera distance/lens distortion risk.
- Lighting exposure and harsh shadow.
- Blur/sharpness.
- Expression neutrality.
- Eye visibility.
- Hair/hat/glasses obstruction.
- Body pose neutrality.
- Clothing obstruction.
- Full body not cropped.
- Shot comparability across before/after or multi-view sets.

### Output Example

```json
{
  "photo_quality": {
    "overall": "usable_with_limitations",
    "retakes_required": ["right_profile", "full_body_side"],
    "warnings": [
      "front_face appears too close to camera; jaw and nose proportions may be distorted",
      "body_front has uneven stance; shoulder symmetry confidence reduced"
    ],
    "confidence": 0.72
  }
}
```

### Investor Message

Aurii should visibly show this step in the product. It communicates scientific seriousness and reduces liability.

---

## 6. Module B — Face Measurement Engine

### Candidate Extractors

- MediaPipe Face Landmarker / Face Mesh.
- OpenCV face detection for fallback.
- Multimodal LLM for qualitative cues.
- Optional future: custom model trained on annotated reports.

### Metric Families

#### Global Face Shape

- Face height/width ratio.
- Upper/mid/lower facial thirds.
- Facial fifths estimate.
- Cheekbone width estimate.
- Jaw width estimate.
- Chin width/length estimate.
- Facial taper and lower-face balance.

#### Eyes/Brows

- Eye spacing relative to eye width.
- Eye openness.
- Eye tilt estimate.
- Upper eyelid exposure.
- Under-eye shadow cues.
- Brow height and asymmetry.
- Brow density/framing.

#### Nose/Midface

- Nose width relative to face width.
- Nose length relative to face height.
- Bridge/nostril visibility cues.
- Nose centre offset as pose/asymmetry warning.
- Midface length cues.

#### Mouth/Smile

- Mouth width relative to face width.
- Philtrum/lip/chin balance.
- Smile symmetry.
- Teeth visibility and colour cues.
- Lip fullness relative to surrounding features.

#### Jaw/Chin/Neck

- Jaw-to-cheek relationship.
- Chin projection cue from profile.
- Neck-jaw separation.
- Submental fullness cue.
- Head posture effect on jawline.

#### Skin/Hair/Grooming

- Skin texture/clarity cue.
- Redness/uneven tone cue.
- Under-eye darkness.
- Hair density/framing.
- Beard line, neckline, cheek line, density, and bulk.
- Grooming consistency.

### Output Example

```json
{
  "face_metrics": {
    "face_height_width_ratio": {"value": 1.42, "confidence": 0.81},
    "eye_spacing_ratio": {"value": 1.03, "confidence": 0.77},
    "jaw_to_cheek_ratio": {"value": 0.83, "confidence": 0.68},
    "nose_width_to_face_width": {"value": 0.24, "confidence": 0.74},
    "notes": ["front image usable but mild head rotation reduces symmetry confidence"]
  }
}
```

---

## 7. Module C — Body + Posture Engine

This is the main Aurii differentiator.

### Candidate Extractors

- MediaPipe Pose / BlazePose.
- Human segmentation model for silhouette.
- OpenCV contour/silhouette analysis.
- Optional future: DensePose or higher-fidelity body landmarking.
- Multimodal LLM for clothing/fit/presence reasoning.

### Metric Families

#### Frame + Proportion

- Shoulder width / waist visual ratio estimate.
- Torso-to-leg visual ratio.
- Head-to-body proportion cue.
- Neck length/readability.
- Upper-body vs lower-body balance.
- Visible muscularity/softness cues, carefully non-diagnostic.

#### Posture

- Forward head posture cue.
- Shoulder elevation/slope/asymmetry.
- Rounded shoulders cue.
- Pelvic tilt indicator from side view.
- Knee/ankle/stance alignment cue.
- Relaxed vs tense stance.

#### Silhouette + Fit

- Clothing fit: too loose/tight/boxy/unclear.
- Waist definition visibility.
- Shoulder line readability.
- Trouser length and shoe balance.
- Colour contrast and vertical line.

#### Presence

- Does the person read composed, athletic, relaxed, guarded, slouched, tense, narrow, broad, polished?
- Is the pose helping or hurting first impression?
- Are camera angle and lens height distorting body proportions?

### Output Example

```json
{
  "body_metrics": {
    "shoulder_waist_visual_ratio": {"value": 1.42, "confidence": 0.61},
    "forward_head_posture_cue": {"value": "mild", "confidence": 0.66},
    "shoulder_asymmetry_cue": {"value": "low_to_moderate", "confidence": 0.58},
    "clothing_fit_signal": {"value": "silhouette_partly_obscured", "confidence": 0.72},
    "notes": ["front body image is usable but clothing reduces body-composition confidence"]
  }
}
```

---

## 8. Module D — Context + Goal Engine

The same measurements can imply different advice depending on user goals.

### Inputs

- Primary goal: dating, professional, social media, general, fitness.
- Desired impression: masculine, refined, warm, approachable, sharp, athletic, elegant, youthful.
- Budget range.
- Time horizon: 7 days, 30 days, 90 days, 12 months.
- Non-negotiables: no procedures, no hair dye, modest fashion, no gym, etc.

### Output Example

```json
{
  "context": {
    "primary_goal": "dating_profile",
    "desired_impression": ["masculine", "polished", "approachable"],
    "constraints": ["low_budget", "non_surgical_first"],
    "recommendation_bias": "prioritise grooming, photo protocol, posture, fit, skin basics"
  }
}
```

---

## 9. Recommendation Ranking Algorithm

The recommendation engine is the commercial moat. Observations alone are cheap. Prioritisation is valuable.

### Base Formula

```text
Priority Score = (Impact × Fixability × Confidence × Safety × Goal Relevance) ÷ (Effort × Cost × Risk)
```

### Factors

- **Impact:** visible difference if improved.
- **Fixability:** how practical/reversible the change is.
- **Confidence:** how confident the engine is from the photo set.
- **Safety:** non-medical, low-risk recommendations rank higher.
- **Goal relevance:** tied to dating/professional/social/body goals.
- **Effort:** time and behavioural load.
- **Cost:** money required.
- **Risk:** health/social/aesthetic downside.

### Recommendation Tiers

- **Do First:** high impact, reversible, low cost/risk.
- **Worth Testing:** medium effort, likely visible upside.
- **Professional Optional:** dermatology, dental, cosmetic, physio, trainer, stylist.
- **Ignore:** low-ROI internet nitpicks or unsupported precision.

### Recommendation Object

```json
{
  "recommendation": {
    "id": "photo_protocol_eye_level_soft_window_light",
    "title": "Reshoot at eye level in soft side-window light",
    "category": "photo_presence",
    "tier": "do_first",
    "priority_score": 8.7,
    "impact": 8,
    "fixability": 10,
    "confidence": 9,
    "safety": 10,
    "goal_relevance": 9,
    "effort": 2,
    "cost": 1,
    "risk": 1,
    "why_it_matters": "The current photo angle exaggerates lower-face and nose proportions, reducing confidence in the analysis and making presentation harsher than reality.",
    "action_steps": [
      "Stand 1.5–2m from camera",
      "Keep lens at eye level",
      "Use soft window light from 30–45 degrees",
      "Take neutral and relaxed-smile versions"
    ]
  }
}
```

---

## 10. Report Composer — 30-Page Structure

The report should be modular. Each page pulls from structured JSON and renders into a designed template.

1. Cover / private report identity.
2. Scope and non-medical disclaimer.
3. Executive summary.
4. Top 5 highest-ROI upgrades.
5. Photo quality audit.
6. Retake instructions.
7. Face harmony overview.
8. Facial proportion snapshot.
9. Symmetry and expression caveats.
10. Eye and brow area.
11. Nose and midface.
12. Mouth, lips, smile, teeth cues.
13. Jaw, chin, neck.
14. Skin, under-eyes, complexion.
15. Hair and grooming.
16. Full-body frame snapshot.
17. Posture and alignment.
18. Shoulder/waist/torso-leg visual proportions.
19. Clothing fit and silhouette.
20. Photo presence and camera strategy.
21. Dating/profile direction.
22. Professional/headshot direction.
23. Social media/candid direction.
24. Strengths to preserve.
25. Bottlenecks to reduce.
26. Priority matrix.
27. 7-day action plan.
28. 30-day action plan.
29. 90-day strategy.
30. Progress tracking and next scan protocol.

---

## 11. Engine Options

### Option 1 — Fast MVP: Multimodal LLM + Basic CV

**Use when:** Investor demo, early prototype, fastest path to believable reports.

**Flow:**
- Use photo-quality heuristics and simple landmarks.
- Send validated photo set + structured prompt to a strong multimodal LLM.
- LLM emits structured JSON for face/body observations and recommendations.
- Renderer turns JSON into report.

**Pros:**
- Fastest to build.
- Good investor demo.
- Flexible language and reasoning.

**Cons:**
- Less deterministic.
- Harder to ensure consistency.
- Needs careful guardrails to avoid overclaiming.

**Best first milestone:** Build a working report generator with repeatable JSON schema and PDF output.

---

### Option 2 — Deterministic CV Engine + Rule-Based Expert System

**Use when:** We need consistency, credibility, and measurable repeatability.

**Flow:**
- MediaPipe Face Landmarker for face ratios.
- MediaPipe Pose for body posture and proportions.
- OpenCV for quality/blur/exposure/cropping.
- Segmentation for silhouette and clothing-fit cues.
- Rule engine maps metrics into observations.
- LLM only writes final prose from structured facts.

**Pros:**
- More defensible.
- Easier to debug.
- Better for before/after tracking.
- Can show investors real measurement outputs.

**Cons:**
- More engineering.
- Still needs good thresholds and validation.
- Body metrics from photos can be noisy.

**Best first milestone:** Metric registry + quality gate + face/body landmarks for a known photo set.

---

### Option 3 — Hybrid Engine: Deterministic Metrics + LLM Reasoning + Human Review Layer

**Use when:** Best near-term product quality.

**Flow:**
- Deterministic extractors produce metrics and confidence.
- LLM interprets only the allowed metric set and visible cues.
- Recommendation ranker scores action items.
- Human expert review available for premium tier.
- Reports improve over time from reviewer corrections.

**Pros:**
- Best balance of speed and credibility.
- Can launch with high-quality reports.
- Creates data for future training.
- Allows premium/human-reviewed tier.

**Cons:**
- Requires workflow tooling.
- Human review costs money.
- Needs privacy operations.

**Best first milestone:** Automated draft report + manual review checklist.

---

### Option 4 — Long-Term Moat: Proprietary Dataset + Learned Recommendation Model

**Use when:** After enough reports and user outcomes exist.

**Flow:**
- Collect consented before/after datasets.
- Track recommendations and user outcomes.
- Build learned scoring models for improvement potential and recommendation effectiveness.
- Personalise by goal, demographics, culture, age, and constraints.

**Pros:**
- True moat.
- Improves over time.
- Harder for competitors to copy.

**Cons:**
- Not MVP.
- Needs data consent, privacy, labels, outcomes.
- Significant operational complexity.

**Best first milestone:** Design data schema now so future learning is possible.

---

## 12. Recommended Path

Start with **Option 3**, implemented in phases:

### Phase 0 — Investor Prototype

- Landing page communicates pipeline.
- Sample report modules show quality gate, face/body map, ROI matrix.
- Local/reference visual assets can be used only for internal understanding; public site should use owned/generated media.

### Phase 1 — Working MVP

- Photo upload intake.
- Quality gate.
- Basic face landmarks.
- Basic pose/body extraction.
- Structured JSON output.
- LLM-generated report draft.
- Designed PDF renderer.

### Phase 2 — Repeatability

- Metric registry.
- Confidence scoring.
- Rule-based recommendation ranking.
- Before/after tracking.
- Test suite with known image fixtures.

### Phase 3 — Premium Product

- Human review workflow.
- Clinic/professional referrals as optional later-tier guidance.
- User dashboard.
- Progress scans.

### Phase 4 — Defensible Moat

- Consented outcome dataset.
- Recommendation effectiveness model.
- Personalisation by user goals and constraints.

---

## 13. Proposed Repo Structure for Future Engine

```text
engine/
  README.md
  pyproject.toml
  aurii_engine/
    intake/
      quality_gate.py
      photo_protocol.py
    extractors/
      face_landmarks.py
      body_pose.py
      image_quality.py
      silhouette.py
    metrics/
      registry.py
      face_metrics.py
      body_metrics.py
    reasoning/
      observation_builder.py
      recommendation_ranker.py
      guardrails.py
    reports/
      schema.py
      composer.py
      templates/
    tests/
      fixtures/
      test_quality_gate.py
      test_recommendation_ranker.py
```

Keep the current GitHub Pages site separate from the engine until the prototype needs a backend.

---

## 14. Guardrails

Aurii must remain:

- non-medical;
- non-diagnostic;
- non-shaming;
- privacy-first;
- transparent about photo limitations;
- careful with body composition, age, ethnicity, gender, and attractiveness language;
- focused on reversible, low-risk recommendations first.

Avoid:

- beauty scores;
- surgical-first advice;
- fake precision;
- claiming proprietary QOVES methodology;
- diagnosing health/body issues from photos.

---

## 15. Parked Next Steps

When we unpark this:

1. Create `engine/` as a Python package.
2. Implement quality gate first.
3. Add MediaPipe face extraction.
4. Add MediaPipe pose extraction.
5. Create a `ReportInput` / `ReportOutput` JSON schema.
6. Implement recommendation-ranker unit tests.
7. Generate one sample 30-page report from fixture photos.
8. Add the report preview back into the landing site.

This doc is the current saved algorithm direction for Aurii. The next build should start here, not from scratch.
