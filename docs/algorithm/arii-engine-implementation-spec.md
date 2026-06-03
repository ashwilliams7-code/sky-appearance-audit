# Arii Engine Implementation Spec

> **Status:** implementation-ready spec parked for the first real Arii engine prototype. Pair this with `arii-engine-architecture.md` and `arii-analysis-engine-options.md`.

## 1. MVP target

Build an offline prototype that accepts a structured photo manifest and produces a report JSON package. The prototype does not need billing, auth, or production upload infrastructure. It must prove the core loop:

```text
photo manifest -> quality gate -> face/body metrics -> recommendation ranking -> report JSON -> sample report render
```

The prototype should be boringly deterministic wherever possible and explicit when it cannot make a claim.

## 2. Proposed stack

### Prototype stack

- Python 3.11+
- OpenCV for image quality, blur, exposure, crop checks.
- MediaPipe Face Landmarker / Face Mesh for face points.
- MediaPipe Pose / BlazePose for posture and body landmarks.
- Pydantic models for contracts.
- JSON Schema export for cross-language compatibility.
- Jinja2 or static HTML templates for sample report render.
- Optional multimodal LLM pass for narrative, constrained by module outputs.

### Production stack options

- Python FastAPI analysis service with async job queue.
- Object storage with signed URLs.
- Postgres for jobs/users/report metadata.
- Frontend report renderer using the same report JSON.
- Human review dashboard for low-confidence/concierge reports.

## 3. Core domain objects

### 3.1 `AnalysisJob`

```json
{
  "job_id": "job_01HX...",
  "user_id": "usr_01HX...",
  "created_at": "2026-06-03T00:00:00Z",
  "intake_profile": {},
  "photo_manifest": {},
  "status": "queued|running|needs_retake|complete|failed",
  "module_outputs": [],
  "report_id": null
}
```

### 3.2 `PhotoManifest`

Each uploaded image must be explicitly typed. Do not infer all photo roles from filenames.

```json
{
  "photos": [
    {
      "photo_id": "img_front_face_001",
      "role": "face_front_neutral",
      "original_filename": "front.jpg",
      "storage_uri": "private://...",
      "mime_type": "image/jpeg",
      "width": 3024,
      "height": 4032,
      "metadata_stripped": true,
      "user_supplied": true
    }
  ]
}
```

Allowed MVP roles:
- `face_front_neutral`
- `face_smile`
- `face_profile_left`
- `face_profile_right`
- `face_three_quarter`
- `body_front`
- `body_side`
- `body_back`
- `profile_photo_current`

### 3.3 `ModuleOutput`

All modules emit the same envelope:

```json
{
  "module": "quality_gate",
  "version": "0.1.0",
  "status": "pass|conditional|needs_retake|failed|skipped",
  "confidence": 0.86,
  "inputs": ["img_front_face_001"],
  "observations": [],
  "recommendation_candidates": [],
  "limitations": [],
  "requires_retake": false
}
```

### 3.4 `Observation`

```json
{
  "id": "face.geometry.yaw_degrees",
  "label": "Head yaw",
  "value": 2.8,
  "unit": "degrees",
  "confidence": 0.93,
  "evidence": ["face_landmarks_front"],
  "limitations": []
}
```

### 3.5 `RecommendationCandidate`

```json
{
  "id": "photo.capture.eye_level_retake",
  "category": "photo_capture",
  "title": "Retake your front face image at eye level",
  "claim": "The current front image is slightly above eye level, reducing measurement confidence.",
  "evidence_observation_ids": ["quality.camera_angle.vertical"],
  "impact_estimate": 0.45,
  "confidence": 0.88,
  "user_goal_relevance": 0.72,
  "effort": 0.12,
  "cost": 0.0,
  "risk": 0.02,
  "time_to_visible_result": 0.05,
  "reversibility": 1.0,
  "priority_score": 0.0,
  "time_horizon": "today|7_days|30_days|90_days|later",
  "guardrail_level": "safe|professional_consult|not_allowed"
}
```

## 4. Module contracts

### 4.1 Quality gate

Inputs:
- `PhotoManifest`
- raw/normalized image files

Outputs:
- per-photo quality observations;
- pass/conditional/retake status;
- module allow-list.

Checks:
- `quality.blur.laplacian_variance`
- `quality.exposure.mean_luma`
- `quality.exposure.clipped_highlights_pct`
- `quality.face.visible`
- `quality.body.full_frame_visible`
- `quality.duplicate.near_duplicate_score`
- `quality.filter.synthetic_or_heavy_filter_risk`
- `quality.camera_angle.vertical_risk`
- `quality.crop.face_margin_pct`
- `quality.crop.body_missing_parts`

Retake logic:
- if no analyzable face image -> stop face modules;
- if no analyzable full-body image -> stop body modules;
- if only skin confidence is weak -> proceed but block skin observations;
- if images are too filtered -> generate retake instructions only.

### 4.2 Face geometry

Inputs:
- quality-gated face images;
- normalized crops.

Outputs:
- landmark confidence;
- capture pose;
- proportion descriptors;
- symmetry descriptors;
- feature-framing observations.

MVP metrics:
- yaw/pitch/roll;
- face bounding box and crop margins;
- eye-line tilt;
- approximate face width/height ratio;
- thirds/fifths approximations where reliable;
- left-right landmark deltas;
- smile/rest expression difference;
- hair/framing visibility flags.

Not MVP:
- surgical suggestions;
- exact attractiveness score;
- diagnosis of asymmetry disorders;
- claims based on a single distorted selfie.

### 4.3 Skin/hair/grooming cue module

Inputs:
- face images;
- quality gate outputs;
- optional user routine notes.

Outputs:
- non-medical presentation cues;
- grooming/hair/framing recommendations;
- dermatologist/professional-consult prompts only when boundary allows.

MVP metrics:
- lighting-normalized skin-readability score;
- facial hair boundary clarity if applicable;
- hairline/hairstyle framing visibility;
- contrast between hair/face/clothing/background;
- grooming consistency across photos.

Blocked claims:
- acne diagnosis;
- hair-loss diagnosis;
- dermatology prescriptions;
- “you need X procedure.”

### 4.4 Body posture and silhouette

Inputs:
- full-body photos;
- body role labels;
- quality outputs.

Outputs:
- pose keypoint observations;
- posture and stance descriptors;
- clothing/fit readability;
- body-photo retake instructions.

MVP metrics:
- shoulder line angle;
- hip line angle;
- head-over-shoulder alignment proxy;
- stance width and rotation risk;
- full-body crop completeness;
- garment fit flags where visible;
- camera-height distortion risk.

Blocked claims:
- precise body-fat percentage;
- medical posture diagnosis;
- health risk claims;
- body shaming language.

### 4.5 Style and fit

Inputs:
- full-body images;
- body module output;
- optional wardrobe goal.

Outputs:
- fit observations;
- styling opportunities;
- low-cost/high-return wardrobe recommendations.

MVP examples:
- shirt shoulder seam appears dropped/oversized;
- trouser break shortens/lengthens perceived leg line;
- neckline competes with face framing;
- outfit contrast is low/high for the photo use case.

### 4.6 Photo presence

Inputs:
- current profile/headshot images;
- face/body observations;
- user goal.

Outputs:
- dating/professional shot list;
- lighting/framing guidance;
- expression/crop recommendations;
- photo order suggestions.

MVP examples:
- “Use one eye-level, clean-background headshot first.”
- “Use a full-body image where posture and fit read clearly.”
- “Retake from farther away to reduce lens distortion.”

### 4.7 Recommendation ranking

Inputs:
- all recommendation candidates;
- intake profile;
- module confidence summaries;
- guardrail policy.

Outputs:
- ranked recommendations;
- grouped roadmap;
- blocked/deferred suggestions;
- “do not prioritize” list.

Algorithm v0:

```python
def rank(candidate, intake):
    if candidate.guardrail_level == "not_allowed":
        return None

    confidence = clamp(candidate.confidence, 0, 1)
    impact = clamp(candidate.impact_estimate, 0, 1)
    relevance = clamp(candidate.user_goal_relevance, 0, 1)
    reversibility_bonus = 0.75 + 0.25 * clamp(candidate.reversibility, 0, 1)
    compounding_bonus = 1.15 if candidate.category in {"posture", "training", "skin_routine", "photo_capture"} else 1.0
    readiness = intake.readiness_by_category.get(candidate.category, 0.7)

    burden = 0.20 + candidate.effort + candidate.cost + candidate.risk + candidate.time_to_visible_result
    score = (impact * confidence * relevance * reversibility_bonus * compounding_bonus * readiness) / burden

    if candidate.guardrail_level == "professional_consult":
        score *= 0.45

    return round(score, 4)
```

Rules:
- Retake recommendations outrank analysis recommendations when core confidence is below threshold.
- “Today” actions must be safe, cheap, reversible, and high-confidence.
- Clinical/professional consult prompts are separated from the main action list.
- Arii should recommend leaving some things alone when ROI is low.

## 5. Metric registry

Metrics should live in a registry so the report can cite what supports each claim.

Example registry record:

```json
{
  "metric_id": "quality.blur.laplacian_variance",
  "module": "quality_gate",
  "label": "Image sharpness",
  "description": "Variance of Laplacian blur proxy used to detect soft images.",
  "input_roles": ["face_front_neutral", "body_front"],
  "output_type": "number",
  "unit": "variance_proxy",
  "confidence_dependencies": ["resolution", "exposure"],
  "user_facing": false
}
```

Metric families:
- `quality.*`
- `face.capture.*`
- `face.geometry.*`
- `face.presentation.*`
- `body.pose.*`
- `body.silhouette.*`
- `style.fit.*`
- `photo.presence.*`
- `recommendation.*`

## 6. Report JSON package

The report is generated from structured data, not from a freeform prompt.

Top-level sections:

```json
{
  "report_id": "rpt_01HX...",
  "job_id": "job_01HX...",
  "schema_version": "0.1.0",
  "user_goal_summary": "Dating profile + full face/body direction",
  "overall_confidence": "moderate",
  "photo_quality_summary": {},
  "executive_summary": {},
  "face_analysis": {},
  "body_analysis": {},
  "style_and_photo_presence": {},
  "ranked_recommendations": [],
  "roadmap": {},
  "retake_requests": [],
  "limitations": [],
  "safety_disclaimer": "Arii is non-medical appearance guidance only."
}
```

## 7. 30-page report composition plan

1. Cover page.
2. Privacy and non-medical boundary.
3. Executive summary.
4. Photo-set quality scorecard.
5. What Arii could and could not infer.
6. Face capture analysis.
7. Face proportion/framing overview.
8. Hair/framing direction.
9. Skin/complexion readability and non-medical routine cues.
10. Grooming/presentation opportunities.
11. Expression and camera presence.
12. Body photo quality.
13. Posture and stance overview.
14. Frame/silhouette descriptors.
15. Clothing fit and proportion effects.
16. Training/posture emphasis, non-medical.
17. Dating/professional photo presence.
18. Highest-ROI quick wins.
19. 7-day action plan.
20. 30-day action plan.
21. 90-day action plan.
22. Wardrobe/fit checklist.
23. Hair/grooming checklist.
24. Photo retake recipe.
25. What not to prioritize yet.
26. Professional consult boundaries, if user allows.
27. Progress tracking method.
28. Metric appendix.
29. Limitations appendix.
30. Closing summary.

## 8. Prompt architecture for LLM reasoning

LLM calls should be constrained.

### Reasoning prompt input

- intake profile;
- module outputs;
- metric confidence summaries;
- allowed categories;
- blocked categories;
- tone guide.

### Reasoning prompt output

- JSON only first;
- every claim references observation IDs;
- no unsupported medical claims;
- no beauty scoring;
- include limitations.

### Narrative prompt output

- transforms approved JSON findings into report prose;
- cannot introduce new findings;
- must include confidence qualifiers;
- must maintain Arii tone: calm, private, practical.

## 9. Validation and evaluation plan

### Unit tests

- quality gate thresholds;
- module output schema validation;
- ranking formula monotonicity;
- guardrail filtering;
- report schema validity.

### Fixture tests

Fixture sets:
- high-quality full set;
- blurred face image;
- cropped body image;
- heavy filter image;
- inconsistent angle selfie set;
- good face / bad body;
- good body / bad face.

Expected behavior:
- no crash;
- correct retake requests;
- low-confidence observations blocked;
- report JSON validates;
- ranked recommendations contain no forbidden categories.

### Human review

Use expert/human QA to rate:
- usefulness;
- kindness/tone;
- overclaiming;
- prioritization quality;
- whether recommendations are actionable.

### Bias and robustness review

- Test across skin tones, body types, ages, genders/presentation styles, lighting, and camera devices.
- Avoid universal aesthetic assumptions; route to user goals and personal presentation context.
- Keep clinical/medical boundary explicit.

## 10. Engineering milestones

### Milestone A — schemas and fixtures

- Create Pydantic models.
- Export JSON schemas.
- Add sample manifests.
- Add validation tests.

### Milestone B — quality gate

- Implement OpenCV blur/exposure/crop checks.
- Generate retake instructions.
- Add fixture tests.

### Milestone C — landmark extraction

- Add MediaPipe face/pose extraction.
- Store landmark summaries, not raw unnecessary data.
- Add confidence aggregation.

### Milestone D — recommendation ranking

- Implement candidate generator.
- Implement ranking formula.
- Add guardrail filters.
- Add “do not prioritize” output.

### Milestone E — report package

- Compose report JSON.
- Render sample HTML/PDF.
- Add snapshot test for sample report.

### Milestone F — product integration

- Secure upload flow.
- Async job orchestration.
- Frontend report viewer.
- Deletion/privacy tooling.

## 11. First prototype CLI shape

```bash
arii-engine analyze \
  --manifest fixtures/sample_manifest.json \
  --out reports/sample_report.json \
  --render-html reports/sample_report.html
```

Expected output:

```text
✓ quality_gate: conditional pass, 0.82 confidence
✓ face_geometry: pass, 0.78 confidence
✓ body_pose: pass, 0.74 confidence
✓ recommendation_ranking: 18 candidates ranked
✓ report_composer: reports/sample_report.json
```

## 12. Acceptance criteria for MVP engine

- Produces valid report JSON from a fixture manifest.
- Clearly requests retakes when required.
- Never produces a beauty score.
- Never states medical diagnosis.
- Every recommendation has evidence IDs, confidence, effort/cost/risk estimates, and a priority score.
- Web/PDF report can be generated from the JSON without custom manual edits.
- Low-confidence modules fail closed rather than hallucinating.
