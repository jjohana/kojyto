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
   - Set target level, target slide count, difficulty, content profile and question count.
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
- The backend caps concurrent generation work through `KOJYTO_MAX_PARALLELISM`; the Render performance test is set to `10`.
- The backend is the source of truth for the active course through `GET /admin/generation/active`.
- Refreshing the GitHub Pages admin does not cancel a generation; the admin reloads the active course from the backend.
- If another course is already queued or running, upload, resume, slide regeneration and audio regeneration return `409` with the active course id.
- Interrupted running jobs are requeued on backend startup, then picked up again by the worker if the database and storage survived the restart.
- Reliable resume requires durable database and file storage. Ephemeral Render storage cannot guarantee recovery after a service restart.

Required deployment variables:

| Variable | Purpose |
| --- | --- |
| `OPENAI_API_KEY` | Text, image and audio generation. |
| `LEARNER_SITE_URL` | `https://jjohana.github.io/kojyto/index.html` |
| `ADMIN_SITE_URL` | `https://jjohana.github.io/kojyto/admin` |
| `BACKEND_API_URL` | Public HTTPS URL of the deployed backend, used when exporting GitHub Pages. |
| `DATABASE_URL` | Course metadata. The free demo Render config uses SQLite in `/app/storage/app.db`; use PostgreSQL or another durable database for reliable resume. |
| `STORAGE_DIR` | Uploaded documents and generated course files. The free demo Render config uses `/app/storage`; attach persistent storage or external object storage before relying on resume after a restart. |
| `KOJYTO_DEFAULT_PARALLELISM` | Default concurrent generation work. Render performance test: `10`. |
| `KOJYTO_MAX_PARALLELISM` | Hard concurrent generation cap, regardless of frontend payload. Render performance test: `10`. |

`Dockerfile` and `render.yaml` are included as deployment starters for a FastAPI-compatible HTTPS host. After the backend URL exists, set `BACKEND_API_URL` to that HTTPS URL, regenerate `docs/`, then publish GitHub Pages.
GitHub Actions can publish the backend image to `ghcr.io/jjohana/kojyto-backend:latest` when the `Backend container` workflow is run manually.

## Backend robustness probe

Run the backend robustness probe inside Codex before asking an administrator to test manually:

```powershell
.\.python311\python.exe scripts\test_backend_robustness.py --keep
```

The probe does not call OpenAI and does not generate a real course. It verifies the operational contract that matters before generation starts:

- `/health` is reachable.
- `/admin/debug/status` reports writable storage and the active-job count.
- a course upload creates exactly one queued generation job.
- a second upload is rejected while the first job is active.
- `/admin/debug/courses/{course_id}` exposes source files, working directory, artifacts, steps, jobs and events.
- a running job is requeued after a simulated backend restart if the database and storage survive.

The script writes a JSON report named `backend_robustness_report.json` in its temporary work directory. If this probe fails, do not launch a real course generation.

For a local scheduler stress test with 10 concurrent slide/QCM tasks:

```powershell
.\.python311\python.exe scripts\test_parallel_generation_probe.py --parallelism 10 --slides 20 --delay 0.2
```

This produces `parallel_generation_report.json` with elapsed time, generated slides, generated QCM and measured maximum concurrency. The deployed Render backend is controlled by its own `KOJYTO_DEFAULT_PARALLELISM` and `KOJYTO_MAX_PARALLELISM` variables.

## Debug endpoints

Use these endpoints when a course appears blocked or lost:

| Endpoint | Purpose |
| --- | --- |
| `GET /admin/debug/status` | Backend health, storage writability, worker state, active generation, latest courses and jobs. |
| `GET /admin/debug/courses/{course_id}` | Course-specific progress, steps, jobs, events, alerts, artifact list and file existence checks. |

These endpoints deliberately avoid raw secrets. They are designed to identify whether the problem is the worker, the job queue, the database, generated artifacts or missing files.

The bundled Render demo target is:

- backend URL: `https://kojyto-api-jjohana.onrender.com`
- plan: `free`
- only required Render secret: `OPENAI_API_KEY`
- automatic deploys disabled
- shutdown window: 300 seconds
- GitHub Actions keepalive ping every 10 minutes

This is suitable for demonstration, not for durable storage. A restart or redeploy can remove generated files and SQLite data. For long-running use, replace SQLite with PostgreSQL and add persistent storage.

Render deployment:

1. Keep Render auto-deploy disabled for `kojyto-api-jjohana`.
2. Deploy only when backend code must change.
3. Use Render `Manual Deploy` > `Deploy latest commit`, or run the GitHub Actions workflow `Backend container` manually with `deploy_render=true` if `RENDER_DEPLOY_HOOK_URL` is configured.

This avoids killing an active generation when the admin site or GitHub Pages snapshot changes.

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
