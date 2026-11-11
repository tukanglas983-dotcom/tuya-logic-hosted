# Tuya Logic hosted deployment

Hosted 1.7.1 adds automatic parallel wire joins, dependable wire deletion, endpoint dragging, and rectangle selection for deleting groups of Ladder elements, retaining the existing hosted authentication, Supabase persistence, and single-engine lease. The source archive contains no credentials, device inventory, saved programs, native clients, or private test evidence. Previously published archives remain immutable.

Ladder v1 programs remain supported. The editor migrates their series and parallel paths to explicit v2 nodes and wires. Save validates connectivity and compiles the graph on the server. Function Block Diagram uses the same execution engine and device capability metadata. The update also coalesces live state events and preserves the ordering guards introduced in 1.6.2.

No Supabase schema migration or Windows/Android client rebuild is required. Export any unsaved draft before reloading. After server replacement, sign in again if needed; execution remains stopped.

## Runtime contract

| Setting | Required value |
| --- | --- |
| Runtime | Node.js 22 LTS or newer supported LTS; Dockerfile uses `node:22-bookworm-slim` |
| Container user | Docker image uses UID/GID `10001:10001`, non-root; a native Node host manages its own runtime user |
| Listen | `0.0.0.0` on the provider's `PORT`, default 3210, behind its HTTPS proxy |
| Replicas | Configure **one**; no autoscaling or preview engine. Supabase leasing protects against replacement overlap |
| Data | Choose persistent file storage **or** the Supabase REST option below |
| Updates | Prefer stop-before-start. If the platform overlaps replacements, use Supabase leasing |
| Health | Public `GET /api/health` returns `{"ok":true}`; probe the actual container port |
| Proxy | HTTPS, same-origin requests, and streaming `/api/events` without response buffering |

Docker's health check only tests server availability. It does not claim Tuya connectivity, fresh device values, or a running Ladder program. Platforms that ignore image health checks need their own HTTP probe.

Keep the application process running continuously. Closing the browser or APK does not stop the server's engine. A server restart, deployment, fault, or lost Tuya event connection stops execution; **the program never resumes automatically**. Sign in and deliberately choose Run or Live Test after checking device conditions. Stop holds physical device values and cannot retract an already sent command.

File storage alone cannot coordinate overlapping engines. Supabase mode uses an atomic database lease, with non-owners kept in standby. Continue configuring one replica and avoiding unnecessary overlap. The lease only coordinates instances using the same Supabase project and storage namespace; it cannot coordinate the old local Windows server or another namespace. Before enabling this hosted engine, stop the local Windows engine controlling the same devices. Keep the local app in STOP while using the hosted engine.

## Runtime variables and secrets

Set these in the host's runtime configuration, never in build arguments, Git, the image, or the archive:

| Variable | Meaning |
| --- | --- |
| `TUYA_HOSTED=1` | Mandatory hosted mode; already set by the Dockerfile |
| `PUBLIC_ORIGIN` | Exact external HTTPS origin, for example `https://tuya.example.com`, without a path or trailing slash |
| `TUYA_LOGIC_DATA=/data` | Required absolute persistent directory for file mode; set by the Dockerfile. Supabase mode requires no local volume |
| `TUYA_AUTH_SALT` | Application password's 32-character hexadecimal scrypt salt; runtime secret |
| `TUYA_AUTH_HASH` | Matching 128-character hexadecimal scrypt hash; runtime secret |
| `TUYA_API_KEY` | Tuya.AI Bearer key; runtime secret. Present selects live devices; absent selects demo preview |
| `HOST=0.0.0.0`, `PORT` | Listen on all interfaces and honor the provider's assigned port. The Dockerfile supplies default port 3210 |
| `TUYA_SUPABASE_URL` | Supabase project HTTPS origin, for example `https://project-ref.supabase.co`; set together with its service key to select remote storage |
| `TUYA_SUPABASE_SERVICE_KEY` | Preferred `sb_secret_...` key or legacy JWT with role `service_role`, stored only as a runtime secret; public/publishable keys are rejected |
| `TUYA_STORAGE_NAMESPACE` | Stable storage/lease identifier; defaults to `default`. Replacements for the same engine must use the same namespace |

Missing or malformed required hosted settings fail startup before listening. Provision the salt/hash through a trusted local process or reuse the application's existing salt/hash pair securely. The password itself is not a server environment variable. Never paste secret values into build logs or chat. Existing Windows DPAPI credential files are tied to the Windows account and are not Linux deployment credentials.

Hosted mode disables public password setup and API-key/mode changes, including requests forwarded by a loopback proxy. Those settings are administered using platform runtime secrets. The browser's login still uses the application password. This is independent of the Tuya account password. Sessions are memory-only, so restarting requires login again.

## Persistence options

**File storage:** keep a Single Read/Write volume mounted at `/data` through container replacement, writable by the application user. Configure exactly one replica with stop-before-start updates. Never use an ephemeral directory as a substitute for durable storage. Back up the volume separately.

**Supabase REST storage:** use this when the Node host has no persistent disk. Install the supplied SQL and configure both Supabase variables. Programs and up to 250 recent log entries are stored remotely; log updates are batched approximately every 30 seconds, so an abrupt termination can lose the latest unsaved log entries. No local cache, volume or `TUYA_LOGIC_DATA` is required. Authentication material and Tuya/Supabase keys remain runtime secrets and are not stored in these database records. Sessions and runtime timer/counter state are not persisted.

For either option, export programs before maintenance and retain a private copy outside the hosting account. Moving from local file storage to Supabase does not automatically migrate a user's Ladder programs. Import an intentionally selected program through the authenticated editor, save it, and verify the saved revision after reload while the engine remains stopped.

### Configure Supabase

1. Create or select a project on the Free plan. Keep the account and project on a free configuration; no payment card or paid upgrade is part of this deployment.
2. Review `sql/supabase.sql`, then execute it in that project's SQL editor using the project administrator. The script creates the application's storage/lease schema and atomic load, lease, write and release functions. Run the full script, including its grants and revokes. Do not add permissive browser-access policies.
3. Keep the Supabase Data API enabled for the schema used by the script. Add `TUYA_SUPABASE_URL` and `TUYA_SUPABASE_SERVICE_KEY` to the Node host's runtime environment, with the key marked secret. This integration uses HTTPS REST/RPC; it does not need a PostgreSQL connection string, database password, direct IPv6 connection, or browser-side Supabase SDK. [Data REST API](https://supabase.com/docs/guides/api).
4. Keep a single stable `TUYA_STORAGE_NAMESPACE` for the engine and all of its replacement instances. An unrelated namespace is independent storage with an independent lease; do not use it to run a second engine on the same real devices.
5. Start in demo preview by omitting `TUYA_API_KEY`. Verify login, Save/reload, persistence after a controlled restart, and standby behavior before configuring real devices. A restart or ownership takeover must leave execution stopped.

The SQL enables Row Level Security, revokes direct table access from public, anonymous, authenticated and service roles, and grants only the application's security-definer RPC functions to `service_role`. Anonymous and normal authenticated roles cannot invoke them. The adapter sends a new secret key only in the `apikey` header; legacy service-role JWTs use that header and Bearer authorization. Elevated keys are still sensitive administrative credentials: never put one in GitHub, the APK, Windows client, public JavaScript, or exported programs. [Supabase API keys](https://supabase.com/docs/guides/getting-started/api-keys).

### Ownership, interruption and recovery

The database grants one owner per namespace. The lease lasts 45 seconds and is renewed approximately every 10 seconds. The application applies a shorter local deadline, reserving 20 seconds to drain commands before the database lease can be reassigned. A replacement that cannot claim ownership remains in standby: draft editing and export remain available, while Save, Run and physical commands are blocked. After it claims ownership, it reloads durable state before enabling mutations; it does not resume a program automatically.

Storage failure or loss of ownership stops the engine and cancels queued commands. It cannot retract a command already sent to Tuya. An unavailable or paused database does not fall back to a fresh, empty file database. Restore connectivity/project availability, allow ownership to recover, check the saved program and reported device states, then explicitly choose Run or Live Test again. Do not bypass a lease by changing namespaces or deleting its records while another instance might still be active.

The lease traffic supports ownership correctness; it is not an artificial keep-alive workaround. At a 10-second interval it produces about 8,640 small RPC requests per day per instance, plus program/log operations. Keep one instance and monitor actual database size and egress. Do not assume that request counts alone prove the database will never pause.

Supabase's Free plan currently lists 500 MB database capacity, 5 GB egress and a maximum of two active projects; automatic database backups are not included. Free projects with low database activity can pause after a week. These limits and service availability remain external constraints, so this setup does not promise uninterrupted 24/7 operation. Keep private program exports and monitor the provider's usage notices. [Free-plan limits](https://supabase.com/pricing), [project pausing](https://supabase.com/docs/guides/platform/free-project-pausing).

## Abasthan source-archive deployment

The observed Free configuration provides 512 MB RAM and 0.1 vCPU, with Disk unavailable. Use Supabase persistence for this configuration. An application being created on the Free plan does not itself establish durable storage or successful engine operation.

Use **Public Git Repository** with the authorized repository URL; no broad Git-account integration is needed. Choose branch `main`, leave **Root Directory** blank, select **Node.js 22**, and keep one Free instance. Abasthan supports a public repository URL and explicit build/start commands. [Deployment configuration](https://abasthan.com/docs/deployment-flow).

The repository must contain `Tuya-Logic-Hosted-v1.7.1.tar.gz` and the matching `package.json`, `package-lock.json` and `DEPLOYMENT.md`. The application sources, SQL and Dockerfile are inside the reviewed archive. Preserve the old 1.6.0 and 1.6.1 artifacts when publishing the new version; point this build at the exact new filename.

Set **Build Command** to:

```sh
tar -xzf Tuya-Logic-Hosted-v1.7.1.tar.gz --strip-components=1 && npm ci --omit=dev --omit=optional --ignore-scripts
```

Set **Start Command** to:

```sh
node src/server.mjs
```

Use `HOST=0.0.0.0` and allow the runtime's `PORT` to select the listening port; if the platform requires an explicit port setting, align it with the application and health probe. Do not bind to localhost. [Abasthan Node.js port configuration](https://abasthan.com/docs/deploy-nodejs).

Add the runtime variables and private secrets described above, including the final assigned HTTPS `PUBLIC_ORIGIN`, auth salt/hash and both Supabase variables. Do not place them in the public repository or build command. Leave `TUYA_API_KEY` absent for the initial demo validation. Keep automatic deployment off while validating updates. Missing hosted origin/auth settings intentionally prevent listening; a platform error such as `NO_LIVE_PORT` alone does not identify a port bug. Check the server's bounded startup error and fix the missing configuration.

After a successful build, verify `/api/health`, authentication, streaming state, program Save/reload and a controlled restart without commanding real devices. A successful build, healthy HTTP process or standby instance is not evidence that the engine owns its lease or that Tuya is connected. Check the application's storage/ownership status and STOP state separately. Native Node deployment does not test or use the supplied Dockerfile.

## Local verification and source archive

Run from this directory with Node.js 22+:

```text
node scripts/package-hosting.mjs --check
node scripts/package-hosting.mjs --archive
node scripts/package-hosting.mjs --verify artifacts/Tuya-Logic-Hosted-v1.7.1.tar.gz
```

The archive uses an explicit file allowlist, rejects symlinks, verifies every file's SHA-256 and byte length against an included manifest, and includes no tests or user-data paths. `--verify` also checks the archive against the current reviewed sources. It refuses to overwrite an earlier archive. Extract it into a new directory; the build context is the extracted `tuya-logic` directory. The companion `.sha256` file records the archive hash. Verification does not substitute for actually building and running the Linux container.

`package-lock.json` pins `ws` and its registry integrity. The supplied `generate-lock.mjs` checks the exact dependency metadata and downloaded archive hash without executing package scripts. Run it only when intentionally updating the lock. If the dependency graph grows, use `npm install --package-lock-only --ignore-scripts` and review the resulting lock instead. The image installs with `npm ci --omit=dev --omit=optional --ignore-scripts`.

Where Docker is already available, build with `docker build -t tuya-logic:1.6.2-hosted .`. Provide runtime secrets through the deployment system and configure either the persistent volume or Supabase before running. The Docker image uses a maintained major-version tag so rebuilds receive Node/security updates; record and deploy the resulting immutable image digest after validating it. No Docker installation or runtime build was performed on the current Windows machine because Docker is absent.

## Northflank: not eligible for this no-card deployment

Checked November 11, 2026. The current [pricing page](https://northflank.com/pricing) advertises no-card Sandbox access, always-on services, and up to $20/month of resources. However, its [billing documentation](https://northflank.com/docs/v1/application/billing/pricing-on-northflank) still says every plan requires a payment method and describes Sandbox as intended for exploration rather than production. The account's actual create-service screen required a payment card, despite showing a Free team. **Northflank therefore does not meet this user's no-card requirement. No service was deployed.** Do not add a payment card, upgrade plans, or buy resources for this deployment. The following technical notes remain reference material for an independently eligible account; they are not a claim that this deployment is free or available.

The pricing page lists persistent disk separately at $0.11657/GiB/month and egress at $0.06/GB. It does not clearly guarantee which volume allocation is included for Sandbox. Choose only a configuration whose account estimate is $0. Ephemeral container storage is not durable storage; use either a qualifying persistent volume or the explicit Supabase option. A generic managed database allowance is not automatically compatible with the Supabase REST/RPC integration. Do not use BYOC as a free substitute: its underlying cloud can charge separately. Published quotas and terms may change.

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
