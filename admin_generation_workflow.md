# KOJYTO Admin Generation Workflow

This document is the operational checklist for the separate KOJYTO admin site.

## Site Roles

| Site | URL / entry point | Role |
| --- | --- | --- |
| Learner public site | `https://jjohana.github.io/kojyto/index.html` | Static public catalog and published learner courses. |
| Admin public site | `https://jjohana.github.io/kojyto/admin/` | Admin interface for course creation and publication. Technical endpoints are configured before export, not entered in the page. |
| Generation service | HTTPS backend configured in `BACKEND_API_URL` | Operational service for upload, generation, preview, publication and export. |
| Worker | `scripts/run_public_service.py` or `scripts/run_worker.py` | Executes queued course generation, regeneration and narration operations. |

The GitHub Pages admin must not ask the administrator to enter technical URLs. Public learner/admin/generation URLs are configured in `kojyto.site.json`, environment variables, or deployment settings before export. The public admin never redirects to a local backend.

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
4. Plan du cours.
5. Recherche web et sources.
6. Redaction des slides.
7. Images et visuels.
8. Verification des images.
9. QCM par slide.
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
   - Select the school for the course.
   - IAAG is available by default.
   - To use another logo, create or select another school and provide its logo during course launch.
   - Set target level, target slide count, difficulty, content profile, pipeline profile, parallelism and question count.
   - Web enrichment is always enabled; guide text and preferred domains can steer the research.
   - Use the selected school's logo for the course pages.

3. Logo and brand handling
   - Default school: IAAG with `assets/iaag-logo.png`.
   - Other schools are persisted as `School` records with name, logo source and logo file.
   - A custom school logo can be uploaded or supplied by URL.
   - File upload takes priority over URL when both are supplied.
   - Existing schools reuse their saved logo when a new course is launched.
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
   - Run slide-scoped web research.
   - Respect preferred domains where configured.
   - Keep web references on generated slides for admin and learner review.

7. Slide and visual generation
   - Generate structured slide content.
   - Generate the opening visual with the existing image engine.
   - Run validation and repair when required.
   - Persist slide specs and rendered HTML.

8. QCM generation
   - Generate one training QCM for each slide.
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

## Public Backend Deployment

The GitHub Pages admin is static. Course creation requires a deployed HTTPS backend because PDF/DOCX ingestion, OpenAI calls, image/audio generation and publication cannot run inside GitHub Pages.

The simplest deployable process is:

```powershell
python scripts/run_public_service.py
```

This command starts FastAPI and the queue worker in the same service so generated files, jobs and progress share the same storage.

Generation robustness:

- KOJYTO runs one active generation at a time.
- The backend is the source of truth for the active course through `GET /admin/generation/active`.
- Refreshing the GitHub Pages admin does not cancel a generation; the admin reloads the active course from the backend.
- If another course is already queued or running, upload, resume, slide regeneration and audio regeneration return `409` with the active course id.
- Interrupted running jobs are requeued on backend startup, then picked up again by the worker.

Required deployment variables:

| Variable | Purpose |
| --- | --- |
| `OPENAI_API_KEY` | Text, image and audio generation. |
| `LEARNER_SITE_URL` | `https://jjohana.github.io/kojyto/index.html` |
| `ADMIN_SITE_URL` | `https://jjohana.github.io/kojyto/admin` |
| `BACKEND_API_URL` | Public HTTPS URL of the deployed backend, used when exporting GitHub Pages. |
| `DATABASE_URL` | Course metadata. The free demo Render config uses SQLite in `/app/storage/app.db`; use PostgreSQL for durable production usage. |
| `STORAGE_DIR` | Uploaded documents and generated course files. The free demo Render config uses `/app/storage` without a paid persistent disk. |

`Dockerfile` and `render.yaml` are included as deployment starters for a FastAPI-compatible HTTPS host. After the backend URL exists, set `BACKEND_API_URL` to that HTTPS URL, regenerate `docs/`, then publish GitHub Pages.
GitHub Actions also publishes the backend image to `ghcr.io/jjohana/kojyto-backend:latest` after backend changes.

The bundled Render demo target is:

- backend URL: `https://kojyto-api-jjohana.onrender.com`
- plan: `free`
- only required Render secret: `OPENAI_API_KEY`

This is suitable for demonstration, not for durable storage. A restart or redeploy can remove generated files and SQLite data. For long-running use, replace SQLite with PostgreSQL and add persistent storage.

Render redeploy automation:

1. In Render, open `kojyto-api-jjohana`.
2. Open `Settings`, then copy the service deploy hook URL.
3. In GitHub, add a repository secret named `RENDER_DEPLOY_HOOK_URL`.
4. Run the GitHub Actions workflow `Backend container` manually once, or push a backend change.

When this secret exists, the workflow builds the backend image and then asks Render to deploy the latest commit. Without this secret, Render may keep serving an older backend until `Manual Deploy` > `Deploy latest commit` is clicked in Render.

Use the repository helper to avoid editing the wrong field:

```powershell
.\.python311\python.exe scripts\republish_site.py `
  --learner-url https://jjohana.github.io/kojyto/index.html `
  --admin-url https://jjohana.github.io/kojyto/admin `
  --backend-url https://your-kojyto-backend.example.com
```

The helper rejects non-HTTPS URLs.
Add `--pages-repo C:\path\to\pages-repo` when the generated `docs/` snapshot must be copied into a separate GitHub Pages checkout.

## GitHub Pages Export

The public learner snapshot is generated with:

```powershell
.\.python311\python.exe scripts\export_github_pages.py --output docs
```

The exported admin at `docs/admin/index.html` intentionally has no visible URL input. Generation actions call only the configured `BACKEND_API_URL`. If it is empty, the exported page remains a static course management shell and does not attempt any local fallback.
The normal export command rejects an empty `BACKEND_API_URL`; this prevents publishing an admin that cannot generate courses. Use `--allow-unconnected-admin` only for local artifact review, not for the delivered GitHub Pages site.

Public URLs are centralized in `kojyto.site.json`:

```json
{
  "LEARNER_SITE_URL": "https://jjohana.github.io/kojyto/index.html",
  "ADMIN_SITE_URL": "https://jjohana.github.io/kojyto/admin",
  "BACKEND_API_URL": "https://your-kojyto-backend.example.com"
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
