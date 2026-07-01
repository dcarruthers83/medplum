# Deploying the Medplum sandbox to Render

This deploys a **complete, self-contained Medplum instance** (API + web UI + Postgres +
Redis) on Render from a single [`render.yaml`](./render.yaml) Blueprint, so your
colleagues can reach it at public URLs. It uses Medplum's **official prebuilt Docker
images**, so nothing is compiled — deploys take minutes, not the full monorepo build.

## What gets created

| Render resource | Type | What it is |
|---|---|---|
| `medplum-server` | Web service (Docker image) | The API server, `medplum/medplum-server:5.1.22` |
| `medplum-app` | Web service (Docker image) | The web UI, `medplum/medplum-app:5.1.22` |
| `medplum-db` | Postgres | Managed database |
| `medplum-redis` | Key Value | Managed Redis (cache + BullMQ job queues) |

The server and Redis talk over Render's **private network**; only the two web
services are exposed publicly.

## Rough cost

With the defaults in `render.yaml` (server `standard`, db `basic-256mb`, redis + app `free`):
roughly **~$25–32/month** (the `standard` server instance is the bulk of it). To cut cost
for a low-traffic sandbox, drop the server to `starter` and the db to `free` — see
[Tuning cost](#tuning-cost-and-persistence) below. Confirm current prices at
<https://render.com/pricing>.

---

## Step 1 — Push this Blueprint to your fork

`render.yaml` and this guide need to be on a branch Render can read. Your `origin`
already points at your personal fork (`dcarruthers83/medplum`):

```bash
cd medplum
git checkout -b render-sandbox
git add render.yaml DEPLOY-RENDER.md
git commit -m "Add Render all-in-one deployment blueprint"
git push -u origin render-sandbox
```

## Step 2 — Create the Blueprint in Render

1. Sign in at <https://dashboard.render.com> (create a free account if needed).
2. **New → Blueprint**.
3. Connect your GitHub and pick the **`dcarruthers83/medplum`** repo, branch `render-sandbox`.
4. Render parses `render.yaml` and lists the 4 resources. It will prompt for the four
   `sync: false` URL variables — you don't know the real URLs yet, so just enter
   **placeholders** for now (e.g. `https://placeholder/` ). You'll fix them in Step 4.
5. Click **Apply**. Postgres and Redis provision first; the two web services pull their
   images and start.

## Step 3 — Note the assigned URLs

Once the services are live, open each web service in the dashboard and copy its URL:

- `medplum-server` → e.g. `https://medplum-server-ab12.onrender.com`
- `medplum-app` → e.g. `https://medplum-app-cd34.onrender.com`

## Step 4 — Set the real URLs (the important bit)

Render assigns the `*.onrender.com` hostnames only after creation, so the URL env vars
must be filled in now. **Trailing slashes matter** — include them exactly as shown.

On **`medplum-server`** → *Environment*:

| Variable | Value |
|---|---|
| `MEDPLUM_BASE_URL` | `https://<your-server>.onrender.com/` |
| `MEDPLUM_APP_BASE_URL` | `https://<your-app>.onrender.com/` |
| `MEDPLUM_STORAGE_BASE_URL` | `https://<your-server>.onrender.com/storage/` |

On **`medplum-app`** → *Environment*:

| Variable | Value |
|---|---|
| `MEDPLUM_BASE_URL` | `https://<your-server>.onrender.com/` |

Save each service's changes — Render redeploys it automatically. (Both must redeploy:
the server bakes the URLs into OAuth/links; the app rewrites its JS bundle to point at
the API on container start.)

## Step 5 — Verify and log in

1. Health check: open `https://<your-server>.onrender.com/healthcheck` →
   `{"ok":true,"postgres":true,"redis":true}`.
2. Open the app URL. Sign in with the **auto-seeded super-admin**:
   - **Email:** `admin@example.com`
   - **Password:** `medplum_admin`
   - **⚠️ Change this password immediately** — it's a well-known default and your
     instance is public.
3. Add colleagues: in the app, go to your **Project → Users → Invite** (admin invite is
   the reliable path; self-registration may be blocked by reCAPTCHA domain checks).

---

## Tuning cost and persistence

Edit the `plan:` lines in `render.yaml` (or change plans in the dashboard):

- **Cheaper:** set `medplum-server` to `plan: starter` and remove its `disk:` block;
  set `medplum-db` to `plan: free`. Caveat: `starter`/`free` give 512MB RAM and the
  server **may run out of memory on boot** — watch the deploy logs. Free Postgres is
  **deleted after 30 days**, and free web services **spin down when idle** (first
  request after idle is slow).
- **Binary/file persistence:** the `disk:` block on `medplum-server` (paid instances
  only) keeps uploaded file attachments across deploys. Without it, uploaded `Binary`
  content is lost on each redeploy (FHIR records in Postgres are unaffected).
- **Redis durability:** `free` Key Value has no persistence; job queue state is lost on
  restart. That's fine for a sandbox (Postgres is the source of truth).

## Locking it down (recommended before sharing widely)

The defaults favor "just works" over security. For a shared sandbox holding anything
beyond test data:

- Change the seeded admin password (Step 5).
- Set `MEDPLUM_ALLOWED_ORIGINS` to your app URL instead of `*`.
- Set `MEDPLUM_REGISTER_ENABLED` to `"false"` on `medplum-app` and invite users instead.
- Remember this is **synthetic/test data only** — it is not a HIPAA-eligible
  configuration.

## Deploying your own fork code (instead of the official images)

The Blueprint runs the official `medplum/medplum-{server,app}:5.1.22` images, so it does
**not** include any code changes you make in this fork. To ship your own build you'd
publish custom images and point `image.url` at them. The repo's build path is
`scripts/build-docker-server.sh` (it produces the tarballs the root `Dockerfile`
expects) plus the app image in `packages/app/`. This is a larger lift (CI to build and
push multi-arch images to a registry Render can pull) — ask and I'll set it up when you
actually need to deploy modified code.

## Troubleshooting

- **Server crashes on boot with `ENOENT: ... open 'medplum.config.json'`:** the server
  wasn't told to read config from env vars. Ensure `medplum-server` has **`dockerCommand: env`**
  in `render.yaml` (or set the **Docker Command** field to `env` on the service in the dashboard),
  then redeploy. Without it the image defaults to a config *file* that doesn't exist.
- **Server deploy "live" but app can't reach API / login fails:** the app's
  `MEDPLUM_BASE_URL` is wrong or missing a trailing slash (Step 4). Fix and redeploy the app.
- **Server crashes on boot with OOM:** bump `medplum-server` to `standard` (2GB).
- **DB connection / SSL errors:** if you see `server does not support SSL connections`,
  delete `MEDPLUM_DATABASE_SSL_REJECT_UNAUTHORIZED` from the server env and redeploy.
- **Health check shows `redis:false`:** confirm `medplum-redis` and `medplum-server` are
  in the **same region** (`oregon`) so the private connection resolves.
