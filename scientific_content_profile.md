# Scientific Content Profile

This document describes the new `content_profile` branch added on top of the existing generation pipeline.

The goal is narrow:

- keep the current behavior for ordinary narrative or procedural lessons
- automatically detect lessons that are formula-heavy, theorem-heavy, scientifically structured, or programming-oriented
- switch only those lessons into a stricter scientific generation mode

`pipeline_profile` and `content_profile` are now distinct:

- `pipeline_profile`: `standard` or `compact`
- `content_profile_selection`: `auto`, `standard`, or `scientific`
- `content_profile`: resolved runtime profile, `standard` or `scientific`

## 1. Design Principles

- `standard` lessons must not degrade
- scientific behavior is opt-in through `scientific` or conservative `auto`
- auto-classification must be persisted, inspectable, and resumable
- the scientific branch affects prompts, slide validation, quiz generation, and visual-summary prompting

## 2. End-to-End Flow

### 2.1 Upload-time state

`pages/admin_upload.py` and `src/backend_api.py` now store:

- the operator selection: `auto|standard|scientific`
- a pending or forced `ContentProfileAnalysis`
- the run config fields:
  - `content_profile_selection`
  - `content_profile`

If the operator selects:

- `standard`: the run is immediately forced to `standard`
- `scientific`: the run is immediately forced to `scientific`
- `auto`: the run starts with a pending analysis and resolves after ingestion

### 2.2 Resolution point

The profile is resolved in `CourseOrchestrator.run_generation_job(...)` after chunk ingestion and before:

- `build_course_map(...)`
- `build_slide_plan(...)`
- `build_compact_slide_plan(...)`
- slide generation
- quiz generation

This timing matters because the classifier uses real chunk content, not just the filename.

## 3. Classification Algorithm

Implementation lives in `src/content_profiles.py`.

### 3.1 Inputs

The classifier uses:

- source filenames
- the ingested document chunks
- local heuristics extracted from the chunks
- an LLM classification pass when `content_profile_selection=auto`

### 3.2 Heuristic signals

The local analysis counts these signals:

- `formula_hits`
- `theorem_hits`
- `derivation_hits`
- `code_hits`
- `science_hits`
- `notation_hits`
- `example_hits`
- `symbolic_chunks`

The regex families look for:

- formulas and symbolic relations: `\frac`, `\sum`, `\int`, `x = y`, `=>`, `->`, etc.
- theorem vocabulary: `theoreme`, `lemme`, `proof`, `preuve`, `corollaire`, `axiome`
- derivation/calculation vocabulary: `derivee`, `equation`, `formule`, `matrice`, `vecteur`, `algorithme`
- programming syntax: fenced code, inline code, `def`, `class`, `return`, `import`, `for`, `while`, `if`, `function`, `const`, etc.
- scientific domains: maths, physics, chemistry, engineering, statistics, programming, algorithmics, signal, data structures
- notation markers: `on note`, `variable`, `notation`, `unite`, `symboles`

### 3.3 Heuristic score

The current weighted score is:

```text
score =
  formula_hits * 4
  + theorem_hits * 6
  + derivation_hits * 3
  + code_hits * 3
  + science_hits * 2
  + notation_hits * 1
  + symbolic_chunks * 3
```

This weighting is intentionally biased toward false negatives over false positives. A lesson should not become scientific because it contains one stray equation or a technical acronym.

### 3.4 LLM classifier

When `selection=auto`, the LLM receives:

- source filenames
- heuristic score and breakdown
- heuristic signals
- a ranked sample of the most symbolic chunks

The exact classifier prompt is:

```text
Tu classes un cours importe dans un seul profil:
- `standard`
- `scientific`

Choisis `scientific` seulement si le support repose de facon structurelle sur au moins un des elements suivants:
- formules, equations, relations symboliques, notations mathematiques
- theoremes, lemmes, preuves, demonstrations, derivations
- raisonnement scientifique, physique, chimique, statistique ou d'ingenierie
- algorithmique, programmation, syntaxe de code, pseudo-code, structures de donnees

Choisis `standard` si le support reste majoritairement narratif, descriptif, reglementaire, procedurier, management, economie non calculatoire, histoire, langue ou culture generale.

Regle de prudence:
- en cas de doute limite, choisis `standard`
- ne choisis pas `scientific` sur un simple vocabulaire technique isole
- la presence occasionnelle d'un chiffre ou d'un acronyme ne suffit pas

Retour attendu:
- `profile`: `standard` ou `scientific`
- `confidence`: `low`, `medium` ou `high`
- `rationale`: justification courte en francais
- `signals`: 2 a 6 indices concrets en francais
```

### 3.5 Resolution logic

The final profile is resolved as follows:

1. If the operator forced `standard` or `scientific`, that wins immediately.
2. If `auto` is active and the heuristic score is `>= 12`, resolve to `scientific`.
3. Otherwise, if the LLM says `scientific` with `medium` or `high` confidence, resolve to `scientific`.
4. Otherwise, if the LLM says `scientific` and the heuristic score is already moderately supportive (`>= 6`), resolve to `scientific`.
5. Else resolve to `standard`.
6. If the LLM classification fails, fall back to heuristics only:
   - `score >= 12` -> `scientific`
   - else `standard`

This keeps the auto mode conservative and protects non-scientific lessons.

## 4. Scientific Prompt Branch

Prompt builders live in `src/prompts.py`.

The base prompts are unchanged for the standard profile. The scientific mode appends extra rules only when `content_profile == "scientific"`.

### 4.1 Course-map delta

```text
Profil scientifique:
- repere explicitement les definitions, lois, theoremes, notations, derivations, algorithmes et exemples de calcul
- ne dilue pas les passages symboliques dans une paraphrase uniquement textuelle
- preserve les dependances entre hypotheses, formule, interpretation et application
```

### 4.2 Slide-plan delta

```text
Profil scientifique:
- reserve des slides distinctes quand une formule, un theoreme, une preuve, une derivation ou un algorithme porte a lui seul un objectif d'apprentissage
- separe si necessaire l'enonce, l'interpretation, les hypothesees, l'exemple d'application et les limites
- garde les notations et symboles centraux visibles dans les points de cours
```

### 4.3 Slide-spec delta

```text
Profil scientifique:
- si la slide porte sur une relation symbolique centrale, utilise obligatoirement un bloc `formula`
- explique les symboles, les unites, les hypotheses ou les conditions de validite dans les autres blocs
- pour un theoreme, une loi ou un principe formel, utilise volontiers un bloc `quote` pour l'enonce puis des blocs d'explication
- pour une derivation ou un raisonnement technique, montre les etapes ou l'interpretation, pas seulement le resultat final
- pour la programmation et l'algorithmique, preserve la precision des noms, de la syntaxe et des invariants dans un format sobre
- n'ecris jamais une slide scientifique purement decorative: chaque bloc doit aider a raisonner ou a calculer
```

### 4.4 Quiz delta

```text
Profil scientifique:
- privilegie des questions d'application, de calcul, d'interpretation de formule, de lecture de notation, de conditions de validite ou de trace d'algorithme
- quand c'est utile, reinsere la formule ou la notation exacte en LaTeX
- evite les questions purement recitatives si une verification par raisonnement est possible
```

## 5. Scientific Slide Guidance

Per-slide guidance is computed from the selected chunks in `build_slide_profile_guidance(...)`.

For each slide context, the system derives booleans such as:

- `formula_priority`
- `theorem_priority`
- `derivation_priority`
- `notation_priority`
- `worked_example_priority`
- `code_priority`

These are sent into the slide prompt payload as `profile_guidance`.

Example of the guidance intent:

- `formula_priority=true`: the slide should contain a real `formula` block
- `theorem_priority=true`: the slide should expose a formal statement clearly
- `notation_priority=true`: symbols and units should be defined, not implied
- `code_priority=true`: structured bullets or tables should preserve syntax and invariants

## 6. Scientific Validation

The scientific branch adds one extra post-generation control in `validate_slide_content_profile(...)`.

Current rule:

- if the scientific guidance says the slide has a central symbolic relation, the final slide must contain at least one valid `formula` block

This validation is intentionally narrow. It enforces the most valuable behavior change without making the scientific branch fragile.

## 7. Visual-Summary Adaptation

The visual-summary pipeline in `src/slide_visuals.py` also receives `content_profile`.

Scientific-specific visual rules:

- keep formulas, notation, theorem statements, and algorithm structures legible
- prefer scientific diagrams over generic business infographics
- preserve symbols and variables when they matter pedagogically

The scientific visual delta is:

```text
Profil scientifique:
- si la slide contient une formule, une notation, un theoreme, une loi ou un algorithme, le visuel doit les rendre memorisables sans les degrader
- privilegie les formats tels que relation entre variables, schema explicatif, derivation guidee, carte de notation, comparaison de cas, trace de calcul ou structure algorithmique
- n'ajoute pas d'icones generiques si elles masquent le raisonnement
- preserve les symboles, noms de variables et intitulés techniques utiles avec une formulation courte et exacte
```

The visual extractor now also includes `formula` LaTeX content in its text digest so scientific labels are not silently dropped.

## 8. Files Changed

Core implementation:

- `src/models.py`
- `src/config.py`
- `src/content_profiles.py`
- `src/prompts.py`
- `src/planner.py`
- `src/quiz_generator.py`
- `src/slide_visuals.py`
- `src/orchestrator.py`
- `src/persistence.py`
- `src/backend_api.py`
- `src/backend_client.py`
- `src/worker.py`
- `pages/admin_upload.py`
- `pages/admin_publish.py`
- `scripts/test_html_slides_generation.py`

Tests:

- `tests/test_content_profiles.py`
- `tests/test_planner.py`
- `tests/test_backend_api.py`
- `tests/test_resume_flow.py`

## 9. Current Limitations

- Programming lessons are classified as `scientific`, but there is still no dedicated `code` block type in the slide schema.
- The scientific validator currently enforces formula presence only for clearly symbolic slides, not full theorem/proof structure.
- Auto-classification remains conservative by design; borderline courses may still stay `standard`.

## 10. Practical Outcome

The workflow is now adapted in a way that should not deteriorate non-scientific lessons:

- ordinary courses keep the previous prompt behavior
- scientific or programming-heavy courses can be forced manually
- auto mode can detect many symbolic courses without globally tightening the pipeline
- scientific lessons receive stronger prompts, stronger slide checks, more appropriate quizzes, and more relevant visual summaries
