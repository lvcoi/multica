# Outside-US dependency audit of the lvcoi/multica fork

- Issue: LVCO-4
- Author: Platform Architect
- Date: 2026-11-06 (revision 2, after review on PR #2)
- Audited revision: `565c9128d` (upstream `main` when this task branched)
- Status: proposed, for review

## Scope

This audit covers the `lvcoi/multica` fork only. It inventories the outbound network calls made by the **internal fork build**: the build LVCOI runs as its **internal instance**, for LVCOI internal use only, under the Multica License, with all Multica branding and attribution kept.

This document was written by reading Multica source, so it is not an input to Multicle specs. Multicle is the separate clean-room MIT codebase decided on LVCO-2.

## Summary

Goal: run the internal instance with no runtime dependency on servers outside the US. This audit lists every outbound call, classifies where it goes, and proposes a replacement. No code is changed here.

Under the classification rule in section 1, which counts both the server's location and its operator:

1. **With default configuration, the server makes one outbound call, and that call is Non-US.** The call is self-host telemetry to `telemetry.multica.ai` (`server/internal/selfhosttelemetry/client.go:11`).
   - The server answering it is in AWS `us-west-2`.
   - The service is run by Index Labs (Hong Kong) Limited (`LICENSE:15-16`, `NOTICE:2`).
   - The endpoint cannot be changed.
   - Fix: set `DO_NOT_TRACK=1` (`server/cmd/server/main.go:331`) and block the host at egress.
2. **Every shipped client defaults to the same Hong Kong-operated service** (Non-US by operator; servers in `us-west-2`):
   - the desktop app (`apps/desktop/src/shared/runtime-config.ts:18-20`)
   - the mobile production env (`apps/mobile/.env.production:4-5`)
   - the CLI's `multica setup` and `multica login --token` (`server/cmd/multica/cmd_agent.go:292-293`)

   An internal fork build has to point all three at our host.
3. **Clients also call Unknown-class hosts by default.** The desktop updater polls GitHub Releases every hour (C2), and the runtime page fetches the latest release from `api.github.com` (C4). Neither call sends user data, but neither origin location is established. This audit therefore does not claim that clients call nothing outside the US.
4. **The server itself can send task content to a Non-US service: Qoder Cloud Agents** (`server/internal/qoderruntime/bridge.go:498-502`).
   - The runtime is gated by `MULTICA_VCS_SECRET_KEY`, the same key that enables GitLab and Forgejo.
   - One Qoder endpoint has no gate at all (`server/internal/handler/qoder_catalog.go:143-197`).
   - Configuration alone does not close Qoder. The egress block is required (S6).
5. **The agent runtimes are the largest data-residency risk.**
   - The daemon launches agent CLIs, and each CLI sends issue text and code to its vendor.
   - The repo's own records name 13 supported CLIs whose operator or model provider is China-based (section 5).
   - Controlling what is installed on the daemon host is not enough. A workspace admin or an agent's owner can redirect a CLI with per-agent `custom_env`, or add a custom runtime profile, from the UI.
6. **The update and release channels are hardcoded to `multica-ai` on GitHub.** The fork cannot publish its own builds through them without code changes (sections 6, 7 and 9).

Configuration closes every Non-US row except three. For these, the egress allowlist in section 8 is the control until the code follow-ups in section 9 land:

- the ungated Qoder endpoint (S6)
- the Bilibili docs embed (C10), which is avoided by not deploying the docs app
- agent redirection through `custom_env` and runtime profiles (D5)

## 1. Method

1. **Sweeps.** I ran three read-only sweeps:
   - `server/`: the Go server, CLI and daemon.
   - `apps/` and `packages/`: the web, desktop, mobile and docs apps.
   - The build and deploy files: `Dockerfile*`, `docker/`, `docker-compose*.yml`, `deploy/`, `scripts/`, `Makefile`, `.github/workflows/` and `.env.example`.

   The sweeps excluded test files and locale JSON. They also excluded Markdown content, except where it defines an install path or names a runtime's vendor.
2. **Grep cross-check.** I grepped the whole repository for URL literals (`https?://host`), then for every China-linked and analytics hostname, to catch anything the sweeps missed.
3. **Citations.** Every cited line was re-read in the source before it went into this document. The review on PR #2 independently checked 27 citations and 7 DNS answers.
4. **Host location.** Locations come from DNS lookups (`nslookup` forward, CNAME and reverse) from a US resolver on 2026-11-06. No data was sent to a third-party geolocation service.
5. **Operator.** Operators are identified from records in this repo (`LICENSE`, `NOTICE`, `CLI_AND_DAEMON.md`, code comments) or from the host's domain. Each one is cited.

### Classification rule

The same rule applies to every row. A host is **Non-US** if either test below says so.

| Test | Non-US when |
|---|---|
| Operator | The repo records that the operator or the model provider is based outside the US, or the host is on a mainland-China domain (`.cn`, `qq.com`, `dingtalk.com`, `feishu.cn`, `bilibili.com`) |
| Routing | The DNS answer routes through a China cloud (`alibabadns.com`, `aliyunga*.com`, `alikunlun.com`, `volcddos.com`) |

If neither test applies:

- **US** means the DNS answer names a US region, for example `ec2-*.us-west-2.compute.amazonaws.com`, `*.us-east-1.elb.amazonaws.com` or `*.lax54.r.cloudfront.net`.
- **Unknown** means the answer is an anycast or global CDN edge (Cloudflare, Fastly, Akamai, Google front ends, GitHub) that does not reveal the origin, or the repo does not record the operator.

Where server location and operator disagree, both are shown, for example "Non-US (operator); server `us-west-2`".

Limits:

- DNS shows the edge we connect to, not where data is processed or stored.
- GeoDNS answers can differ between resolvers.

## 2. Hosts resolved

| Host | DNS answer (2026-11-06, US resolver) | Operator (source) | Class |
|---|---|---|---|
| `api.multica.ai`, `telemetry.multica.ai` | `52.32.212.9`, `35.81.124.63`, PTR `ec2-*.us-west-2.compute.amazonaws.com` | Index Labs (Hong Kong) Limited (`LICENSE:15-16`) | **Non-US (operator)**; server `us-west-2` |
| `static.multica.ai` | CNAME `dzqmt3nhxjla4.cloudfront.net`, PTR `server-65-9-126-42.lax54.r.cloudfront.net` | Index Labs (Hong Kong) Limited | **Non-US (operator)**; CDN edge in the US |
| `multica.ai`, `www.multica.ai`, `app.multica.ai` | `216.150.1.1` and CNAME `*.vercel-dns-016.com` (Vercel); zone on Route 53 | Index Labs (Hong Kong) Limited | **Non-US (operator)** |
| `multica-api.copilothub.ai` | `3.169.231.102` | Not recorded | Unknown |
| `multica-static.copilothub.ai` | CNAME `d3txgs98rjafw0.cloudfront.net` | Not recorded | Unknown (edge) |
| `us.i.posthog.com` | CNAME `posthog-ingress-prod-us-*.us-east-1.elb.amazonaws.com` | PostHog | US (`us-east-1`) |
| `open.feishu.cn`, `accounts.feishu.cn` | CNAME `l7-online-*.s.dsa.cdnbuild.net` | ByteDance Feishu (`.cn`) | **Non-US** |
| `open.larksuite.com`, `accounts.larksuite.com` | CNAME Akamai (`a1815.r.akamai.net`, `e31084.a.akamaiedge.net`) | ByteDance Lark, the international Feishu (`server/internal/integrations/lark/types.go:54-57`) | **Non-US (operator)** |
| `api.dingtalk.com` | CNAME `na-v6-cname.dingtalk.com.gds.alibabadns.com` | Alibaba DingTalk | **Non-US** |
| `work.weixin.qq.com`, `openws.work.weixin.qq.com`, `qyapi.weixin.qq.com` | Tencent addresses `43.135.*`, `43.163.*` | Tencent WeCom | **Non-US** |
| `api.qoder.com` | CNAME `ga-*.aliyunga0019.com` (Alibaba Cloud Global Accelerator) | Qoder | **Non-US (routing)** |
| `api.qoder.com.cn` | CNAME `ga-*.aliyunga0018.com` | Qoder CN | **Non-US** |
| `code.kimi.com` | CNAME `poevnnxxg2ygauv0f2cc.volcddos.com`, `103.143.17.156` | Moonshot Kimi (`CLI_AND_DAEMON.md:288`) | **Non-US** |
| `player.bilibili.com` | CNAME `player.bilibili.com.w.alikunlun.com` | Bilibili | **Non-US** |
| `api.telegram.org` | `149.154.166.110` | Telegram, operator outside the US | **Non-US (operator)** |
| `clawhub.ai` | `216.150.1.1` (Vercel edge) | Not recorded | Unknown (edge) |
| `skills.sh`, `logos.composio.dev` | Vercel edge (`64.239.123.*`) | Not recorded | Unknown (edge) |
| `backend.composio.dev`, `api.resend.com`, `registry.npmjs.org`, `unpkg.com`, `api.openai.com` | Cloudflare edge (`104.*`, `162.159.*`) | Not recorded in the repo | Unknown (edge) |
| `slack.com` | `54.71.95.193`, no PTR | Slack | Unknown |
| `registry-1.docker.io` | `34.232.98.167`, no PTR | Docker | Unknown |
| `ghcr.io`, `api.github.com`, `raw.githubusercontent.com`, `objects.githubusercontent.com` | GitHub (`140.82.*`, `185.199.*`) | GitHub | Unknown (edge) |
| `proxy.golang.org`, `sum.golang.org`, `fonts.googleapis.com`, `fonts.gstatic.com`, `accounts.google.com`, `oauth2.googleapis.com`, `www.googletagmanager.com` | Google front ends | Google | Unknown (edge) |
| `dl-cdn.alpinelinux.org` | CNAME `dualstack.j.sni.global.fastly.net` | Alpine | Unknown (edge) |

## 3. Server runtime: calls made with default configuration

| # | path:line | Calls | Class | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| S1 | `server/internal/selfhosttelemetry/client.go:11`, started at `server/cmd/server/main.go:725,757-758` | Once a day, POSTs a usage snapshot (version, bucketed counts) to `https://telemetry.multica.ai/v1/telemetry/events` | **Non-US (operator)**; server `us-west-2` | The endpoint is fixed (`SELF_HOSTING.md:17`). `DO_NOT_TRACK=1` turns the call off (`main.go:331`). Compose forwards the variable at `docker-compose.selfhost.yml:116`; `.env.example:750` leaves it empty | Set `DO_NOT_TRACK=1` and block the host at egress |
| S2 | `server/internal/service/email.go:371` | No call. When `FRONTEND_ORIGIN` is unset, invite links fall back to `https://multica.ai` | n/a | `FRONTEND_ORIGIN` (`email.go:369`) | Always set `FRONTEND_ORIGIN` to our host |

No other server code makes a call without configuration. The daemon's update check (D1) runs only when the daemon points at `api.multica.ai` (`server/internal/daemon/config.go:682`).

## 4. Server runtime: opt-in integrations and providers

| # | path:line | Calls | Class | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| S3 | `server/internal/integrations/lark/http_client.go:50`; `registration.go:51-52`; `types.go:57` | Feishu/Lark bot REST, WebSocket and app registration | **Non-US** | Turned on by `MULTICA_LARK_SECRET_KEY` (`.env.example:652-667`) | Leave the key unset. Use Slack (S8) if a chat bridge is needed |
| S4 | `server/internal/integrations/dingtalk/client.go:28` | DingTalk bot API and stream | **Non-US** | Turned on by `MULTICA_DINGTALK_SECRET_KEY` (`.env.example:673`). The host is hardcoded | Leave the key unset |
| S5 | `server/internal/integrations/wecom/wecom_channel.go:40`; `install_provider.go:78` | WeCom bot WebSocket and install flow | **Non-US** | Turned on by `MULTICA_WECOM_SECRET_KEY` (`.env.example:377-381`). The host is hardcoded | Leave the key unset |
| S6 | Runtime bridge: `server/internal/qoderruntime/bridge.go:498-502` posts the task prompt and images to `/sessions/{id}/events`, and `bridge.go:543` creates sessions. The bridge is started by `RunQoderConnections` (`server/cmd/server/main.go:755`; `server/internal/handler/qoder_connection.go:369-372`). Catalog: `server/internal/handler/qoder_catalog.go:20,25,143-197` | **Qoder Cloud Agents as a server-side agent runtime.** The server sends task content (prompt, images, task and workspace IDs) to Qoder sessions, with no daemon or CLI involved. Allowed hosts are `api.qoder.com` and `api.qoder.com.cn` | **Non-US** | **Saved connections and the bridge** need `MULTICA_VCS_SECRET_KEY` (`qoder_connection.go:131,370`; `.env.example:764-766`). The **same key** turns on GitLab and Forgejo (S19). Once it is set, any workspace owner or admin can connect Qoder (`server/cmd/server/router.go:1869-1874`). Compose already defaults `MULTICA_VCS_INTEGRATION_ENABLED` to `true` (`docker-compose.selfhost.yml:195`).<br><br>**`POST /qoder/environments` checks no key.** With no saved connection, it takes the token from the request body and calls Qoder (`qoder_catalog.go:155-197`). It sends only that token and a catalog query. This was found by reading the code, not by running it | Leave `MULTICA_VCS_SECRET_KEY` unset unless GitLab or Forgejo is needed.<br><br>Whatever the key setting, **block `api.qoder.com` and `api.qoder.com.cn` at egress**; configuration alone does not close this path. F5 removes the Qoder routes |
| S7 | `server/internal/integrations/telegram/api.go:26` | Telegram Bot API long-poll | **Non-US (operator)** | Turned on by `MULTICA_TELEGRAM_SECRET_KEY` (`.env.example:686`) | Leave the key unset |
| S8 | `server/internal/integrations/slack/byo_install.go:146-172` | Slack API | Unknown | Turned on by `MULTICA_SLACK_SECRET_KEY` (`.env.example:679`) | Allowed if needed. Slack data residency is a Slack plan setting |
| S9 | `server/internal/handler/skill.go:849,1067-1069` | Skill registry search and import at `clawhub.ai/api/v1` | Unknown (edge) | Hardcoded. Runs when a user searches or imports | Block `clawhub.ai` at egress |
| S10 | `server/internal/handler/skill.go:938,1228,1361,1989,2031,2166` | skills.sh and GitHub skill imports. A skills.sh URL is fetched from `raw.githubusercontent.com` (`skill.go:1214-1228`) | Unknown (edge) | Hardcoded. Runs when a user imports | GitHub stays on the allowlist, so egress does not block this. Policy: import only from repos LVCOI controls. F5 can add an allowlist of import sources |
| S11 | `server/pkg/llm/client.go:118-122` (openai-go SDK default `https://api.openai.com/v1`); env read at `server/cmd/server/router.go:447-449` | Internal LLM calls, such as generating chat titles | Unknown (edge) | `MULTICA_LLM_API_KEY`, `MULTICA_LLM_BASE_URL`, `MULTICA_LLM_DEFAULT_MODEL`. Off when both the key and the base URL are empty (`client.go:107-112,307-309`). **Setting only the key sends traffic to OpenAI** | Leave it off. To turn it on, set `MULTICA_LLM_BASE_URL` to an OpenAI-compatible gateway we run in a US region, in front of a model hosted in a US region (Lawrence decides; see section 10) |
| S12 | `server/internal/analytics/client.go:72-79`; `server/internal/handler/config.go:147-152` | PostHog events, server side and browser side | US (`us-east-1`) | Active only when `POSTHOG_API_KEY` is set; an empty key gives the no-op client (`client.go:72-76`). `ANALYTICS_DISABLED` forces it off (`client.go:90`). `docker-compose.selfhost.yml` forwards neither variable and has no `env_file`, so under compose analytics is always off and `ANALYTICS_DISABLED` has no effect | Under compose: nothing to do. Elsewhere (a directly run binary, custom manifests): leave the key unset and set `ANALYTICS_DISABLED=true` |
| S13 | `server/internal/service/email.go:178` (resend-go SDK) | Transactional email through `api.resend.com` | Unknown (edge) | `RESEND_API_KEY` (`.env.example:256-257`) | Use SMTP (`SMTP_HOST`, `.env.example:274-281`) through a US relay. SMTP takes priority over Resend |
| S14 | `server/internal/handler/auth.go:587,635`; `packages/views/auth/login-page.tsx:291` | Google OAuth | Unknown (edge) | Only when `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are set | Leave unset and use email-code login. A later ADR decides auth |
| S15 | `server/internal/storage/s3.go:57`; `server/internal/auth/cloudfront.go:45-106` | AWS S3 (default region `us-west-2`), Secrets Manager, CloudFront | US when the region is a US region | `S3_BUCKET` (empty means local disk), `S3_REGION`, `AWS_ENDPOINT_URL`, `CLOUDFRONT_*` (`.env.example:387-409`) | Start on local disk, or use a bucket we own in a US region (a cost decision, section 10) |
| S16 | `server/internal/handler/github.go:36,489`; `server/internal/integrations/ghsnapshot/client.go:39` | GitHub App API | Unknown (edge) | Only when `GITHUB_APP_ID` and `GITHUB_APP_PRIVATE_KEY` are set | Allowed. The repo already lives on GitHub |
| S17 | `server/pkg/composio/client.go:16`; `server/internal/integrations/composio/service.go:69`; `packages/views/common/composio-toolkit-logo.tsx:6` | Composio API and logos | Unknown (edge) | `COMPOSIO_API_KEY` plus a feature flag | Leave unset |
| S18 | `server/internal/auth/cloud_pat.go:180`; `server/internal/cloudruntime/client.go`; `server/internal/entitlement/client.go:241`; `server/internal/seatcapacity/client.go:190` | Multica Cloud (Fleet, entitlements, seats) | Depends on the value. The upstream service would be Non-US (operator) | `MULTICA_CLOUD_URL`, empty by default (`.env.example:238`) | Leave empty |
| S19 | `server/internal/integrations/vcs/{gitlab,forgejo}.go`; `server/internal/service/plugin_hook.go:262-281`; `server/pkg/remotemcp/*` | Git hosts, plugin webhooks and remote MCP servers that the operator or a user supplies | Depends on the value | VCS needs `MULTICA_VCS_SECRET_KEY`, which also turns on Qoder (S6). Endpoints are checked by `ValidatePublicHTTPSEndpoint` | Policy: US-hosted endpoints only, enforced by the egress allowlist |

## 5. Daemon, CLI and agent runtimes (on the user's machine)

| # | path:line | Calls | Class | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| D1 | `server/internal/cli/update.go:34,55-61,283,372`; `server/internal/daemon/auto_update.go:18,69,117-129` | Checks for releases at `api.github.com/repos/multica-ai/multica/releases/*` and downloads GitHub release assets. Runs on `multica update`, and in the daemon every 6 hours when its server is `api.multica.ai` | Unknown (edge), upstream repo | `MULTICA_RELEASE_API_BASE_URL`, `MULTICA_RELEASE_DOWNLOAD_BASE_URL` (`update.go:39-40`), `MULTICA_DAEMON_AUTO_UPDATE` (`daemon/config.go:607`) | Set `MULTICA_DAEMON_AUTO_UPDATE=false`. Point the release variables at the fork's releases once they exist (F3) |
| D2 | `server/internal/cli/update.go:352` | Runs `brew upgrade multica-ai/tap/multica` | Unknown (edge) | No | Install the CLI from the fork's release |
| D3 | `server/cmd/multica/cmd_agent.go:292-293,356-361`; `server/cmd/multica/cmd_setup.go:148-149` | `multica setup` in cloud mode, and `multica login --token` with no server configured, both use `https://api.multica.ai` and `https://multica.ai` | **Non-US (operator)**; server `us-west-2` | `--server-url` / `MULTICA_SERVER_URL` (`cmd_agent.go:318`), `MULTICA_APP_URL`, `multica setup self-host` | Onboarding for the internal instance uses only `multica setup self-host` against our host. F2 changes the compiled default |
| D4 | `server/internal/daemon/repocache/cache.go:612` | `git clone --bare` of task repositories | Depends on the remote | Per-repository config | Allowed for LVCOI repos on GitHub |
| D5 | `server/internal/daemon/config.go:270`; `server/internal/daemon/agents_probe.go:154-155`; `server/internal/daemon/client.go:1138-1142`; `server/internal/daemon/daemon.go:8772-8781` | An agent CLI calls its vendor's model API with issue text, comments and repository code | See the vendor table below | See "How a CLI is chosen" below | See "Proposed replacement" below |
| D6 | `packages/views/onboarding/templates/install-runtime-issue.ts:94,96` | The shipped zh onboarding template tells users to run `curl -fsSL https://code.kimi.com/kimi-code/install.sh \| bash` (and `irm ... \| iex` on Windows) | **Non-US** | No | Do not follow it. F5 removes it from the internal fork build |

### D5 in detail: how a CLI is chosen

The daemon can launch a CLI in four ways. Controlling the daemon host covers only the first two.

1. **Built-in CLI on `PATH`.** The daemon checks the names in `scripts/agent-cli-command-names.txt` (`server/internal/daemon/config.go:270`).
2. **App-bundle discovery.** For example, `dsh` is found inside the DeepSeek Harness Desktop bundle even when it is not on `PATH` (`server/internal/daemon/agents_probe.go:154-155`).
3. **Custom runtime profiles.** These are synced from the server and name a `command_name` for the daemon to launch (`server/internal/daemon/client.go:1138-1142`). Only workspace admins can create or edit them (`server/cmd/server/router.go:1823-1826`). A profile can launch a binary that is not in the built-in list.
4. **Per-agent `custom_env`.** These variables reach the CLI's process (`server/internal/daemon/daemon.go:8772-8781`); the code comment there names `ANTHROPIC_BASE_URL` and `CLAUDE_CODE_USE_BEDROCK`. `isBlockedEnvKey` blocks `MULTICA_*` and system paths, but not provider-routing variables (`daemon.go:10862-10870`). Workspace owners and admins can set them from the UI, and so can the agent's owner (`server/internal/handler/agent_env.go:121-127`).

### D5 in detail: vendor classification from the repo's records

| CLI | Record | Class |
|---|---|---|
| `codebuddy` | Tencent CodeBuddy (`CLI_AND_DAEMON.md:276`) | **Non-US** |
| `codearts` | Huawei Cloud CodeArts (`CLI_AND_DAEMON.md:277`) | **Non-US** |
| `deveco` | Huawei DevEco Code (`CLI_AND_DAEMON.md:278`, `server/pkg/agent/deveco.go:4`) | **Non-US** |
| `zcode` | Zhipu Z.ai ZCode (`CLI_AND_DAEMON.md:282`, `packages/views/runtimes/utils.ts:207`) | **Non-US** |
| `kimi` | Moonshot (`CLI_AND_DAEMON.md:288`); its installer is on `code.kimi.com` (D6) | **Non-US** |
| `reasonix` | "DeepSeek-focused" agent (`CLI_AND_DAEMON.md:289`) | **Non-US** (model provider) |
| `dim` | DimCode, `dimagent.cn` (`apps/docs/content/docs/install-agent-runtime.mdx:50`) | **Non-US** |
| `qodercli` | Qoder, the same service as S6 (`CLI_AND_DAEMON.md:293`) | **Non-US** |
| `qoderclicn` | Qoder CN, documented on `help.aliyun.com` (`CLI_AND_DAEMON.md:294`) | **Non-US** |
| `traecli` | ByteDance TRAE, `docs.trae.cn` (`CLI_AND_DAEMON.md:295`, `server/internal/daemon/agents_probe.go:297`) | **Non-US** |
| `qwen` | Alibaba Qwen Code (`CLI_AND_DAEMON.md:297`) | **Non-US** |
| `mcode` | MiniMax Code (`CLI_AND_DAEMON.md:299`) | **Non-US** |
| `dsh` | DeepSeek Harness (`CLI_AND_DAEMON.md:300`) | **Non-US** |
| `claude` | Anthropic. The region depends on the configured provider (below) | US when routed as below; otherwise Unknown |
| `agy`, `codex`, `copilot`, `cursor-agent`, `devin`, `grok`, `hermes`, `kiro-cli`, `muse`, `omp`, `openclaw`, `opencode`, `pi`, `qwenpaw`, `zeroclaw` | The repo records no operator location, or the model comes from the CLI's own configuration | Unknown |

### D5 in detail: proposed replacement

1. **Install only `claude` on daemon hosts.** Route it to a US region through Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry, not through `api.anthropic.com`.
   - Each of these platforms serves Claude from regions you choose. Model traffic and authentication then go to the provider instead of `api.anthropic.com` ([third-party integrations](https://code.claude.com/docs/en/third-party-integrations), "Regions" row; [network configuration](https://code.claude.com/docs/en/network-config), third-party providers).
   - On a third-party provider, Claude Code sends no operational telemetry to the Datadog intake hosts.
   - WebFetch still calls `api.anthropic.com` for its domain safety check unless `skipWebFetchPreflight: true` is set ([network configuration](https://code.claude.com/docs/en/network-config)). Set it.
2. **Pin the provider with the managed setting `allowedProviders: ["bedrock"]`** ([settings reference](https://code.claude.com/docs/en/settings-reference#allowedproviders)).
   - This holds against an `ANTHROPIC_BASE_URL` set through `custom_env`. That counts as `"customEndpoint"`, which Claude Code admits only for a value pinned in a managed `env` block. It refuses an unlisted provider at startup (same page).
   - It does **not** hold against a region change. `"bedrock"` covers Bedrock's own service "including its regional ... endpoints", so a non-US `AWS_REGION` set through `custom_env` is still admitted.
   - The documentation I read does not say whether a managed `env` value beats a value already in the launch environment, so this audit does not rely on it.
3. **Restrict egress on daemon hosts** to three destinations, by hostname:
   - our server
   - `github.com`
   - the chosen US-region model endpoint, for example the regional Bedrock runtime host

   This is the control that holds against `custom_env`, runtime profiles and any CLI installed by mistake.
4. **Do not install any CLI that the table above classes as Non-US or Unknown.** F6 replaces this rule with a daemon allowlist that also covers runtime profiles and provider-routing `custom_env` keys.

The choice of provider and account is a cost decision for Lawrence (section 10).

## 6. Client apps (desktop, mobile, web)

| # | path:line | Calls | Class | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| C1 | `apps/desktop/src/shared/runtime-config.ts:16-21` | Desktop default API, WebSocket and app URLs on `api.multica.ai` and `multica.ai` | **Non-US (operator)**; server `us-west-2` | In packaged builds, only through `~/.multica/desktop.json` (`apps/desktop/src/main/runtime-config-loader.ts:21-44`). Dev builds use `VITE_API_URL`, `VITE_WS_URL`, `VITE_APP_URL` | Ship a `desktop.json` pointing at our host with each install. F1 makes the default a build-time variable |
| C2 | `apps/desktop/electron-builder.yml:130-139`; `apps/desktop/src/main/updater.ts:18-53` | electron-updater polls GitHub Releases for `multica-ai/multica` 5 seconds after start and then every hour, and downloads automatically. **Runs by default** | Unknown (edge), upstream repo | Only through the user's in-app auto-update setting. The feed is fixed in the yml | Turn auto-update off on installed clients. F3 moves the feed to `lvcoi/multica` |
| C3 | `apps/desktop/src/main/cli-bootstrap.ts:17-18,48,103` | Downloads the `multica` CLI from upstream GitHub releases when the bundled CLI is missing | Unknown (edge), upstream repo | No | F3 |
| C4 | `packages/views/runtimes/components/update-section.tsx:15-16` | Browser fetch of `api.github.com/repos/multica-ai/multica/releases/latest` when the runtime CLI section opens. **Runs by default** and sends no user data | Unknown (edge) | No | F3. Until then, the GitHub host stays on the client allowlist |
| C5 | `apps/mobile/.env.production:4-5` (committed) | The mobile production build points at `api.multica.ai` and `multica.ai` | **Non-US (operator)**; server `us-west-2` | `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_WEB_URL` at build time | Build with our values, or do not ship mobile for the internal instance |
| C6 | `apps/desktop/.env.staging:4,11`; `apps/mobile/.env.staging:5,9`; commented example at `.env.example:707`; doc comment at `server/internal/storage/s3.go:261` | Upstream staging hosts on `copilothub.ai` | Unknown | Same env vars as C1 and C5 | Never build with `.env.staging`. F4 removes the files |
| C7 | `packages/core/analytics/index.ts:113-114,147` | posthog-js events and exception autocapture | US (`us-east-1`) | Starts only if `/api/config` returns a PostHog key (S12) | Covered by S12 |
| C8 | `apps/web/lib/google-analytics.ts:70`; `apps/web/app/layout.tsx:140,178` | Google Analytics | Unknown (edge) | `GOOGLE_ANALYTICS_ID` (`.env.example:703`) | Leave unset |
| C9 | `apps/web/features/landing/utils/use-github-stars.ts:31`; `apps/web/features/landing/utils/github-release.ts:39-40,70` | Landing-page star count and the `/download` release list, from `api.github.com` | Unknown (edge) | No | Not reached in signed-in use. Acceptable internally |
| C10 | `apps/docs/components/video-embed.tsx:35,44`, used at `apps/docs/content/docs/index.zh.mdx:32` | Bilibili iframe on the zh docs index page | **Non-US** | Not configurable | Configuration cannot close this. Do not deploy `apps/docs` for the internal instance (leave `DOCS_URL` empty) |
| C11 | `apps/web/app/layout.tsx:160-166`; `apps/desktop/src/renderer/src/main.tsx:39-41` | The `unpkg.com/react-grab` script | Unknown (edge) | Development builds only, and only with `VITE_REACT_GRAB` | None. Never in production builds |
| C12 | `apps/web/next.config.ts:51-96`; `apps/web/config/runtime-urls.ts:57-68` | Web rewrites to the API and docs upstreams | Our hosts | `REMOTE_API_URL`, `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`, `DOCS_URL` | Set them to our hosts |

Plain hyperlinks to `multica.ai/docs`, `/changelog` and `/download` (for example `packages/views/layout/help-links.ts:1-7`) load nothing until a user clicks one. They are attribution and stay unchanged under the licensing guardrail.

## 7. Build and supply chain

These run when we build or pull images, not while the instance serves users.

| # | path:line | Pulls from or sends to | Class | Configurable | Proposed replacement |
|---|---|---|---|---|---|
| B1 | `docker-compose.selfhost.yml:56,235`; `.env.example:244-246`; `deploy/helm/multica/values.yaml:6,12`; `deploy/helm/multica/templates/backend.yaml:67` | Upstream images `ghcr.io/multica-ai/multica-backend` and `multica-web`, tag `latest` | Unknown (edge), upstream | `MULTICA_BACKEND_IMAGE`, `MULTICA_WEB_IMAGE`, `MULTICA_IMAGE_TAG`. Compose and Helm both build `repository:tag`, so a digest cannot go in the image variable | Build from the fork (`docker-compose.selfhost.build.yml`), push to a US-region registry we own, and set an **immutable tag** such as the release version or git SHA (section 8). Digest pinning needs F7 |
| B2 | `Dockerfile:2,27`; `Dockerfile.web:6,29,59`; `docker-compose.selfhost.yml:37`; `docker-compose.yml:5` | Docker Hub base images: `golang:1.27.3-alpine`, `alpine:3.21`, `node:22.23.4-alpine`, `pgvector/pgvector:pg17` | Unknown | `FROM` lines are hardcoded; no registry `ARG` | Mirror into our registry. F7 adds a registry `ARG` and digest pins |
| B3 | `Dockerfile:4,29`; `Dockerfile.web:66` | Alpine packages | Unknown (Fastly) | No | Build-time only. Accept |
| B4 | `Dockerfile:10`; `go.mod` (no proxy settings) | Go modules via `proxy.golang.org` and `sum.golang.org` | Unknown (Google) | `GOPROXY` on the build host | Accept. Downloads are verified against `go.sum` |
| B5 | `Dockerfile.web:23-26`; `.npmrc` (only `shamefully-hoist=true`) | npm packages and pnpm | Unknown (Cloudflare) | Registry settings on the build host | Accept. The lockfile pins integrity hashes |
| B6 | `apps/web/app/layout.tsx:3,41-72`; `apps/web/app/(landing)/layout.tsx:11-15`; `apps/docs/app/[lang]/layout.tsx:20-39` | `next/font/google` downloads fonts at build time, then serves them from our host | Unknown (Google) | No | Build-time only. Accept |
| B7 | `scripts/install.sh:17-20,152-163`; `scripts/install.ps1:15-16,89,250-267` | Upstream installers clone and download releases from `github.com/multica-ai/multica` and use the upstream brew tap. `install.sh` does not verify a checksum | Unknown (edge), upstream | Only the ref and the install directories | Do not use the upstream one-line installers. F3 |
| B8 | `.github/workflows/release.yml:147-459`; `.goreleaser.yml:74-83` | Images and charts go to `ghcr.io/${{ github.repository_owner }}` and follow the fork. Desktop assets and the brew formula go to the hardcoded `multica-ai/multica` and `multica-ai/homebrew-tap` | n/a | Partly | F3 |
| B9 | `Dockerfile.web:56` (`pnpm --filter @multica/web build`, which runs `next build`, `apps/web/package.json:8`) | Next.js anonymous build telemetry. `NEXT_TELEMETRY_DISABLED` is set nowhere in the repo, and the repo does not record the endpoint | Unknown | Only through environment inside the build stage | F8 sets `NEXT_TELEMETRY_DISABLED=1` in `Dockerfile.web`. Until then, set it on any host that runs `next build` outside Docker. Build-time only; no user data |

No Chinese package mirror or registry is configured anywhere in the tree.

## 8. Recommended baseline for the internal instance

This baseline is configuration only. It closes every Non-US row except three, which need the egress allowlist below:

- the ungated Qoder catalog endpoint (S6)
- the Bilibili docs embed (C10)
- the agent bypasses through `custom_env` and runtime profiles (D5)

```dotenv
# Server
DO_NOT_TRACK=1                 # S1
FRONTEND_ORIGIN=https://<our-host>   # S2
MULTICA_CLOUD_URL=             # S18
MULTICA_LARK_SECRET_KEY=       # S3
MULTICA_DINGTALK_SECRET_KEY=   # S4
MULTICA_WECOM_SECRET_KEY=      # S5
MULTICA_VCS_SECRET_KEY=        # S6: unset unless GitLab/Forgejo is needed; setting it also enables Qoder
MULTICA_TELEGRAM_SECRET_KEY=   # S7
COMPOSIO_API_KEY=              # S17
RESEND_API_KEY=                # S13: use SMTP_* to a US relay instead
MULTICA_LLM_API_KEY=           # S11: off until a US gateway exists
MULTICA_LLM_BASE_URL=
GOOGLE_ANALYTICS_ID=           # C8
# S12: compose forwards neither POSTHOG_API_KEY nor ANALYTICS_DISABLED, so analytics
# is already off. Outside compose, also set ANALYTICS_DISABLED=true.

# Images (B1): repository only, plus an immutable tag. Compose appends
# ":${MULTICA_IMAGE_TAG}", so do not put "@sha256:" in the image variables.
MULTICA_BACKEND_IMAGE=<our-registry>/<backend-repo>
MULTICA_WEB_IMAGE=<our-registry>/<web-repo>
MULTICA_IMAGE_TAG=<release-version-or-git-sha>

# Daemon hosts
MULTICA_SERVER_URL=https://<our-host>   # D3
MULTICA_DAEMON_AUTO_UPDATE=false        # D1
```

Outside the env file:

- Install only `claude` on daemon hosts, routed as described in D5.
- Ship each desktop install with a `~/.multica/desktop.json` that points at our host, and turn auto-update off (C1, C2).
- Do not deploy `apps/docs` (C10).

### Egress allowlist (default deny)

| Hosts | Allowed destinations |
|---|---|
| Server | Our database and object store, our SMTP relay, `api.github.com` and `github.com` (repositories and the GitHub App), and the US model gateway if S11 is turned on |
| Daemon | Our server, `github.com` and `api.github.com`, and the single US-region model endpoint chosen in D5 |
| Desktop clients | Our server, plus GitHub until F3 lands (C2-C4) |

Because the default is deny, these hosts stay blocked without being named:

- `*.multica.ai` (S1, S18, D3)
- `api.qoder.com` and `api.qoder.com.cn` (S6)
- every China-vendor model endpoint (D5)
- `code.kimi.com` (D6)
- `clawhub.ai` (S9)

The allowlist does **not** stop S10, because skills.sh imports are served from GitHub. That case is handled by policy and F5. The hosting ADR (LVCO-5) owns the final list.

## 9. Follow-ups for Fork Engineer (code, not done here)

None of these change branding or attribution.

| ID | Change | Why |
|---|---|---|
| F1 | Make the desktop default API, WebSocket and app URLs build-time variables (`apps/desktop/src/shared/runtime-config.ts:16-21`) | C1 |
| F2 | Make the CLI's `defaultCloudServerURL` and `defaultCloudAppURL` (`server/cmd/multica/cmd_agent.go:292-293`) build-time `-ldflags` values | D3 |
| F3 | Point the release channels at `lvcoi/multica`:<br>- `apps/desktop/electron-builder.yml:132-133`<br>- `apps/desktop/src/main/cli-bootstrap.ts:17-18`<br>- `packages/views/runtimes/components/update-section.tsx:15-16`<br>- the release-repo path in `server/internal/cli/update.go`<br>- `scripts/install.{sh,ps1}`<br>- `.goreleaser.yml:79`<br>- the compose, Helm and `.env.example` image defaults<br><br>Add a checksum check to `install.sh` | C2-C4, D1, B1, B7, B8 |
| F4 | Remove `apps/desktop/.env.staging` and `apps/mobile/.env.staging` | C6 |
| F5 | Add a switch that removes the Non-US surfaces from the internal fork build and hides their UI:<br>- Lark/Feishu, DingTalk, WeCom and Telegram (S3-S5, S7)<br>- all Qoder routes, including the ungated `POST /qoder/environments` (`qoder_catalog.go:143-197`), and the Qoder runtime bridge (S6)<br>- the 13 Non-US runtimes in section 5 and the zh `kimi` onboarding template (D5, D6)<br><br>Optionally, add an allowlist of skill-import sources (S10) | Turns "do not configure" into "cannot configure" |
| F6 | Add a daemon runtime allowlist, for example `MULTICA_ALLOWED_AGENTS=claude`. It must cover built-in CLIs, app-bundle discovery and custom runtime profiles (`client.go:1138-1142`).<br><br>Add an operator-set list of `custom_env` keys that the daemon refuses, covering provider-routing variables such as `ANTHROPIC_BASE_URL`, `ANTHROPIC_BEDROCK_BASE_URL`, `AWS_REGION` and `CLAUDE_CODE_USE_*` (`daemon.go:10862`) | D5. Today the UI can bypass host-level control |
| F7 | Add a registry `ARG` and digest pins to the `FROM` lines in `Dockerfile` and `Dockerfile.web`. Support digest references in compose and Helm, for example with a full-reference image variable that has no `:tag` suffix | B1, B2 |
| F8 | Set `ENV NEXT_TELEMETRY_DISABLED=1` in the build stage of `Dockerfile.web` | B9 |

## 10. Decisions for Lawrence

Each of these spends money, picks a vendor or affects visibility, so they are listed as options. None has been acted on.

1. **Model provider for the agents (D5) and the server LLM (S11):** Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry, in a US region. All three bill pay-as-you-go through the cloud account ([third-party integrations](https://code.claude.com/docs/en/third-party-integrations), "Billing" row). This audit did not gather pricing; ADR-001 (LVCO-5) should carry the comparison.
2. **Object storage (S15):** local disk (no added cost) or an S3 bucket in a US region.
3. **SMTP relay (S13):** which US provider to use, if email login is needed.
4. **Container registry (B1, B2):** GHCR under `lvcoi`, or a registry in the cloud account.
5. **Visibility:** `lvcoi/multica` is public, so this audit is public. Decide whether these docs belong in a private repo instead.

## Not covered

- **Where each Unknown-class agent CLI vendor processes data (D5).** Answering this needs each vendor's own documentation.
- **Hosts built at runtime from user content,** such as markdown images and attachment URLs. The egress allowlist is the control for these.
- **Mobile push and over-the-air updates.** None are configured: `apps/mobile` has no `expo-updates`, no `expo-notifications` and no `eas.json`.

This is a docs-only change. No code tests were run.
