# Tuya Logic hosted deployment

This isolated Linux deployment is based on the stable 1.6.0 application, with hosted authentication and connection settings. It does not contain the Windows runtime, Android APK, credentials, saved programs, local backups, or test evidence. The surrounding workspace is not a deployment context.

## Runtime contract

| Setting | Required value |
| --- | --- |
| Runtime | Node.js 22 LTS or newer supported LTS; Dockerfile uses `node:22-bookworm-slim` |
| Container user | UID/GID `10001:10001`, non-root |
| Listen | `0.0.0.0:3210`, HTTP behind the provider's HTTPS proxy |
| Replicas | Exactly **one**, no autoscaling, preview deployments, canary, or overlapping replacement |
| Data | Persistent Single Read/Write volume mounted at `/data`, writable by UID/GID 10001 |
| Updates | Stop the old instance before starting its replacement |
| Health | Public `GET /api/health` returns `{"ok":true}`; configure an HTTP health check on port 3210 |
| Proxy | HTTPS, same-origin requests, and streaming `/api/events` without response buffering |

Docker's health check only tests server availability. It does not claim Tuya connectivity, fresh device values, or a running Ladder program. Platforms that ignore image health checks need their own HTTP probe.

Keep the application process running continuously. Closing the browser or APK does not stop the server's engine. A server restart, deployment, fault, or lost Tuya event connection stops execution; **the program never resumes automatically**. Sign in and deliberately choose Run or Live Test after checking device conditions. Stop holds physical device values and cannot retract an already sent command.

The container alone cannot guarantee one active engine. The hosting platform must enforce the replica and update rules. Before enabling this hosted engine, stop the local Windows engine controlling the same devices. Keep the local app in STOP while using the hosted engine.

## Runtime variables and secrets

Set these in the host's runtime configuration, never in build arguments, Git, the image, or the archive:

| Variable | Meaning |
| --- | --- |
| `TUYA_HOSTED=1` | Mandatory hosted mode; already set by the Dockerfile |
| `PUBLIC_ORIGIN` | Exact external HTTPS origin, for example `https://tuya.example.com`, without a path or trailing slash |
| `TUYA_LOGIC_DATA=/data` | Absolute persistent data directory; already set by the Dockerfile |
| `TUYA_AUTH_SALT` | Application password's 32-character hexadecimal scrypt salt; runtime secret |
| `TUYA_AUTH_HASH` | Matching 128-character hexadecimal scrypt hash; runtime secret |
| `TUYA_API_KEY` | Tuya.AI Bearer key; runtime secret. Present selects live devices; absent selects demo preview |
| `HOST=0.0.0.0`, `PORT=3210` | Container listener; already set by the Dockerfile |

Missing or malformed required hosted settings fail startup before listening. Provision the salt/hash through a trusted local process or reuse the application's existing salt/hash pair securely. The password itself is not a server environment variable. Never paste secret values into build logs or chat. Existing Windows DPAPI credential files are tied to the Windows account and are not Linux deployment credentials.

Hosted mode disables public password setup and API-key/mode changes, including requests forwarded by a loopback proxy. Those settings are administered using platform runtime secrets. The browser's login still uses the application password. This is independent of the Tuya account password. Sessions are memory-only, so restarting requires login again.

Keep `/data` through container replacement; never attach an empty ephemeral directory as a substitute for persistent storage. Back up the volume separately. Export saved programs before maintenance and retain a private copy outside the hosting account. Program import is a deliberate application action; this package does not migrate a user's programs automatically.

## Local verification and source archive

Run from this directory with Node.js 22+:

```text
node scripts/package-hosting.mjs --check
node scripts/package-hosting.mjs --archive
node scripts/package-hosting.mjs --verify artifacts/Tuya-Logic-Hosted-v1.6.0.tar.gz
```

The archive uses an explicit file allowlist, rejects symlinks, verifies every file's SHA-256 and byte length against an included manifest, and includes no tests or user-data paths. `--verify` also checks the archive against the current reviewed sources. It refuses to overwrite an earlier archive. Extract it into a new directory; the build context is the extracted `tuya-logic` directory. The companion `.sha256` file records the archive hash. Verification does not substitute for actually building and running the Linux container.

`package-lock.json` pins `ws` and its registry integrity. The supplied `generate-lock.mjs` checks the exact dependency metadata and downloaded archive hash without executing package scripts. Run it only when intentionally updating the lock. If the dependency graph grows, use `npm install --package-lock-only --ignore-scripts` and review the resulting lock instead. The image installs with `npm ci --omit=dev --omit=optional --ignore-scripts`.

Where Docker is already available, build with `docker build -t tuya-logic:1.6.0-hosted .`. Provide runtime secrets through the deployment system and attach the persistent volume before running. The Docker image uses a maintained major-version tag so rebuilds receive Node/security updates; record and deploy the resulting immutable image digest after validating it. No Docker installation or runtime build was performed on the current Windows machine because Docker is absent.

## Northflank: not eligible for this no-card deployment

Checked November 11, 2026. The current [pricing page](https://northflank.com/pricing) advertises no-card Sandbox access, always-on services, and up to $20/month of resources. However, its [billing documentation](https://northflank.com/docs/v1/application/billing/pricing-on-northflank) still says every plan requires a payment method and describes Sandbox as intended for exploration rather than production. The account's actual create-service screen required a payment card, despite showing a Free team. **Northflank therefore does not meet this user's no-card requirement. No service was deployed.** Do not add a payment card, upgrade plans, or buy resources for this deployment. The following technical notes remain reference material for an independently eligible account; they are not a claim that this deployment is free or available.

The pricing page lists persistent disk separately at $0.11657/GiB/month and egress at $0.06/GB. It does not clearly guarantee which volume allocation is included for Sandbox. Choose only a configuration whose account estimate is $0 and whose volume can be created without a card or paid-plan change. If a free persistent volume is unavailable, this deployment's persistence requirement is unmet; ephemeral container storage is not an acceptable workaround. A free managed database allowance is not a file volume, and the current application does not use that database for persistence. Do not use BYOC as a free substitute: its underlying cloud can charge separately. Published quotas and terms may change.

Three documented deployment routes are available:

1. **Git repository:** publish only the reviewed archive contents to an authorized repository, connect that repository, and create a combined service with Dockerfile build. Use `/Dockerfile` and build context `/` when these contents are the repository root. If deliberately using a dedicated monorepo subdirectory, both paths must point to this isolated directory. Disable automatic deployment until configuration and the storage gate are checked. [Build from Git](https://northflank.com/docs/v1/application/getting-started/build-and-deploy-your-code), [Dockerfile paths](https://northflank.com/docs/v1/application/build/build-with-a-dockerfile).
2. **Container registry:** build the reviewed Docker context on an available builder, publish the image to an authorized registry, and deploy its immutable digest as a continuously running service. Private registries require stored registry credentials. A local tar.gz source file is not itself a deployable registry image. [Run a registry image](https://northflank.com/docs/v1/application/run/run-an-image-from-a-container-registry).
3. **Image archive upload:** the project's Images area accepts an **uncompressed Docker or OCI image tar**, which is imported into Northflank's own registry. This can avoid a separate registry account or Git integration, but still requires a real Linux image built for the deployment architecture. Upload completion is not deployment readiness; wait for the image's Ready status, then select its specific version in a service. The source tar.gz created by this package script is not an image archive and cannot be uploaded using this route. [Import container images](https://northflank.com/docs/v1/application/build/import-container-images).

Use the minimum suitable **free** compute plan, one replica, and a Single Read/Write volume at `/data`. Northflank documents that this volume mode limits the service to one instance and terminates the previous container before starting its replacement. It also derives volume group ownership from the image's explicit `USER uid:gid`; verify `/data` is writable without switching the application to root. [Persistent volumes](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume).

Expose HTTP container port 3210 as a public port; Northflank supplies a hostname and TLS certificate. Set `PUBLIC_ORIGIN` to that exact HTTPS hostname before starting. Add the runtime secrets and HTTP health check. Keep automatic deployment disabled while first validating the service. [Public port and TLS setup](https://northflank.com/docs/v1/application/getting-started/build-and-deploy-your-code).

Before live use, verify health, unauthenticated state rejection, login, SSE updates, program save/reload, volume persistence after a controlled restart, and the required STOP state after restart. Use demo mode or isolated mocks during validation. Do not command real devices to prove hosting works.

## Windows and Android access

Open the external HTTPS origin in a Windows browser, or use the separate hosting Windows client at `hosting/windows-client/dist/Tuya-Logic-Client.exe`. Keep its `.config`, three WebView2 libraries and notice files together. In **Server address**, choose **HTTPS server**, enter the exact HTTPS origin, and select **Connect**. This connection does not start a local engine; browser profiles are separate for each origin. The **Local server** option requires the complete existing server package alongside the executable. The original stable Windows executable remains tied to localhost.

The hosting Android client is `hosting/android-client/dist/Tuya-Logic-Android-v1.1.1.apk`, signed with the same identity and package ID as the existing APK so it can update that installation. Save or export any draft before changing servers. In the app, select **Server**, enter the root HTTPS origin, and select **Connect to server**, then log in with the application password. Changing servers clears cookies and browser storage. Remote access needs internet connectivity; a local HTTP address instead requires access to the server's local network. Public plain HTTP, path-based server addresses and invalid HTTPS certificates are rejected. The original APK already supports HTTPS; version 1.1.1 corrects the native help text. Actual physical Android device testing is pending.

Client binaries are separate artifacts and are not included in this Linux source archive. Both clients can close while the hosted engine continues independently of the laptop. The hosted server must first be deployed and reachable; these client changes do not provide hosting by themselves.

The Tuya.AI API has its own availability and usage limits; free hosting does not make third-party API usage unlimited. This deployment does not add separate Tuya Cloud project credentials or promise access to phone-app scene/automation data unavailable to the current API.
