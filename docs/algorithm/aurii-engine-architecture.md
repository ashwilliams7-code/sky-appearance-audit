# Aurii Analysis Engine Architecture

> **Status:** parked engineering blueprint for the Aurii face + body analysis engine. This is not a claim about QOVES' proprietary internals; it is a public-inferred, differentiated Aurii architecture.

## 1. Product definition

Aurii should be a private appearance-strategy engine, not an attractiveness rater.

The engine's job is to turn a user's uploaded photo set and goals into a structured, calm, non-medical plan that answers:

1. What can be observed reliably from the supplied photos?
2. Which face, body, posture, grooming, styling, and photo-presence factors seem most actionable?
3. Which actions have the highest likely return for this person, given confidence, effort, risk, cost, reversibility, and the user's goals?
4. What should be deferred because the input quality is weak, the topic is medical/clinical, or confidence is too low?

The premium output is a ~30-page private report with clear evidence, limitations, ranked priorities, and a staged plan.

## 2. Core principles

- **Non-medical first:** Aurii does not diagnose, prescribe, or imply clinical certainty.
- **No public beauty score:** The system may compute internal normalized metrics, but the user-facing product is not a leaderboard or attractiveness verdict.
- **Confidence-aware:** Every observation includes confidence, inputs used, limitations, and whether retakes are needed.
- **Actionability over judgement:** Recommendations are ranked by likely personal ROI, not by how harshly something can be described.
- **Face + body together:** Face, body, posture, clothing fit, and photo capture are treated as one presentation system.
- **Deterministic where possible:** Measurements, geometry, and capture checks should be computed with repeatable CV methods before LLM reasoning.
- **LLM as explainer/orchestrator, not sole judge:** Multimodal models can interpret and phrase findings, but structured metrics and guardrails control what they are allowed to claim.
- **Privacy by design:** Strip metadata, isolate sessions, encrypt storage, and support deletion from day one.

## 3. System overview

```text
User intake + consent
  -> secure upload session
  -> photo-set quality gate
  -> image normalization + redaction
  -> face detection / landmarks
  -> body pose / silhouette / clothing-readability checks
  -> metric registry execution
  -> confidence + limitation aggregation
  -> goal/context matching
  -> recommendation ranking
  -> report composition schema
  -> designed web/PDF report
  -> optional retake/progress loop
```

Aurii should never jump straight from raw images to narrative text. The engine is a sequence of modules that each emit structured JSON.

## 4. Required photo set

### Minimum MVP set

- Face front: neutral expression, eye-level, natural light.
- Face smile: eye-level.
- Face side/profile: left or right.
- Face 45-degree: optional but useful.
- Full body front: fitted clothing, neutral posture, camera at mid-torso height.
- Full body side: neutral posture.

### Ideal v1 set

- Front face, smile, left profile, right profile, 45-degree left/right.
- Hairline/framing photo if hair analysis is included.
- Full body front, side, back, and natural candid/standing photo.
- One preferred dating/professional profile photo for photo-presence analysis.
- Optional self-stated goals: dating, professional presence, fitness/posture, skin/hair/grooming, wardrobe, non-surgical only, open-to-clinical-discussion.

## 5. Module map

### 5.1 Intake and consent module

**Purpose:** capture boundaries before analysis.

Inputs:
- user goals;
- gender/presentation preference if voluntarily provided;
- age range if voluntarily provided;
- medical boundary preference;
- budget/time constraints;
- upcoming event deadline;
- consent, privacy, deletion preference.

Outputs:
- `intake_profile` JSON;
- allowed recommendation categories;
- forbidden recommendation categories;
- tone and report emphasis.

Notes:
- The engine should never infer sensitive identity attributes unless the user explicitly supplies them and they are necessary.
- If a request implies diagnosis, dysmorphia spiralling, or unsafe self-harm framing, the engine switches to supportive language and refuses clinical certainty.

### 5.2 Capture-quality gate

**Purpose:** decide whether the supplied photo set is analyzable.

Checks:
- resolution and aspect ratio;
- blur/sharpness;
- under/overexposure;
- face visibility;
- body visibility;
- occlusion from hair, glasses, hands, hats, heavy filters;
- camera angle and lens distortion;
- head yaw/pitch/roll;
- posture neutrality;
- full-body crop completeness;
- clothing fit/readability;
- duplicate or near-duplicate images;
- EXIF removal status.

Outputs:
- pass / conditional pass / retake required;
- per-image quality scores;
- retake instructions;
- list of modules allowed to run confidently.

Example decisions:
- If face landmarks are reliable but body photo is cropped, face modules can proceed while body modules request retakes.
- If lighting makes skin texture unreliable, Aurii may discuss capture quality and grooming/photo direction but should not assert skin quality.

### 5.3 Image normalization module

**Purpose:** create consistent inputs for downstream analysis.

Operations:
- strip metadata;
- generate safe internal filenames;
- downscale to analysis resolutions;
- preserve originals in encrypted storage if required;
- create thumbnails for report rendering;
- align face crop where possible;
- detect and record transformations.

Outputs:
- normalized image manifests;
- derivative paths/IDs;
- hashes for duplicate detection;
- transformation logs.

### 5.4 Face geometry module

**Purpose:** deterministic face-landmark and proportion extraction.

Candidate stack:
- MediaPipe Face Landmarker;
- OpenCV;
- optional 3D face mesh normalization;
- internal metric registry.

Metric families:
- capture: yaw, pitch, roll, lens/crop risk;
- thirds/fifths approximation;
- symmetry deltas where landmark confidence is high;
- jaw/chin/cheek contour descriptors;
- eye/brow/nose/lip relative placement descriptors;
- facial width/height and face-shape descriptors;
- smile/resting expression delta;
- facial-hair/hairline framing if visible;
- skin/complexion readability, not diagnosis.

Important: user-facing language should be descriptive and directional. Avoid saying a feature is objectively bad. Prefer: “The strongest high-confidence lever appears to be hair/framing and photo angle before any clinical consideration.”

### 5.5 Skin, hair, grooming, and presentation cue module

**Purpose:** identify non-medical presentation levers.

Possible cues:
- lighting and skin texture visibility;
- shine/flatness caused by lighting;
- beard/facial-hair boundary clarity;
- haircut framing vs face shape;
- eyebrow/hair contrast and grooming clarity;
- color contrast with clothing/background;
- camera distance and expression.

Guardrail:
- Do not diagnose acne, rosacea, hair loss conditions, or dermatological disorders.
- If a clinical topic is likely, mark it as “consult qualified professional if this concerns you.”

### 5.6 Body posture and frame module

**Purpose:** analyze full-body presence from readable images.

Candidate stack:
- MediaPipe Pose / BlazePose;
- segmentation/silhouette model;
- OpenCV geometric checks;
- optional clothing/fit classifier.

Metric families:
- posture line: head-forward, shoulder slope, shoulder rotation, hip alignment where visible;
- stance and camera angle reliability;
- shoulder/waist/hip silhouette descriptors;
- limb visibility and crop completeness;
- clothing fit and proportion effects;
- muscle/fat composition cues only as low-certainty, non-clinical visual descriptors;
- style/cut recommendations.

Guardrail:
- Do not estimate body-fat percentage as a medical or precise value from photos.
- Do not shame body shape. Translate observations into practical presentation levers: posture, fit, training emphasis, tailoring, photo angle.

### 5.7 Clothing, fit, and style module

**Purpose:** capture high-ROI changes that can shift presentation immediately.

Cues:
- shoulder seam placement;
- garment length;
- waist suppression / boxiness;
- neckline and face framing;
- color contrast with skin/hair;
- pattern scale;
- shoe/trouser silhouette if full-body visible.

Outputs:
- fit observations;
- wardrobe priority list;
- photo-ready outfit guidance;
- “cheap win” recommendations.

### 5.8 Photo-presence module

**Purpose:** optimize dating/professional image outcomes.

Cues:
- lens distance and distortion risk;
- eye contact;
- expression warmth;
- crop/framing;
- background distraction;
- lighting direction;
- posture and body language;
- photo ordering for dating/professional sets.

Outputs:
- current photo strengths;
- retake recipe;
- dating/professional profile ordering;
- shot list.

### 5.9 Reasoning layer

**Purpose:** synthesize structured metrics into human-readable insight.

The reasoning layer receives only sanitized structured outputs and thumbnails. It should:
- respect confidence levels;
- cite the modules that support each claim;
- avoid unsupported medical/clinical conclusions;
- convert measurements into plain-English implications;
- separate “observed” from “recommended.”

LLM prompts should require JSON first, narrative second. The report composer should reject unsupported claims.

### 5.10 Recommendation ranking engine

**Purpose:** prioritize what to do first.

Aurii recommendations are ranked by estimated ROI:

```text
priority_score =
  impact
  × confidence
  × user_goal_relevance
  × reversibility_bonus
  × compounding_bonus
  × readiness
  ÷ (effort + cost + risk + time_to_visible_result)
```

Categories:
- instant photo/capture wins;
- grooming/hair/skin routine wins;
- clothing/fit wins;
- posture/mobility/training wins;
- professional consult prompts, only when user allows and guardrails permit;
- retake requests when confidence is too low.

Ranking behavior:
- High-confidence, low-risk, reversible actions go first.
- Low-confidence observations become retake guidance, not recommendations.
- Clinical suggestions are never framed as instructions.
- The engine should include “do not prioritize” items to prevent unnecessary spending.

### 5.11 Report composer

**Purpose:** turn structured analysis into a premium, designed report.

Report layers:
- executive summary;
- photo quality and confidence;
- face read;
- body/posture read;
- grooming/style/photo read;
- ranked action plan;
- 7-day / 30-day / 90-day roadmap;
- retake/progress tracking instructions;
- safety and limitations appendix.

The composer should use a JSON report schema so web and PDF renderers stay consistent.

### 5.12 Privacy and storage module

Requirements:
- secure upload URLs;
- metadata stripping;
- encryption at rest;
- short-lived signed derivative URLs;
- deletion job and audit trail;
- separated user identity and image-analysis IDs;
- no training on user photos without explicit opt-in.

## 6. Module output contract

Every module should emit this shape:

```json
{
  "module": "face_geometry",
  "version": "0.1.0",
  "status": "pass",
  "confidence": 0.82,
  "inputs": ["image_face_front_001"],
  "observations": [
    {
      "id": "face.capture.yaw",
      "label": "Head angle",
      "value": 3.1,
      "unit": "degrees",
      "confidence": 0.91,
      "evidence": ["landmark_set_front"],
      "limitations": []
    }
  ],
  "recommendation_candidates": [
    {
      "id": "photo.retake.eye_level_front",
      "category": "photo_capture",
      "claim": "A sharper eye-level front image would improve face measurement confidence.",
      "confidence": 0.88,
      "impact_estimate": 0.55,
      "effort": 0.15,
      "risk": 0.02
    }
  ],
  "limitations": ["Skin texture confidence reduced by side lighting."],
  "requires_retake": false
}
```

## 7. Confidence model

Use layered confidence rather than a single global “AI confidence.”

Confidence dimensions:
- image quality confidence;
- landmark detection confidence;
- module-specific confidence;
- cross-photo consistency;
- recommendation evidence strength;
- user-goal relevance.

User-facing confidence tiers:
- **High confidence:** supported by clear image quality and repeatable metrics.
- **Moderate confidence:** useful but affected by angle, lighting, clothing, or limited views.
- **Low confidence:** do not present as a finding; convert into retake guidance.

## 8. Safety and tone guardrails

Aurii should refuse or reframe:
- requests for a public attractiveness score;
- harsh ranking against other people;
- medical diagnosis from images;
- claims about protected/sensitive attributes;
- certainty about procedures or treatment outcomes;
- extreme self-criticism loops.

Preferred language:
- “The strongest lever appears to be…”
- “From these photos, confidence is moderate because…”
- “This is a presentation/photo/grooming lever, not a medical conclusion.”
- “A qualified professional is the right path if this concerns your health.”

Avoid:
- “defective,” “ugly,” “abnormal,” “fix your face/body,” “guaranteed.”

## 9. Build phases

### Phase 0 — parked blueprint

- Maintain canonical docs.
- Define schemas.
- Decide MVP photo set.
- Keep investor site truthful: prototype UI, not working diagnosis.

### Phase 1 — offline prototype

- Local Python package for upload manifest processing.
- Quality gate using OpenCV.
- Face landmarks with MediaPipe.
- Pose landmarks with MediaPipe.
- Static JSON report generation.
- Render a sample web/PDF report from fixture images.

### Phase 2 — MVP analysis API

- Secure upload session.
- Async job queue.
- Module outputs stored as JSON.
- Recommendation ranking v0.
- Report schema renderer.
- Manual review fallback for low-confidence cases.

### Phase 3 — premium v1

- Better calibration across diverse capture conditions.
- Progress comparison across time.
- Clinic-facing version with strict disclaimers.
- Report designer and PDF export.
- Human-in-the-loop quality review option.

### Phase 4 — defensibility

- Proprietary metric registry and evaluation dataset.
- Retake feedback loop.
- Longitudinal progress insights.
- Expert-authored recommendation library.
- Privacy/security certifications if the product matures.

## 10. Open decisions

1. MVP implementation language: Python service vs TypeScript full-stack.
2. Initial report renderer: static HTML-to-PDF vs app-native report page.
3. Upload/storage provider.
4. Whether concierge review includes human expert QA.
5. Which recommendations are allowed in beta: non-surgical only vs consult prompts.
6. Whether the engine launches as a standalone backend repo or as a subdirectory here until product direction is stable.

## 11. Recommended repository structure when implementation begins

```text
engine/
  pyproject.toml
  aurii_engine/
    pipeline.py
    config.py
    privacy.py
    modules/
      quality_gate.py
      face_geometry.py
      body_pose.py
      style_fit.py
      photo_presence.py
      ranking.py
      report_composer.py
    schemas/
      module_output.schema.json
      report.schema.json
    prompts/
      reasoning_system.md
      report_writer.md
  tests/
    fixtures/
    test_quality_gate.py
    test_ranking.py
    test_report_schema.py
reports/
  sample-report-output.json
  sample-report.html
```

For now, this repository parks the architecture and schemas under `docs/algorithm/` so the direction is available to investors/builders without pretending the engine is complete.
