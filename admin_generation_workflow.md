# KOJYTO Admin Generation Workflow

This document is the operational checklist for the separate KOJYTO admin site.

## Site Roles

| Site | URL / entry point | Role |
| --- | --- | --- |
| Learner public site | `https://jjohana.github.io/kojyto/index.html` | Static public catalog and published learner courses. |
| Admin public site | `https://jjohana.github.io/kojyto/admin/` | Admin interface for course creation and publication. Technical endpoints are configured before export, not entered in the page. |
| Generation service | `/admin/` on the backend | Authenticated operational service for upload, generation, preview, publication and export. |
| Worker | `scripts/run_worker.py` | Executes queued course generation, regeneration and narration operations. |

The GitHub Pages admin must not ask the administrator to enter technical URLs. Public learner/admin/generation URLs are configured in `kojyto.site.json`, environment variables, or deployment settings before export.

## Admin Follow-Up Experience

After the administrator launches a course, the admin opens the course follow-up screen automatically. The page shows a global progress bar and a 16-step timeline with user-facing statuses only:

- `En attente`
- `En cours`
- `Termine`
- `Attention requise`
- `Erreur bloquante`

The timeline covers:

1. Validation des documents.
2. Lecture des documents.
3. Analyse pedagogique.
4. Recherche web et sources.
5. Plan du cours.
6. Redaction des slides.
7. Images et visuels.
8. Controle qualite des images.
9. QCM intermediaires.
10. QCM final.
11. Verification des QCM.
12. Scenario de narration.
13. Fichiers audio.
14. Assemblage du site.
15. Verification finale.
16. Publication.

The administrator sees only useful controls: resume after failure, publish when the course is ready, regenerate slide, verify QCM, regenerate audio, regenerate the course, and open the learner version when available. Raw logs and job identifiers are not displayed as the main experience.

## Complete Course Generation Steps

1. Document upload
   - Upload 1 to 5 `PDF` / `DOCX` files.
   - Store original sources and source manifest.
   - Reject unsupported file types and oversized files before queuing generation.

2. Pedagogical configuration
   - Set target level, target slide count, difficulty, content profile, pipeline profile, parallelism, QCM mode and question count.
   - Set web enrichment policy: enabled/disabled, guide text and preferred domains.
   - Use the default IAAG logo unless a course-specific logo is supplied.

3. Logo and brand handling
   - Default logo: `assets/iaag-logo.png`.
   - Optional custom logo: uploaded file or logo URL.
   - File upload takes priority over URL when both are supplied.
   - The selected logo is copied into the generated course site and displayed on the left side of slide and QCM visual pages.
   - The image-generation prompt forbids generated school names, IAAG, logos, institutional marks or watermarks inside the image itself.

4. Ingestion
   - Extract PDF/DOCX text.
   - Render or inspect page images where useful for comprehension.
   - Build namespaced document chunks for multi-source courses.
   - Persist ingestion alerts.

5. Content planning
   - Resolve or force the content profile.
   - Build the course map.
   - Build the slide plan.

6. Research and sources
   - Run slide-scoped web research when enabled.
   - Respect preferred domains where configured.
   - Keep web references on generated slides for admin and learner review.

7. Slide and visual generation
   - Generate structured slide content.
   - Generate the opening visual with the existing image engine.
   - Run validation and repair when required.
   - Persist slide specs and rendered HTML.

8. QCM generation
   - Generate intermediate QCMs by slide or by subpart.
   - Run automatic quality cleanup.
   - Build the final validation QCM from the cleaned training bank.
   - Persist QCM reports and blocking alerts when the final bank is too small.

9. Narration and audio
   - Generate the detailed narration scenario.
   - Create scripts for course intro, slides, intermediate QCM intros and final evaluation intro.
   - Generate audio files.
   - Run audio QA when enabled.
   - Rebuild the static site after audio generation.

10. Assembly, preview and publication
    - Build the dual-mode static site: `master`, `user`, `deck`.
    - Preview admin/master and learner versions from the admin.
    - Publish only when the generated site and final QCM are valid.
    - Export the published learner snapshot into `docs/` for GitHub Pages.

## Local Operation

Start the generation service:

```powershell
.\.python311\python.exe scripts\run_backend_api.py
```

Start the worker in another terminal:

```powershell
.\.python311\python.exe scripts\run_worker.py
```

Open the admin served by the generation service:

```powershell
start http://127.0.0.1:8000/admin/
```

If the service runs on another port, open the same `/admin/` path on that service.

## GitHub Pages Export

The public learner snapshot is generated with:

```powershell
.\.python311\python.exe scripts\export_github_pages.py --output docs
```

The exported admin at `docs/admin/index.html` intentionally has no local URL and no visible URL input. If `PUBLIC_ADMIN_API_URL` is set before export, generation actions call that configured service automatically. If it is empty, the page remains a clean published admin entry without exposing technical wiring.

Public URLs are centralized in `kojyto.site.json`:

```json
{
  "LEARNER_SITE_URL": "https://jjohana.github.io/kojyto/index.html",
  "ADMIN_SITE_URL": "https://jjohana.github.io/kojyto/admin",
  "PUBLIC_ADMIN_API_URL": ""
}
```

## Admin Acceptance Checklist

- The admin can upload valid source documents.
- The admin can configure pedagogy, web enrichment, QCM, pipeline profile and logo.
- The selected logo is visible in generated slide and QCM pages.
- Generated images do not contain school names or logos.
- The admin can follow the 16 user-facing generation stages with clear statuses, progress, alerts and recent activity.
- The admin can inspect sources, slides, QCMs, narration and audio status.
- The admin can resume, rename, regenerate a slide, regenerate narration, clean QCMs, regenerate a full course, publish and export.
- The learner site remains static and GitHub Pages-compatible.
