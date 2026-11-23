# Developer guide

This page takes you from nothing to a live app. It names what you do and what to expect from the owner; the host is not your concern. The container contract is in [`AGENTS.md`](../AGENTS.md).

## The steps

| # | Step | You | The owner |
|---|---|---|---|
| 1 | [Request a repo](#1-request-a-repo) | Open the form | Runs one command |
| 2 | [Run it locally](#2-run-it-locally) | `podman run` | |
| 3 | [What the contract asks](#3-what-the-contract-asks) | Keep it true | |
| 4 | [Set secrets](#4-set-secrets) | `./set-secret NAME` | |
| 5 | [Release](#5-release) | Push a tag | Approves it |
| 6 | [Use staging](#6-use-staging-optional) | Push to `staging` | Assigns a slot |
| 7 | [Roll back](#7-roll-back) | Re-run with an old version | Approves it |
| 8 | [Get logs](#8-get-logs) | Ask | Reads them |

## 1. Request a repo

1. Open **Issues, New issue, New app** on [ArachneDev/app-starter](https://github.com/ArachneDev/app-starter/issues/new/choose). Fill every field. The issue is public: name secrets, never paste a value.
2. Wait. The owner runs the new-app command in a terminal, so there is no progress to watch.
3. Accept two GitHub invitations: `ArachneDev/<app>`, a copy of this template with the pipeline set up, and the private `ArachneDev/<stack>-secrets`. You get Write on both.

Check: both repos open for you. Tell the owner on the issue if either does not.

## 2. Run it locally

```sh
podman build --no-cache --build-arg VERSION=dev -t app .
echo '{"greeting": "hi"}' > /tmp/app.json   # fake values only
podman run --rm -p 3000:3000 --env-file .env.example \
  --read-only --tmpfs /tmp --cap-drop all --memory 512m \
  -v /tmp/app.json:/run/secrets/app.json:ro,Z app
```

Check: `curl localhost:3000/healthz` prints `ok dev`.

## 3. What the contract asks

| It asks | Why |
|---|---|
| Listen on `PORT`; answer `/healthz` with the version | The host routes to the port and checks that the new version runs |
| Read secrets only from `/run/secrets/app.json` | Values reach the app without passing through the owner, the environment or the image |
| Build URLs from `PUBLIC_URL` | The app sits behind a proxy |
| Run non-root on a read-only root, writing only to `/tmp` | One app cannot change the host or another app |
| List volumes and dependencies in the README | The host mounts and backs up what you list |

Open a pull request to check a change. The pipeline builds the image, runs it under these limits, scans it and names any broken line with its fix.

## 4. Set secrets

1. Clone `ArachneDev/<stack>-secrets` and read its `README.md`. Install `age` as it says.
2. Run `./set-secret NAME` and type the value at the hidden prompt. Add `--env staging` for a staging value. Then `git push`.
3. In the app, read `NAME` from `/run/secrets/app.json` and add a fake value to `ci/app.json`.

Check: the value is in `/run/secrets/app.json` within about 5 minutes. You can't read a value back; set it again to change it. Never print, commit or send a value.

## 5. Release

1. Merge to `main`, then tag that commit: `git tag v1.0.0 && git push origin v1.0.0`.
   Check: the `release` run builds, scans and pushes `ghcr.io/arachnedev/<app>:v1.0.0`.
2. The owner approves the `production` job. It waits until they do.
3. About 4 minutes after the approval, the app runs the new version. The job then polls `/healthz` for up to 900 s. If it fails, the job moves `:prod` back to the previous version and goes red. Fix the code and release a new tag.

Check: `curl https://<your hostname>/healthz` prints `ok v1.0.0`. The first release needs the owner to set up `promote`, `production-url` and the `production` reviewer: [app-pipeline README](https://github.com/ArachneDev/app-pipeline#release-channels).

## 6. Use staging (optional)

Ask the owner for a slot in an issue. They assign it for a limited time. `git push origin main:staging` then publishes `:staging` with no approval, and the slot runs it with the values you set with `--env staging`.

Check: `https://staging<n>.doublew.nu/healthz` prints the new version.

## 7. Roll back

1. Open **Actions, release, Run workflow** and enter an earlier `version`, such as `v0.9.0`.
2. The owner approves it as in step 5.

Check: `/healthz` prints that version.

## 8. Get logs

You have no host access. Open an issue on your app's repo with the time and the symptom, and ask the owner for the logs. They send the relevant lines.

---

The owner's half of each step is in the private `infra` repo, `docs/runbook/`: `new-app.md`, `app-secrets.md`, `staging-slots.md` and `rollback.md`.
