# Card Audit v2 + Reviews Analysis Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Внедрить контракт аудита карточек v2 (промпт + схема + payload) и пересобрать отчёты аудита и анализа отзывов в Manager Portal (+ marketing mock отзывов) под эталонные A4-макеты с печатью; без chrome extension и без серверного PDF.

**Architecture:** Backend `card_audit` переходит на JSON-контракт v2 при сохранении модульной expert knowledge через `knowledge_loader`. Portal рендерит print-first A4-листы из view-model с адаптером v1→v2. Выбор `generationPrompts[].frameId` передаётся в card-generate. `reviews_analysis` backend не меняется — только UI.

**Tech Stack:** Python/Pydantic, React/Next (manager-portal), TanStack Router (market-navigators), существующий OpenRouter LLM client.

**Spec:** `docs/superpowers/specs/2026-10-29-card-audit-and-reviews-redesign-design.md`

## Global Constraints

- Не трогать `extension-chrome/`.
- Не добавлять серверную генерацию PDF.
- Не включать archive/trace страницы эталона аудита.
- UI: только понятные пользовательские формулировки (без имён моделей/схем).
- Минимальный diff вне задачи; без новых библиотек без необходимости.
- Перед правками кода — согласование с пользователем, если работаете вне уже одобренного плана по шагам этого файла (пользователь уже одобрил spec+подход).

## File map

| File | Responsibility |
|---|---|
| `backend/.../card_audit/domain/report_schema.py` | Pydantic v2 report models |
| `backend/.../card_audit/infrastructure/llm/prompts.py` | System frame + response schema + user instructions |
| `backend/.../card_audit/infrastructure/llm/pipeline.py` | Result payload (`generationPrompts`) |
| `backend/.../card_audit/infrastructure/llm/parser.py` | Unchanged API; validates new schema |
| `backend/.../card_generate/application/service.py` | Extract prompt by frameId / fallback |
| `backend/tests/unit/card_audit/test_report_schema_v2.py` | Schema + parse + payload tests |
| `manager-portal/src/lib/api.ts` | TS types for v2 |
| `manager-portal/src/lib/card-audit-report-view-model.ts` | v1/v2 adapter + image URL resolution |
| `manager-portal/src/components/card-audit-report/**` | A4 report UI (split from monolith) |
| `manager-portal/src/components/reviews-analysis-report.tsx` | A4 reviews UI |
| `manager-portal/.../card-generations/new/page.tsx` | `frameId` query support |
| `market-navigators/src/routes/reviews-analysis.tsx` | Marketing mock restyle |
| `docs/architecture/card-audit.md` | Doc sync |

Tracks **A–C** (audit) and **D–E** (reviews) can run in parallel after Task 1 types exist on backend; reviews UI does not need Task 1.

---

### Task 1: Expand `CardAuditReport` Pydantic schema to v2

**Files:**
- Modify: `backend/src/markethacker/modules/card_audit/domain/report_schema.py`
- Create: `backend/tests/unit/card_audit/test_report_schema_v2.py`

**Interfaces:**
- Consumes: existing `CamelModel`, `CardAuditReport` consumers
- Produces: v2 models; v1 payloads still validate where fields overlap; unknown-only v2 keys no longer dropped

- [ ] **Step 1: Write failing tests for v2 models**

```python
# backend/tests/unit/card_audit/test_report_schema_v2.py
from markethacker.modules.card_audit.domain.report_schema import CardAuditReport
from markethacker.modules.card_audit.infrastructure.llm.parser import parse_card_audit_report


def test_v2_minimal_report_validates() -> None:
    report = CardAuditReport.model_validate(
        {
            "reportVersion": "2.0",
            "platform": "wildberries",
            "mode": "quick",
            "advertisingDecision": False,
            "score": 63,
            "confidenceScore": 46,
            "headline": "Диагноз",
            "summary": "Кратко",
            "scoreBreakdown": [
                {
                    "component": "Обложка",
                    "score": 72,
                    "weight": 0.15,
                    "confidence": "medium",
                    "evidence": ["S01"],
                    "deductions": ["Декор конкурирует"],
                },
                {
                    "component": "Цена, доставка",
                    "score": None,
                    "weight": 0.1,
                    "confidence": "low",
                    "evidence": [],
                    "deductions": [],
                },
            ],
            "prioritizedActions": [
                {
                    "actionId": "A01",
                    "priority": "P0",
                    "action": "Добавить мерки",
                    "impact": "high",
                    "effort": "low",
                    "confidence": "high",
                }
            ],
            "generationPrompts": [
                {
                    "frameId": "F01",
                    "goal": "Чистая обложка",
                    "status": "ready",
                    "prohibited": ["текст"],
                    "missingReferences": [],
                }
            ],
            "returnRisk": {"overall": "unknown", "confidence": "low"},
            "funnelDiagnosis": {"funnelStage": "unknown"},
            "slides": [
                {
                    "n": "1",
                    "title": "Обложка",
                    "verdict": "Товар читается",
                    "state": "fix",
                    "thumbnailRef": "1",
                    "role": "cover",
                    "evidence": "Красное платье",
                    "annotations": [{"label": "Декор", "x": 0.7, "y": 0.2}],
                }
            ],
        }
    )
    assert report.report_version == "2.0"
    assert report.confidence_score == 46
    assert report.score_breakdown[0].score == 72
    assert report.score_breakdown[1].score is None
    assert report.generation_prompts[0].frame_id == "F01"
    assert report.slides[0].thumbnail_ref == "1"


def test_v1_legacy_report_still_validates() -> None:
    report = parse_card_audit_report(
        """
        {
          "score": 50,
          "headline": "Старый",
          "summary": "v1",
          "generationPrompt": "один промпт",
          "actionsP0": ["Сделать A"],
          "actionsP1": ["Сделать B"]
        }
        """
    )
    assert report.score == 50
    assert report.generation_prompt == "один промпт"
    assert report.actions_p0 == ["Сделать A"]
    assert report.report_version in ("", "1.0") or report.report_version is None
```

- [ ] **Step 2: Run tests — expect fail (fields missing)**

Run: `cd backend && python -m pytest tests/unit/card_audit/test_report_schema_v2.py -v`  
Expected: FAIL on unknown/missing attributes.

- [ ] **Step 3: Implement schema extensions**

In `report_schema.py` add (keep existing v1 fields optional for compat):

```python
ConfidenceLevel = Literal["high", "medium", "low"]
ImpactLevel = Literal["high", "medium", "low"]
FramePromptStatus = Literal["ready", "draft", "blocked"]
ReturnRiskOverall = Literal["low", "medium", "high", "unknown"]
FunnelStage = Literal[
    "impression_to_click",
    "click_to_cart",
    "cart_to_order",
    "order_to_buyout",
    "buyout_to_return",
    "unknown",
]


class ScoreBreakdownItem(CamelModel):
    component: str
    score: int | None = Field(default=None, ge=0, le=100)
    weight: float = Field(default=0, ge=0, le=1)
    confidence: ConfidenceLevel = "low"
    evidence: list[str] = Field(default_factory=list)
    deductions: list[str] = Field(default_factory=list)


class AuditSignal(CamelModel):
    label: str
    count: int = 0  # legacy v1
    mentions: int | None = None
    sample_size: int | None = None
    share: float | None = None
    positive: bool = False
    variants: list[str] = Field(default_factory=list)
    examples: list[str] = Field(default_factory=list)
    confidence: ConfidenceLevel | None = None


class SlideAnnotation(CamelModel):
    label: str
    x: float | None = None  # 0..1 relative, optional
    y: float | None = None
    position: str = ""  # e.g. "top-right" if no coords


class AuditSlide(CamelModel):
    n: str
    title: str
    verdict: str
    state: SlideState
    notes: list[str] = Field(default_factory=list)
    thumbnail_ref: str = ""
    role: str = ""
    evidence: str = ""
    annotations: list[SlideAnnotation] = Field(default_factory=list)
    action_id: str = ""
    acceptance_criteria: list[str] = Field(default_factory=list)


class PrioritizedAction(CamelModel):
    action_id: str
    priority: RiskLevel
    action: str
    impact: ImpactLevel = "medium"
    effort: ImpactLevel = "medium"
    confidence: ConfidenceLevel = "medium"
    dependencies: list[str] = Field(default_factory=list)
    required_assets: list[str] = Field(default_factory=list)
    owner_role: str = ""
    acceptance_criteria: list[str] = Field(default_factory=list)
    metric: str = ""
    do_not_start_before: list[str] = Field(default_factory=list)


class ExpectationGap(CamelModel):
    promise: str = ""
    evidence_for: list[str] = Field(default_factory=list)
    evidence_against: list[str] = Field(default_factory=list)
    risk: str = ""
    action_id: str = ""
    source_ids: list[str] = Field(default_factory=list)


class ReturnRisk(CamelModel):
    overall: ReturnRiskOverall = "unknown"
    confidence: ConfidenceLevel = "low"
    drivers: list[str] = Field(default_factory=list)
    missing_data: list[str] = Field(default_factory=list)
    checks: list[str] = Field(default_factory=list)


class FunnelDiagnosis(CamelModel):
    funnel_stage: FunnelStage = "unknown"
    evidence: list[str] = Field(default_factory=list)
    missing_data: list[str] = Field(default_factory=list)
    recommended_checks: list[str] = Field(default_factory=list)


class ProductLockItem(CamelModel):
    lock_id: str
    property: str = ""
    value: str = ""
    source_ids: list[str] = Field(default_factory=list)
    confidence: ConfidenceLevel = "medium"
    immutable: bool = True


class GenerationFramePrompt(CamelModel):
    frame_id: str
    goal: str = ""
    source_images: list[str] = Field(default_factory=list)
    product_lock_refs: list[str] = Field(default_factory=list)
    scene: str = ""
    composition: str = ""
    camera: str = ""
    lighting: str = ""
    model_and_styling: str = ""
    creative_zone: str = ""
    free_text_zone: str = ""
    prohibited: list[str] = Field(default_factory=list)
    acceptance_criteria: list[str] = Field(default_factory=list)
    text_overlay: str = ""
    status: FramePromptStatus = "draft"
    missing_references: list[str] = Field(default_factory=list)


class ProductLockQa(CamelModel):
    required_checks: list[str] = Field(default_factory=list)
    reject_on_immutable_mismatch: bool = True
```

On `CardAuditReport` add fields with defaults so v1 still parses:

```python
report_version: str = ""
platform: str = "wildberries"
mode: str = "quick"
advertising_decision: bool = False
confidence_score: int | None = Field(default=None, ge=0, le=100)
score_breakdown: list[ScoreBreakdownItem] = Field(default_factory=list)
prioritized_actions: list[PrioritizedAction] = Field(default_factory=list)
expectation_gaps: list[ExpectationGap] = Field(default_factory=list)
return_risk: ReturnRisk = Field(default_factory=ReturnRisk)
funnel_diagnosis: FunnelDiagnosis = Field(default_factory=FunnelDiagnosis)
product_lock: list[ProductLockItem] = Field(default_factory=list)
generation_prompts: list[GenerationFramePrompt] = Field(default_factory=list)
product_lock_qa: ProductLockQa = Field(default_factory=ProductLockQa)
report_layout: list[str] = Field(default_factory=list)
# keep generation_prompt, actions_p0, actions_p1, roadmap for v1
```

Fix `test_v1_legacy_report_still_validates` assertion to `assert report.report_version == ""`.

- [ ] **Step 4: Run tests — expect pass**

Run: `cd backend && python -m pytest tests/unit/card_audit/test_report_schema_v2.py -v`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/src/markethacker/modules/card_audit/domain/report_schema.py \
  backend/tests/unit/card_audit/test_report_schema_v2.py
git commit -m "$(cat <<'EOF'
feat(card-audit): extend report schema for audit contract v2

EOF
)"
```

---

### Task 2: Update LLM prompts to v2 contract

**Files:**
- Modify: `backend/src/markethacker/modules/card_audit/infrastructure/llm/prompts.py`
- Modify: `backend/tests/unit/card_audit/test_report_schema_v2.py` (optional prompt smoke) or add `test_prompts_v2.py`

**Interfaces:**
- Consumes: `build_system_prompt` / `build_user_prompt` signatures unchanged
- Produces: system prompt referencing v2 JSON; user instructions for `generationPrompts`, `scoreBreakdown`, visual slides

- [ ] **Step 1: Write failing assertion on system prompt contents**

```python
from markethacker.modules.card_audit.infrastructure.llm.prompts import build_system_prompt

def test_system_prompt_is_v2() -> None:
    text = build_system_prompt(expert_knowledge="## stub\n")
    assert "Версия инструкции: 2.0" in text
    assert "generationPrompts" in text
    assert "confidenceScore" in text
    assert "scoreBreakdown" in text
    assert "productLock" in text
    assert '"reportVersion": "2.0"' in text
    # expert knowledge still injected, not replaced by full dump
    assert "## stub" in text
```

- [ ] **Step 2: Run — expect fail**

Run: `cd backend && python -m pytest tests/unit/card_audit/test_report_schema_v2.py::test_system_prompt_is_v2 -v`

- [ ] **Step 3: Replace `_SYSTEM_FRAME` and `_RESPONSE_SCHEMA`**

1. Set `_SYSTEM_FRAME` to lines 1–18 of `new-audit-and-new-reviews-analyze/wb_card_audit_prompt_v2.txt` (frame only; **do not** paste expert-core.md into the file).
2. After the frame, still append `expert_knowledge` in `build_system_prompt`.
3. Insert contract narrative block from the same file section `# Контракт отчёта и генерации v2` (lines ~496–619) as `_CONTRACT_V2` string constant **or** a short condensed version that keeps all MUST rules (transparent score, evidence map, visual fields on slides, expectation gaps, return risk, funnel, prioritized actions, product lock, generation prompts, A/B null rules, reportLayout order).
4. Set `_RESPONSE_SCHEMA` to the JSON example from lines 622–767, but **extend `slides` items** to:

```json
"slides": [{
  "n": "1",
  "title": "...",
  "verdict": "...",
  "state": "keep"|"fix"|"move"|"cut",
  "notes": ["..."],
  "thumbnailRef": "1",
  "role": "...",
  "evidence": "...",
  "annotations": [{"label": "...", "x": null, "y": null, "position": ""}],
  "actionId": "",
  "acceptanceCriteria": []
}]
```

5. In `build_user_prompt` `instructions`, replace `generationPrompt` bullet with:

```python
"generationPrompts: отдельный кадр = отдельный объект; status ready|draft|blocked; "
"только подтверждённые свойства; blocked + missingReferences если нет референса.",
"Заполни scoreBreakdown (5–8), confidenceScore, prioritizedActions, "
"expectationGaps, returnRisk, funnelDiagnosis, productLock, productLockQa, reportLayout.",
"Для каждого slides укажи thumbnailRef (порядковый номер кадра 1..N как строка), "
"role, evidence; annotations — только если уверен в зоне.",
"Не заполняй actionsP0/actionsP1/generationPrompt/roadmap — используй prioritizedActions "
"и generationPrompts (legacy поля оставь пустыми).",
```

Remove obsolete instruction that only asks for singular `generationPrompt`.

`build_system_prompt`:

```python
def build_system_prompt(*, expert_knowledge: str) -> str:
    return (
        f"{_SYSTEM_FRAME}\n"
        f"{expert_knowledge.strip()}\n\n"
        f"{_CONTRACT_V2.strip()}\n\n"
        f"Формат ответа (строго JSON):\n{_RESPONSE_SCHEMA}"
    )
```

- [ ] **Step 4: Run prompt + schema tests — expect pass**

- [ ] **Step 5: Commit**

```bash
git add backend/src/markethacker/modules/card_audit/infrastructure/llm/prompts.py \
  backend/tests/unit/card_audit/test_report_schema_v2.py
git commit -m "$(cat <<'EOF'
feat(card-audit): switch LLM prompts to audit contract v2

EOF
)"
```

---

### Task 3: Pipeline payload — `generationPrompts` + legacy bridge

**Files:**
- Modify: `backend/src/markethacker/modules/card_audit/infrastructure/llm/pipeline.py`
- Modify: `backend/tests/unit/card_audit/test_report_schema_v2.py`

**Interfaces:**
- Consumes: `CardAuditReport` v2
- Produces: result dict with `generationPrompts`, `generationPrompt` (legacy first ready/draft text for old clients)

- [ ] **Step 1: Failing test for payload**

```python
from markethacker.modules.card_audit.domain.report_schema import CardAuditReport
from markethacker.modules.card_audit.infrastructure.llm.pipeline import report_to_result_payload


def test_report_to_result_payload_exposes_generation_prompts() -> None:
    report = CardAuditReport.model_validate(
        {
            "score": 10,
            "headline": "h",
            "summary": "s",
            "reportVersion": "2.0",
            "generationPrompts": [
                {"frameId": "F01", "goal": "g", "status": "blocked", "scene": "x"},
                {
                    "frameId": "F02",
                    "goal": "cover",
                    "status": "ready",
                    "scene": "studio",
                    "composition": "full",
                    "textOverlay": "Платье",
                },
            ],
        }
    )
    payload = report_to_result_payload(
        report=report, bundle={"own": {}}, coverage={}, llm_meta={}
    )
    assert isinstance(payload["generationPrompts"], list)
    assert payload["generationPrompts"][1]["frameId"] == "F02"
    # legacy bridge: first ready flattened text must be non-empty
    assert "F02" in payload["generationPrompt"] or "cover" in payload["generationPrompt"]
    assert "generationPrompt" not in payload["report"] or payload["report"].get("generationPrompt") in ("", None)
```

Helper to flatten a frame into a designer-facing Russian text (deterministic):

```python
def flatten_frame_prompt(frame: dict[str, Any]) -> str:
    parts = [
        f"ЦЕЛЬ\n{frame.get('goal', '')}",
        f"ИСХОДНЫЕ ИЗОБРАЖЕНИЯ\n{', '.join(frame.get('sourceImages') or [])}",
        f"ЗАФИКСИРОВАННЫЕ ХАРАКТЕРИСТИКИ\n{', '.join(frame.get('productLockRefs') or [])}",
        f"СЦЕНА\n{frame.get('scene', '')}",
        f"КОМПОЗИЦИЯ\n{frame.get('composition', '')}",
        f"КАМЕРА\n{frame.get('camera', '')}",
        f"СВЕТ\n{frame.get('lighting', '')}",
        f"МОДЕЛЬ И СТИЛЬ\n{frame.get('modelAndStyling', '')}",
        f"ОГРАНИЧЕНИЯ\n{'; '.join(frame.get('prohibited') or [])}",
        f"ТЕКСТОВЫЙ СЛОЙ\n{frame.get('textOverlay', '')}",
    ]
    return "\n\n".join(p for p in parts if not p.endswith("\n"))
```

- [ ] **Step 2: Implement `report_to_result_payload`**

```python
def report_to_result_payload(...):
    report_dict = serialize_model(report)
    generation_prompts = report_dict.get("generationPrompts") or []
    if not isinstance(generation_prompts, list):
        generation_prompts = []
    legacy = str(report_dict.pop("generationPrompt", "") or "").strip()
    if not legacy:
        for status in ("ready", "draft"):
            for frame in generation_prompts:
                if isinstance(frame, dict) and frame.get("status") == status:
                    legacy = flatten_frame_prompt(frame)
                    break
            if legacy:
                break
    return {
        "status": "completed",
        "headline": report.headline,
        "bundle": bundle,
        "report": report_dict,
        "generationPrompt": legacy,
        "generationPrompts": generation_prompts,
        "coverage": coverage,
        "llm": llm_meta,
    }
```

- [ ] **Step 3: Tests pass**

- [ ] **Step 4: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(card-audit): expose generationPrompts in audit result payload

EOF
)"
```

---

### Task 4: card_generate — resolve prompt by `frameId`

**Files:**
- Modify: `backend/src/markethacker/modules/card_generate/application/service.py` (`_extract_generation_prompt`)
- Modify: `backend/src/markethacker/modules/card_generate/api/schemas.py` (optional `source_frame_id: str | None`)
- Modify: `backend/src/markethacker/modules/card_generate/api/router.py` + `application/service.py` create path to pass frame id
- Create/modify unit test under `backend/tests/unit/card_generate/` if present; else add focused test next to service helpers

**Interfaces:**
- Consumes: audit `result` with `generationPrompts`
- Produces: resolved prompt string for selected frame

- [ ] **Step 1: Failing tests for extraction**

```python
from markethacker.modules.card_generate.application.service import (
    _extract_generation_prompt,
)

def test_extract_by_frame_id() -> None:
    result = {
        "generationPrompts": [
            {"frameId": "F01", "status": "blocked", "goal": "back"},
            {"frameId": "F02", "status": "ready", "goal": "cover", "scene": "studio"},
        ]
    }
    text = _extract_generation_prompt(result, frame_id="F02")
    assert text is not None
    assert "cover" in text

def test_extract_skips_blocked_without_frame_id() -> None:
    result = {
        "generationPrompts": [
            {"frameId": "F01", "status": "blocked", "goal": "x"},
            {"frameId": "F02", "status": "ready", "goal": "y", "scene": "s"},
        ]
    }
    text = _extract_generation_prompt(result, frame_id=None)
    assert text is not None and "y" in text
```

- [ ] **Step 2: Implement**

```python
def _extract_generation_prompt(
    result: dict[str, Any], *, frame_id: str | None = None
) -> str | None:
    prompts = result.get("generationPrompts")
    report = result.get("report")
    if not isinstance(prompts, list) and isinstance(report, dict):
        prompts = report.get("generationPrompts")
    if isinstance(prompts, list) and prompts:
        if frame_id:
            for frame in prompts:
                if isinstance(frame, dict) and frame.get("frameId") == frame_id:
                    if frame.get("status") == "blocked":
                        return None
                    return flatten_frame_prompt(frame)  # share helper or duplicate small
        for status in ("ready", "draft"):
            for frame in prompts:
                if isinstance(frame, dict) and frame.get("status") == status:
                    return flatten_frame_prompt(frame)
    # legacy singular
    ...
```

Wire `source_frame_id` through create API body and service `create_*` method; portal will send it.

Share `flatten_frame_prompt` — prefer one module e.g. `card_audit/infrastructure/llm/frame_prompt_text.py` imported by pipeline + card_generate (avoid duplication).

- [ ] **Step 3: Tests pass + commit**

```bash
git commit -m "$(cat <<'EOF'
feat(card-generate): resolve audit frame prompt by frameId

EOF
)"
```

---

### Task 5: Manager Portal — TS types + view-model adapter

**Files:**
- Modify: `manager-portal/src/lib/api.ts` (`CardAuditReport`, `CardAuditResultBody`, related)
- Create: `manager-portal/src/lib/card-audit-report-view-model.ts`
- Create: `manager-portal/src/lib/card-audit-report-view-model.test.ts` if the repo has vitest/jest for portal; otherwise skip unit and verify manually in Task 6

**Interfaces:**
- Consumes: `CardAuditResultBody`
- Produces: `CardAuditReportViewModel` with normalized actions, prompts, slide images

- [ ] **Step 1: Extend TS interfaces** mirroring Pydantic camelCase (all new fields optional for v1).

- [ ] **Step 2: Implement adapter**

```ts
export interface ResolvedSlide extends CardAuditSlide {
  imageUrl: string | null;
}

export interface CardAuditReportViewModel {
  isV2: boolean;
  score: number;
  confidenceScore: number | null;
  headline: string;
  summary: string;
  scoreBreakdown: CardAuditScoreBreakdownItem[];
  risks: CardAuditRisk[];
  actions: CardAuditPrioritizedAction[]; // from prioritizedActions or actionsP0/P1
  slides: ResolvedSlide[];
  generationPrompts: CardAuditGenerationFramePrompt[];
  imageUrls: string[];
  // …pass-through optional blocks
}

export function buildCardAuditReportViewModel(
  result: CardAuditResultBody,
): CardAuditReportViewModel | null {
  const report = result.report;
  if (!report) return null;
  const imageUrls = extractOwnImageUrls(result.bundle);
  const isV2 = report.reportVersion === "2.0" || (report.scoreBreakdown?.length ?? 0) > 0;
  const actions =
    report.prioritizedActions?.length
      ? report.prioritizedActions
      : [
          ...(report.actionsP0 ?? []).map((action, i) => ({
            actionId: `P0-${i + 1}`,
            priority: "P0" as const,
            action,
            impact: "high" as const,
            effort: "medium" as const,
            confidence: "medium" as const,
          })),
          ...(report.actionsP1 ?? []).map((action, i) => ({
            actionId: `P1-${i + 1}`,
            priority: "P1" as const,
            action,
            impact: "medium" as const,
            effort: "medium" as const,
            confidence: "medium" as const,
          })),
        ];
  const generationPrompts =
    result.generationPrompts?.length
      ? result.generationPrompts
      : report.generationPrompts?.length
        ? report.generationPrompts
        : result.generationPrompt || report.generationPrompt
          ? [
              {
                frameId: "LEGACY",
                goal: "Промпт из аудита",
                status: "ready" as const,
                scene: result.generationPrompt || report.generationPrompt || "",
              },
            ]
          : [];
  const slides = (report.slides ?? []).map((slide, index) => ({
    ...slide,
    imageUrl: resolveSlideImageUrl(slide, imageUrls, index),
  }));
  return { isV2, score: report.score, confidenceScore: report.confidenceScore ?? null, /* … */ actions, slides, generationPrompts, imageUrls };
}

function extractOwnImageUrls(bundle: unknown): string[] {
  // navigate bundle.own.imageUrls | image_urls; filter strings
}

function resolveSlideImageUrl(slide: CardAuditSlide, urls: string[], index: number): string | null {
  const ref = slide.thumbnailRef?.trim();
  if (ref && /^https?:\/\//i.test(ref)) return ref;
  if (ref && /^\d+$/.test(ref)) {
    const n = Number(ref);
    return urls[n - 1] ?? urls[index] ?? null;
  }
  return urls[index] ?? null;
}
```

- [ ] **Step 3: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(manager-portal): add card audit v2 types and view-model adapter

EOF
)"
```

---

### Task 6: Manager Portal — card audit A4 report UI

**Files:**
- Replace/split: `manager-portal/src/components/card-audit-report.tsx` → prefer folder:
  - `card-audit-report/index.tsx` (orchestrator)
  - `card-audit-report/report-chrome.tsx` (page shell, brand, footer, print button)
  - `card-audit-report/pages/*.tsx` (executive, visual, return-risk, funnel, actions, production, appendix)
  - `card-audit-report/audit-report.css` (tokens + `@media print` from `audit_report_final.html`, без archive)
- Modify: `manager-portal/src/app/(manager)/card-audits/[id]/page.tsx` — import + print CTA if needed

**Interfaces:**
- Consumes: `CardAuditReportViewModel`, `auditId`
- Produces: printable multi-page report; links `/card-generations/new?auditId=&frameId=`

- [ ] **Step 1: Port CSS variables and `.page` / print rules** from `new-audit-and-new-reviews-analyze/audit_report_final.html` (`:root` through `@media print`), drop `.archive-page` / `.audit-trace`.

- [ ] **Step 2: Implement page sections binding to view-model** (structure mirrors HTML product pages 1–6 + appendix pages without archive). Use `<img src={slide.imageUrl}>` for visual evidence; fallback «Нет изображения».

- [ ] **Step 3: Generation prompts UI** — list frames; for `ready`/`draft` show link:

```tsx
href={`/card-generations/new?auditId=${encodeURIComponent(auditId)}&frameId=${encodeURIComponent(frame.frameId)}`}
```

`blocked` — show missingReferences, no link.

- [ ] **Step 4: Manual check** — open completed audit in portal (v1 old + fixture v2 if available); `window.print()` preview.

- [ ] **Step 5: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(manager-portal): redesign card audit report as printable A4 layout

EOF
)"
```

---

### Task 7: Portal card-generations — honor `frameId` query

**Files:**
- Modify: `manager-portal/src/app/(manager)/card-generations/new/page.tsx`
- Modify: create-generation API call body to include `sourceFrameId` if backend added it

- [ ] **Step 1:** Read `frameId` from `searchParams`.
- [ ] **Step 2:** When loading audit result, pick matching prompt (flatten client-side or use server create with `sourceFrameId`).
- [ ] **Step 3:** Fallback chain: selected frame → first ready → first draft → legacy `generationPrompt`.
- [ ] **Step 4:** Commit

```bash
git commit -m "$(cat <<'EOF'
feat(manager-portal): seed card generation from selected audit frame

EOF
)"
```

---

### Task 8: Reviews analysis report UI redesign

**Files:**
- Modify: `manager-portal/src/components/reviews-analysis-report.tsx`
- Optionally extract `reviews-analysis-report.css`
- Modify: `manager-portal/src/app/(manager)/reviews-analysis/[id]/page.tsx` only if print chrome needed

**Interfaces:**
- Consumes: existing `ReviewsAnalysisLlm` + `ReviewsAnalysisStats` (no schema change)
- Produces: 5 A4 pages matching `reviews_analysis_redesign.html`

- [ ] **Step 1: Port CSS** from `reviews_analysis_redesign.html` (lime/violet tokens, `.page`, print).

- [ ] **Step 2: Rebuild layout**
  1. Overview: always show `llm.summary` as lead; pills; rating; sentiment; gender; recommendations; structure inside right card
  2. Strengths / weaknesses topic cards with % bars
  3. Questions with example quotes
  4. Infographic cards + recommendationSections (title + recommendation + stats)
  5. finalSummary four tinted cards

- [ ] **Step 3: Manual print check** on a completed analysis.

- [ ] **Step 4: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(manager-portal): redesign reviews analysis report as printable A4 layout

EOF
)"
```

---

### Task 9: market-navigators reviews mock

**Files:**
- Modify: `market-navigators/src/routes/reviews-analysis.tsx`

- [ ] **Step 1:** Align section structure/CSS with portal redesign (same 5 pages; keep mock data).
- [ ] **Step 2:** Visual check via `pnpm`/`npm` dev for market-navigators.
- [ ] **Step 3: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(market-navigators): restyle reviews analysis marketing mock

EOF
)"
```

---

### Task 10: Docs + verification sweep

**Files:**
- Modify: `docs/architecture/card-audit.md`
- Modify: `docs/architecture/card-generate.md` if handoff documented
- Optionally short note in `docs/architecture/reviews-analysis.md` (UI-only)

- [ ] **Step 1: Update architecture docs** — reportVersion 2.0 fields, generationPrompts, print UI, no PDF server.
- [ ] **Step 2: Run backend unit tests**

```bash
cd backend && python -m pytest tests/unit/card_audit tests/unit/card_generate -v
```

Expected: PASS (or only pre-existing failures unrelated — do not ignore new failures).

- [ ] **Step 3: Confirm `extension-chrome` untouched**

```bash
git status -- extension-chrome
# clean / no changes
```

- [ ] **Step 4: Commit docs**

```bash
git commit -m "$(cat <<'EOF'
docs: document card audit v2 and printable report redesign

EOF
)"
```

---

## Self-review vs spec

| Spec requirement | Task |
|---|---|
| Prompt v2 + knowledge_loader | Task 2 |
| Schema v2 | Task 1 |
| Payload generationPrompts + legacy | Task 3 |
| UI + print A4 audit | Task 6 |
| Real gallery images | Task 5–6 |
| v1 adapter | Task 5 |
| User selects frame | Task 6–7 + 4 |
| No archive/trace | Task 6 |
| Reviews UI only | Task 8 |
| market-navigators reviews | Task 9 |
| No chrome / no server PDF | Global + Task 10 |
| card-generate handoff | Task 4, 7 |

No TBD placeholders in tasks. Types for `generationPrompts` / `frameId` consistent across Tasks 1, 3, 4, 5, 7.

---

## Execution handoff

Plan saved to `docs/superpowers/plans/2026-10-29-card-audit-and-reviews-redesign.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks  
2. **Inline Execution** — execute tasks in this session with executing-plans checkpoints  

Which approach?
