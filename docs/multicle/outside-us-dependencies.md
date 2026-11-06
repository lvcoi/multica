# Outside-US dependency audit of the lvcoi/multica fork

- Issue: LVCO-4
- Author: Platform Architect
- Date: 2026-11-06
- Audited revision: `565c9128d` (upstream `main` when this task branched)
- Status: proposed, for review

## Summary

The goal is an internal Multicle instance with no runtime dependency on servers outside the US. This audit lists every outbound network call in the fork, records where each one goes, and proposes a replacement. It changes no code.

Findings:

1. **No server outside the US is called by default.** Every non-US endpoint in the fork is reached only after an operator or user installs or configures it. That covers Feishu/Lark, DingTalk, WeCom, Qoder, Telegram, Bilibili and the China-vendor agent CLIs. Section 4 lists each one and how to keep it off.
2. **One call leaves the instance with no configuration at all: self-host telemetry.** It goes to `telemetry.multica.ai` (`server/internal/selfhosttelemetry/client.go:11`). On 2026-11-06 that host resolved to AWS `us-west-2`, so it is not outside the US. It is still a call to a third party (Index Labs), and the endpoint cannot be changed. `DO_NOT_TRACK=1` turns it off (`server/cmd/server/main.go:331`).
3. **The shipped client binaries default to Multica's hosted cloud, not to our instance.** This covers:
   - the desktop app (`apps/desktop/src/shared/runtime-config.ts:18-20`)
   - the mobile production env (`apps/mobile/.env.production:4-5`)
   - the CLI's `multica setup` and `multica login --token` (`server/cmd/multica/cmd_agent.go:292-293`)

   Those hosts are in the US, but an unconfigured client signs in to the wrong service. A Multicle build must point them at our host.
4. **The update and release channels are hardcoded to `multica-ai` on GitHub.** That includes desktop auto-update, the CLI download, daemon self-update, the install scripts, and the image defaults in Helm and compose. The fork cannot publish its own builds through these channels without a code change (sections 6, 7 and 9).
5. **The agent runtimes carry the largest data-residency risk.** The daemon runs whichever agent CLI it finds on `PATH` (`server/internal/daemon/config.go:270`). Each CLI sends issue content and source code to its vendor's model endpoint. Several supported CLIs belong to China-based vendors (section 5).

Each non-US row can be fixed by configuration alone. Section 8 gives the configuration baseline. Section 9 lists code follow-ups for Fork Engineer that turn "do not configure" into "cannot configure".

## 1. Method

1. I ran three read-only sweeps of the tree:
   - `server/`: the Go server, CLI and daemon.
   - `apps/` and `packages/`: the web, desktop, mobile and docs apps.
   - Build and deploy files: `Dockerfile*`, `docker/`, `docker-compose*.yml`, `deploy/`, `scripts/`, `Makefile`, `.github/workflows/` and `.env.example`.

   Test files, locale JSON and Markdown content were excluded, except Markdown that defines the install path.
2. A repository-wide grep of URL literals (`https?://host`), and then of every China-linked and analytics hostname, checked the sweeps for misses.
3. Every cited line was re-read in the source before it went into this document.
4. Host location comes from DNS lookups (`nslookup`: forward, CNAME and reverse) run from a US resolver on 2026-11-06. No data was sent to a third-party geolocation service.

### Classification rules

| Class | Rule |
|---|---|
| **US** | The DNS answer names a US region (for example `ec2-*.us-west-2.compute.amazonaws.com`, `*.us-east-1.elb.amazonaws.com` or `*.lax54.r.cloudfront.net`), or the host is a service we run ourselves in a US region. |
| **Non-US** | Any of these holds: the host belongs to a mainland-China service (`.cn` domains, `qq.com`, `dingtalk.com`, `feishu.cn`, `bilibili.com`); its DNS routes through a China cloud (`alibabadns.com`, `aliyunga*.com`, `alikunlun.com`); or the operator is based outside the US. |
| **Unknown** | The answer is an anycast or global CDN edge (Cloudflare, Fastly, Akamai, Google front ends, GitHub) that does not reveal the origin, and the repo documents no region. |

Limits:

- DNS shows the edge we connect to. It does not show where data is processed or stored.
- A GeoDNS answer can differ from one resolver to another.
- In this document, "US" means the server our instance connects to is in the US. It does not mean LVCOI operates it. Rows operated by someone else are marked "3rd party" so the hosting ADR (LVCO-5) can decide whether to allow them.

## 2. Hosts resolved

| Host | DNS answer (2026-11-06, US resolver) | Class |
|---|---|---|
| `api.multica.ai`, `telemetry.multica.ai` | `52.32.212.9`, `35.81.124.63`, PTR `ec2-*.us-west-2.compute.amazonaws.com` | US (3rd party) |
| `multica.ai` | `216.150.1.1` (Vercel); zone on Route 53 (`ns-*.awsdns-*`) | Unknown (edge) |
| `www.multica.ai`, `app.multica.ai` | CNAME `*.vercel-dns-016.com` | Unknown (edge) |
| `static.multica.ai` | CNAME `dzqmt3nhxjla4.cloudfront.net`, PTR `server-65-9-126-42.lax54.r.cloudfront.net` | US (3rd party, CDN edge) |
| `multica-api.copilothub.ai` | `3.169.231.102` | Unknown |
| `multica-static.copilothub.ai` | CNAME `d3txgs98rjafw0.cloudfront.net` | Unknown (edge) |
| `us.i.posthog.com` | CNAME `posthog-ingress-prod-us-*.us-east-1.elb.amazonaws.com` | US (3rd party) |
| `open.feishu.cn`, `accounts.feishu.cn` | CNAME `l7-online-*.s.dsa.cdnbuild.net` | Non-US |
| `open.larksuite.com`, `accounts.larksuite.com` | CNAME Akamai (`a1815.r.akamai.net`, `e31084.a.akamaiedge.net`) | Non-US (ByteDance Lark; origin not visible) |
| `api.dingtalk.com` | CNAME `na-v6-cname.dingtalk.com.gds.alibabadns.com` | Non-US |
| `work.weixin.qq.com`, `openws.work.weixin.qq.com`, `qyapi.weixin.qq.com` | Tencent addresses `43.135.*`, `43.163.*` | Non-US |
| `api.qoder.com` | CNAME `ga-*.aliyunga0019.com` (Alibaba Cloud Global Accelerator) | Non-US |
| `api.qoder.com.cn` | CNAME `ga-*.aliyunga0018.com` | Non-US |
| `player.bilibili.com` | CNAME `player.bilibili.com.w.alikunlun.com` | Non-US |
| `api.telegram.org` | `149.154.166.110` | Non-US (operator outside US) |
| `clawhub.ai` | `216.150.1.1` (Vercel edge) | Unknown (edge) |
| `skills.sh`, `logos.composio.dev` | Vercel edge (`64.239.123.*`) | Unknown (edge) |
| `backend.composio.dev`, `api.resend.com`, `registry.npmjs.org`, `unpkg.com`, `api.openai.com` | Cloudflare edge (`104.*`, `162.159.*`) | Unknown (edge) |
| `slack.com` | `54.71.95.193`, no PTR | Unknown |
| `registry-1.docker.io` | `34.232.98.167`, no PTR | Unknown |
| `ghcr.io`, `api.github.com`, `raw.githubusercontent.com`, `objects.githubusercontent.com` | GitHub (`140.82.*`, `185.199.*`) | Unknown (edge) |
| `proxy.golang.org`, `sum.golang.org`, `fonts.googleapis.com`, `fonts.gstatic.com`, `accounts.google.com`, `oauth2.googleapis.com`, `www.googletagmanager.com` | Google front ends | Unknown (edge) |
| `dl-cdn.alpinelinux.org` | CNAME `dualstack.j.sni.global.fastly.net` | Unknown (edge) |

## 3. Server runtime: calls made with default configuration

| # | path:line | Calls | Resolves to | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| S1 | `server/internal/selfhosttelemetry/client.go:11`, started at `server/cmd/server/main.go:725,757-758` | A daily POST of a usage snapshot (version, bucketed counts) to `https://telemetry.multica.ai/v1/telemetry/events` | US, `us-west-2` (3rd party) | The endpoint is fixed (`SELF_HOSTING.md:17`). `DO_NOT_TRACK=1` turns it off (`main.go:331`). Compose passes the variable at `docker-compose.selfhost.yml:116`; `.env.example:750` leaves it empty | Set `DO_NOT_TRACK=1` in the instance env. Also block the host at the egress firewall |
| S2 | `server/internal/service/email.go:371` | No call. When `FRONTEND_ORIGIN` is unset, invite links fall back to `https://multica.ai` | n/a | `FRONTEND_ORIGIN` (`email.go:369`) | Always set `FRONTEND_ORIGIN` to our host |

Nothing else in the server makes a call without configuration. The daemon's update check (D1) runs only when the daemon points at `api.multica.ai` (`server/internal/daemon/config.go:682`).

## 4. Server runtime: opt-in integrations and providers

Each of these stays off until an env var, a workspace integration or a user action turns it on.

| # | path:line | Calls | Resolves to | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| S3 | `server/internal/integrations/lark/http_client.go:50`; `registration.go:51-52`; `types.go:57` | Feishu/Lark bot REST, WebSocket and app registration (`open.feishu.cn`, `accounts.feishu.cn`, `open.larksuite.com`, `accounts.larksuite.com`) | **Non-US** | Enabled by `MULTICA_LARK_SECRET_KEY` (`.env.example:652-667`). Base URLs can be overridden but must still reach Feishu/Lark | Leave `MULTICA_LARK_SECRET_KEY` unset. There is no US equivalent. Use Slack (S8) if a chat bridge is needed |
| S4 | `server/internal/integrations/dingtalk/client.go:28` | DingTalk bot API and stream (`api.dingtalk.com`) | **Non-US** | Enabled by `MULTICA_DINGTALK_SECRET_KEY` (`.env.example:673`). Host hardcoded | Leave the key unset |
| S5 | `server/internal/integrations/wecom/wecom_channel.go:40`; `install_provider.go:78` | WeCom bot WebSocket (`wss://openws.work.weixin.qq.com`) and install flow (`work.weixin.qq.com`) | **Non-US** | Enabled by `MULTICA_WECOM_SECRET_KEY` (`.env.example:377-381`). Host hardcoded | Leave the key unset |
| S6 | `server/internal/handler/qoder_catalog.go:20,25` | Qoder agent catalog. Only `api.qoder.com` and `api.qoder.com.cn` are allowed | **Non-US** (both behind Alibaba Cloud routing) | Only when a workspace creates a Qoder connection | Do not create Qoder connections. Follow-up F5 removes the option |
| S7 | `server/internal/integrations/telegram/api.go:26` | Telegram Bot API long-poll (`api.telegram.org`) | **Non-US** | Enabled by `MULTICA_TELEGRAM_SECRET_KEY` (`.env.example:686`). Host hardcoded | Leave the key unset |
| S8 | `server/internal/integrations/slack/byo_install.go:146-172` (slack-go SDK) | Slack API (`slack.com/api`) | Unknown | Enabled by `MULTICA_SLACK_SECRET_KEY` (`.env.example:679`) | Allowed if needed. Slack data residency is set in the Slack plan, outside this repo |
| S9 | `server/internal/handler/skill.go:849,1067-1069` | Skill registry search and import (`clawhub.ai/api/v1`) | Unknown (edge) | Hardcoded. Fires when a user searches or imports | Block at egress, or import skills only from an LVCOI-controlled GitHub repo |
| S10 | `server/internal/handler/skill.go:938,1228,1361,1989,2031,2166` | skills.sh and GitHub skill imports via `api.github.com` and `raw.githubusercontent.com` | Unknown (edge) | Hardcoded. Fires on user import | Same as S9 |
| S11 | `server/pkg/llm/client.go:118-122` (openai-go SDK default); env read at `server/cmd/server/router.go:447-449` | Internal LLM calls such as chat titles. The SDK default host is `https://api.openai.com/v1` | Unknown (edge) | `MULTICA_LLM_API_KEY`, `MULTICA_LLM_BASE_URL`, `MULTICA_LLM_DEFAULT_MODEL`. Disabled when both key and base URL are empty (`client.go:107-112,307-309`). **Setting only the key sends traffic to OpenAI** | Leave it off at first. To enable it, set `MULTICA_LLM_BASE_URL` to an OpenAI-compatible gateway we run in a US region, in front of a US-region model endpoint (Lawrence decides, section 10) |
| S12 | `server/internal/analytics/client.go:79`; `server/internal/handler/config.go:147-152` | PostHog product events, server side and browser side (the browser gets the key via `/api/config`) to `us.i.posthog.com` | US, `us-east-1` (3rd party) | Only when `POSTHOG_API_KEY` is set. `ANALYTICS_DISABLED` forces it off (`client.go:90`) | Leave the key unset and set `ANALYTICS_DISABLED=true` |
| S13 | `server/internal/service/email.go:178` (resend-go SDK) | Transactional email (`api.resend.com`) | Unknown (edge) | `RESEND_API_KEY` (`.env.example:256-257`) | Use SMTP (`SMTP_HOST`, `.env.example:274-281`) with a US relay we choose. SMTP takes priority over Resend |
| S14 | `server/internal/handler/auth.go:587,635`; `packages/views/auth/login-page.tsx:291` | Google OAuth (`accounts.google.com`, `oauth2.googleapis.com`, `www.googleapis.com`) | Unknown (edge) | Only when `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are set | Leave unset and use email-code login. A later ADR decides the auth model |
| S15 | `server/internal/storage/s3.go:57`; `server/internal/auth/cloudfront.go:45-106` | AWS S3 (default region `us-west-2`), Secrets Manager and CloudFront | US when the region is a US region | `S3_BUCKET` (empty means local disk), `S3_REGION`, `AWS_ENDPOINT_URL`, `CLOUDFRONT_*` (`.env.example:387-409`) | Local disk for the first instance, or a bucket we own in a US region (cost decision, section 10) |
| S16 | `server/internal/handler/github.go:36,489`; `server/internal/integrations/ghsnapshot/client.go:39` | GitHub App API (`api.github.com`) | Unknown (edge) | Only when `GITHUB_APP_ID` and `GITHUB_APP_PRIVATE_KEY` are set | Allowed. `lvcoi/multica` already lives on GitHub |
| S17 | `server/pkg/composio/client.go:16`; `server/internal/integrations/composio/service.go:69`; `packages/views/common/composio-toolkit-logo.tsx:6` | Composio API (`backend.composio.dev`) and logos (`logos.composio.dev`) | Unknown (edge) | `COMPOSIO_API_KEY` plus a feature flag | Leave unset |
| S18 | `server/internal/auth/cloud_pat.go:180`; `server/internal/cloudruntime/client.go`; `server/internal/entitlement/client.go:241`; `server/internal/seatcapacity/client.go:190` | Multica Cloud (Fleet, entitlements, seats) at whatever URL `MULTICA_CLOUD_URL` names | Depends on value | `MULTICA_CLOUD_URL`, empty by default (`.env.example:238`) | Leave empty |
| S19 | `server/internal/integrations/vcs/{gitlab,forgejo}.go`; `server/internal/service/plugin_hook.go:262-281`; `server/pkg/remotemcp/*` | Git hosts, plugin webhooks and remote MCP servers supplied by the operator or a user | Depends on value | User or operator config, validated by `ValidatePublicHTTPSEndpoint` | Policy: allow only US-hosted endpoints. Enforce with the egress allowlist (section 8) |

## 5. Daemon, CLI and agent runtimes (on the user's machine)

| # | path:line | Calls | Resolves to | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| D1 | `server/internal/cli/update.go:34,55-61,283,372`; `server/internal/daemon/auto_update.go:18,69,117-129` | Checks releases at `api.github.com/repos/multica-ai/multica/releases/*` and downloads the binary from GitHub release assets. Runs on `multica update`, and in the daemon every 6 hours when its server is `api.multica.ai` | Unknown (edge), upstream repo | `MULTICA_RELEASE_API_BASE_URL`, `MULTICA_RELEASE_DOWNLOAD_BASE_URL` (`update.go:39-40`), `MULTICA_DAEMON_AUTO_UPDATE` (`daemon/config.go:607`) | Daemons pointed at our host already skip auto-update. Set `MULTICA_DAEMON_AUTO_UPDATE=false` anyway. Point both release variables at the fork's releases once they exist (F3) |
| D2 | `server/internal/cli/update.go:352` | Runs `brew upgrade multica-ai/tap/multica` | Unknown (edge) | No | Do not install the CLI from the upstream tap. Install from the fork's release |
| D3 | `server/cmd/multica/cmd_agent.go:292-293,356-361`; `server/cmd/multica/cmd_setup.go:148-149` | `multica setup` in cloud mode, and `multica login --token` with no server configured, use `https://api.multica.ai` and `https://multica.ai` | US, `us-west-2` (3rd party) | `--server-url` / `MULTICA_SERVER_URL` (`cmd_agent.go:318`), `MULTICA_APP_URL`, `multica setup self-host` | Multicle onboarding docs use only `multica setup self-host` against our host. F2 changes the compiled default |
| D4 | `server/internal/daemon/repocache/cache.go:612` | `git clone --bare` of task repositories | Depends on remote | Per-repository config | Allowed for LVCOI repos on GitHub |
| D5 | `server/internal/daemon/config.go:270` (CLIs found on `PATH`) | Each agent CLI calls its vendor's model API with issue text, comments and repository code | Depends on CLI and its config | Which CLIs are installed on the daemon host, plus each CLI's own provider settings | See below |

### D5 in detail: agent CLIs

If the daemon finds any of these CLIs on `PATH`, it can launch it (`scripts/agent-cli-command-names.txt`): `agy claude codebuddy codearts codex copilot cursor-agent deveco devin dim dsh grok hermes kimi kiro-cli mcode muse omp openclaw opencode pi qodercli qoderclicn qwen qwenpaw reasonix traecli zeroclaw zcode`.

The repository does not record where each vendor serves its models, so this audit cannot classify the CLIs by host. Some names do identify a China-based vendor or a China-region build. Treat these three as **non-US**:

- `qoderclicn`: Qoder CN, the same service as S6.
- `dsh`: DeepSeek Harness (`server/internal/daemon/config.go:1021`).
- `kimi`: Moonshot AI's Kimi.

Treat every other CLI as **unknown** until its vendor's documented region is checked.

**Proposed replacement:**

1. Allow only `claude` on daemon hosts at first.
2. Route it to a US region through Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry, not `api.anthropic.com`.
   - Lawrence's reference docs say each of these platforms offers a choice of regions (`10-enterprise-deployment-overview.md:74-80`).
   - With these providers, model traffic and authentication go to the provider instead of `api.anthropic.com` (`11-enterprise-network-configuration.md:257`).
3. Pin the provider with the managed setting `allowedProviders: ["bedrock"]` (`10-enterprise-deployment-overview.md:182`).
4. Set `skipWebFetchPreflight: true`.
   - On a third-party provider, Claude Code sends no operational telemetry to the Datadog intake hosts (`11-enterprise-network-configuration.md:246-253`).
   - WebFetch, however, still calls `api.anthropic.com` for its domain safety check unless this setting is on (`11-enterprise-network-configuration.md:257`).
5. Do not install `qodercli`, `qoderclicn`, `dsh`, `kimi` or any unchecked CLI on daemon hosts. The daemon has no runtime allowlist; it uses whatever it finds (follow-up F6).

Choosing the provider and account is a cost decision for Lawrence (section 10).

## 6. Client apps (desktop, mobile, web)

| # | path:line | Calls | Resolves to | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| C1 | `apps/desktop/src/shared/runtime-config.ts:16-21` | Desktop default API, WebSocket and app URLs: `api.multica.ai`, `wss://api.multica.ai/ws`, `multica.ai` | US `us-west-2` / edge (3rd party) | In packaged builds, only through `~/.multica/desktop.json` (`apps/desktop/src/main/runtime-config-loader.ts:26-44`). Dev builds use `VITE_API_URL`, `VITE_WS_URL`, `VITE_APP_URL` | Ship a `desktop.json` pointing at our host with every install. F1 makes the default a build-time variable |
| C2 | `apps/desktop/electron-builder.yml:130-139`; `apps/desktop/src/main/updater.ts:18-53` | electron-updater checks GitHub Releases `multica-ai/multica` 5 s after start, then hourly, and downloads automatically | Unknown (edge), upstream repo | Only the user's in-app auto-update toggle. The feed is fixed in the yml | Turn auto-update off on installed clients. F3 moves the feed to `lvcoi/multica` |
| C3 | `apps/desktop/src/main/cli-bootstrap.ts:17-18,48,103` | Downloads the `multica` CLI from `github.com/multica-ai/multica/releases/latest/download` when the bundled one is missing | Unknown (edge), upstream repo | No | F3 |
| C4 | `packages/views/runtimes/components/update-section.tsx:15-16` | The browser fetches `api.github.com/repos/multica-ai/multica/releases/latest` when the runtime CLI section opens | Unknown (edge) | No | Harmless, since no user data is sent, but it reports the upstream version. F3 |
| C5 | `apps/mobile/.env.production:4-5` (committed) | The mobile production build uses `api.multica.ai` and `multica.ai` | US / edge (3rd party) | `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_WEB_URL` at build time | Build Multicle mobile with our values, or do not ship mobile for the internal instance |
| C6 | `apps/desktop/.env.staging:4,11`; `apps/mobile/.env.staging:5,9`; commented example at `.env.example:707`; doc comment at `server/internal/storage/s3.go:261` | Upstream staging hosts `multica-api.copilothub.ai`, `multica-app.copilothub.ai`, `multica-static.copilothub.ai` | Unknown | Same env vars as C1 and C5 | Never build with `.env.staging`. F4 removes these files from the fork |
| C7 | `packages/core/analytics/index.ts:113-114,147` | posthog-js events and exception autocapture | US `us-east-1` (3rd party) | Starts only if `/api/config` returns a PostHog key (S12) | Covered by S12 |
| C8 | `apps/web/lib/google-analytics.ts:70`; `apps/web/app/layout.tsx:140,178` | Google Analytics (`www.googletagmanager.com`) | Unknown (edge) | `GOOGLE_ANALYTICS_ID` (`.env.example:703`) | Leave unset |
| C9 | `apps/web/features/landing/utils/use-github-stars.ts:31`; `apps/web/features/landing/utils/github-release.ts:39-40,70` | Landing-page star count and the `/download` release list, from `api.github.com` | Unknown (edge) | No | Signed-in use never hits these pages. Acceptable for internal use |
| C10 | `apps/docs/components/video-embed.tsx:35,44`, used at `apps/docs/content/docs/index.zh.mdx:32` | A Bilibili iframe on the Chinese docs index page | **Non-US** | No | Do not deploy `apps/docs` for the internal instance (leave `DOCS_URL` empty), or deploy English docs only |
| C11 | `apps/web/app/layout.tsx:160-166`; `apps/desktop/src/renderer/src/main.tsx:39-41` | The `unpkg.com/react-grab` script | Unknown (edge) | Loads only in development builds with `VITE_REACT_GRAB` set | None. Never present in production builds |
| C12 | `apps/web/next.config.ts:51-96`; `apps/web/config/runtime-urls.ts:57-68` | Web rewrites to the API and docs upstreams | Our hosts | `REMOTE_API_URL`, `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`, `DOCS_URL` | Set all four to our hosts |

Plain hyperlinks to `multica.ai/docs`, `/changelog` and `/download` (for example `packages/views/layout/help-links.ts:1-7`) load nothing until a user clicks one. They are attribution, and under the licensing guardrail they stay unchanged.

## 7. Build and supply chain

These calls happen when we build images or pull them, not while the instance serves users. They affect availability and integrity, and they count against "no dependency" in the strict sense.

| # | path:line | Pulls from | Resolves to | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| B1 | `docker-compose.selfhost.yml:56,235`; `.env.example:244-246`; `deploy/helm/multica/values.yaml:6,12` | Upstream images `ghcr.io/multica-ai/multica-backend` and `multica-web`, tag `latest` | Unknown (edge), upstream | `MULTICA_BACKEND_IMAGE`, `MULTICA_WEB_IMAGE`, `MULTICA_IMAGE_TAG`. `.env.example` sets them explicitly, so a copied `.env` pins upstream | Build from the fork (`docker-compose.selfhost.build.yml`), push to a registry we own in a US region, and pin by digest. Upstream images are not built from our tree |
| B2 | `Dockerfile:2,27`; `Dockerfile.web:6,29,59`; `docker-compose.selfhost.yml:37`; `docker-compose.yml:5` | Docker Hub base images `golang:1.27.3-alpine`, `alpine:3.21`, `node:22.23.4-alpine`, `pgvector/pgvector:pg17` | Unknown | `FROM` lines are hardcoded, with no registry `ARG` | Mirror them into our registry and pin by digest (F7) |
| B3 | `Dockerfile:4,29`; `Dockerfile.web:66` | Alpine packages (`dl-cdn.alpinelinux.org`) | Unknown (Fastly) | No | Build-time only. Accept, or use a pinned mirror |
| B4 | `Dockerfile:10`; `go.mod` (no proxy settings) | Go modules via `proxy.golang.org` and `sum.golang.org` | Unknown (Google) | `GOPROXY` on the build host | Accept for now. Downloads are verified against `go.sum` |
| B5 | `Dockerfile.web:23-26`; `.npmrc` (only `shamefully-hoist=true`) | npm packages and pnpm via `registry.npmjs.org` | Unknown (Cloudflare) | npm registry and `COREPACK_NPM_REGISTRY` on the build host | Accept for now. The lockfile pins integrity hashes |
| B6 | `apps/web/app/layout.tsx:3,41-72`; `apps/web/app/(landing)/layout.tsx:11-15`; `apps/docs/app/[lang]/layout.tsx:20-39` | `next/font/google` downloads fonts from Google **at build time** and serves them from our host at runtime | Unknown (Google) | No | Build-time only. Accept |
| B7 | `scripts/install.sh:17-20,152-163`; `scripts/install.ps1:15-16,89,250-267` | Upstream installers clone and download releases from `github.com/multica-ai/multica` and use the upstream Homebrew tap. `install.sh` checks no checksum | Unknown (edge), upstream | Only `MULTICA_SELFHOST_REF` and the install directories | Do not use the upstream one-line installers for Multicle. Install from the fork. F3 |
| B8 | `.github/workflows/release.yml:147-459`; `.goreleaser.yml:74-83` | Release CI publishes images and charts to `ghcr.io/${{ github.repository_owner }}`, which follows the fork. Desktop assets and the brew formula go to the hardcoded `multica-ai/multica` and `multica-ai/homebrew-tap` | n/a | Partly | F3 |

**The tree configures no Chinese package mirror or registry.** There is no npmmirror, `goproxy.cn`, Aliyun, Tuna or `registry.cn-*` anywhere.

## 8. Recommended baseline for the internal instance

This baseline is configuration only, with no code changes. With it, the server already meets the goal of no runtime dependency outside the US.

```dotenv
# Server
DO_NOT_TRACK=1                 # S1
ANALYTICS_DISABLED=true        # S12, C7
POSTHOG_API_KEY=
GOOGLE_ANALYTICS_ID=           # C8
FRONTEND_ORIGIN=https://<our-host>   # S2
MULTICA_CLOUD_URL=             # S18
MULTICA_LARK_SECRET_KEY=       # S3
MULTICA_DINGTALK_SECRET_KEY=   # S4
MULTICA_WECOM_SECRET_KEY=      # S5
MULTICA_TELEGRAM_SECRET_KEY=   # S7
COMPOSIO_API_KEY=              # S17
RESEND_API_KEY=                # S13; use SMTP_* to a US relay instead
MULTICA_LLM_API_KEY=           # S11; off until a US gateway exists
MULTICA_LLM_BASE_URL=
MULTICA_BACKEND_IMAGE=<our-registry>/<backend-image>@sha256:<digest>   # B1
MULTICA_WEB_IMAGE=<our-registry>/<web-image>@sha256:<digest>

# Daemon hosts
MULTICA_SERVER_URL=https://<our-host>   # D3
MULTICA_DAEMON_AUTO_UPDATE=false        # D1
```

On daemon hosts, install only `claude`, configured for a US-region provider (D5). Ship every desktop install with a `~/.multica/desktop.json` that points at our host, and turn auto-update off (C1, C2).

**Defense in depth.** Configuration alone works only as long as nobody forgets a setting. Run the server and daemon hosts behind a default-deny egress allowlist that contains only:

- our database, object store and SMTP relay
- the chosen model endpoint
- `api.github.com` and `github.com`, for repositories

A future setting change, or a code path this audit missed, then fails closed instead of leaking. The allowlist also covers S9, S10 and S19, which users can trigger. The hosting ADR (LVCO-5) should own the exact list.

## 9. Follow-ups for Fork Engineer (code, not done here)

Each item hardens one of the configuration fixes above. None of them changes branding or attribution.

| ID | Change | Why |
|---|---|---|
| F1 | Make the desktop default API, WebSocket and app URLs build-time variables in `apps/desktop/src/shared/runtime-config.ts:16-21`, so a packaged Multicle build needs no `desktop.json` | C1. If the file is missing, the client signs in to Multica's cloud |
| F2 | Make the CLI's `defaultCloudServerURL` and `defaultCloudAppURL` (`server/cmd/multica/cmd_agent.go:292-293`) build-time `-ldflags` values | D3 |
| F3 | Point the release channels at `lvcoi/multica`:<br>- `apps/desktop/electron-builder.yml:132-133`<br>- `apps/desktop/src/main/cli-bootstrap.ts:17-18`<br>- `packages/views/runtimes/components/update-section.tsx:15-16`<br>- the release-repo path in `server/internal/cli/update.go`<br>- `scripts/install.{sh,ps1}`<br>- `.goreleaser.yml:79`<br>- the image defaults in compose, Helm and `.env.example`<br><br>Also add a checksum check to `install.sh` | C2-C4, D1, B1, B7, B8 |
| F4 | Remove `apps/desktop/.env.staging` and `apps/mobile/.env.staging` from the fork | C6 |
| F5 | Add a build flag or env switch that removes the China-region integrations from the server and hides their UI tabs: Lark/Feishu, DingTalk, WeCom, the Qoder catalog, and the `qoderclicn`, `dsh` and `kimi` runtimes | S3-S6, D5. "Do not configure" becomes "cannot configure" |
| F6 | Add a daemon allowlist for agent runtimes, for example `MULTICA_ALLOWED_AGENTS=claude` | D5. Today the daemon runs whatever it finds on `PATH` |
| F7 | Add a registry `ARG` to the `FROM` lines in `Dockerfile` and `Dockerfile.web`, so base images can come from our mirror | B2 |

## 10. Decisions for Lawrence

Each of these spends money or picks a vendor, so they appear here as options. No action has been taken on any of them.

1. **Model provider for the agents (D5) and the server LLM (S11):** Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry, in a US region. All three bill pay-as-you-go through the cloud account (`10-enterprise-deployment-overview.md:60-71`). This audit did not gather prices. ADR-001 (LVCO-5) should carry the comparison.
2. **Object storage (S15):** local disk (no extra cost) or an S3 bucket in a US region.
3. **SMTP relay (S13):** which US provider to use, if email login is needed.
4. **Container registry (B1, B2):** GHCR under `lvcoi`, where the repo already lives, or a cloud registry in the hosting account.

## Not covered

- Where each agent CLI vendor processes data (D5). Answering this needs each vendor's own documentation.
- Connections to hosts that come from user content at runtime, such as markdown images and attachments at arbitrary URLs. The egress allowlist in section 8 is the control for these.
- Mobile push and over-the-air updates. None are configured: `apps/mobile` has no `expo-updates`, no `expo-notifications` and no `eas.json`.

This is a docs-only change, so no code tests were run.
