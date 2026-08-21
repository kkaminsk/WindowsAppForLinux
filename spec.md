# WindowsAppForLinux — Windows 365 / Azure Virtual Desktop Client for Linux

## Specification

---

## 1. Introduction and scope

WindowsAppForLinux is a Linux desktop client for **Windows 365 Cloud PCs** and **Azure Virtual Desktop (AVD)
desktops and RemoteApps** — functionally analogous to Microsoft's "Windows App" on other platforms, which
Microsoft does not ship for Linux.

The application provides:

- Entra ID sign-in with support for **multiple simultaneously signed-in accounts** and fast account switching.
- A main resource area that **enumerates the signed-in user's resources** from both providers: Windows 365
  Cloud PCs (via Microsoft Graph) and AVD desktops/RemoteApps (via the AVD workspace feed).
- A **per-resource choice of connection method**: native (FreeRDP) or web-based (browser + Microsoft web client).
- **Cloud PC management controls** surfaced through Microsoft Graph: restart, rename, troubleshoot, and
  reset/reprovision.
- Robust handling of **token expiration**: silent refresh, re-authentication prompts, and per-account token
  caches.

In scope: an end-user client. Out of scope: admin-console functionality (provisioning policies, tenant
management), although actions that require admin capability are noted where they intersect the end-user UI.

This specification is **implementation-stack-agnostic**: no UI toolkit or language is committed. Authentication
is specified at the protocol level (OAuth 2.0), with candidate libraries noted.

Functional requirements are numbered **FR-1 … FR-5** (section 3). The delivery roadmap (section 11) maps each
requirement onto three phases.

---

## 2. Background and constraints

### 2.1 Why this is not `xfreerdp /v:host`

Windows 365 and AVD do **not** expose a traditional inbound RDP listener on `IP:3389`. Connectivity uses
**Reverse Connect**: the client authenticates with Entra ID, obtains a digitally signed connection
configuration, and connects outbound to an AVD gateway over TCP 443; the broker arranges for the Cloud
PC/session host to connect back to the same gateway, and only then does the actual RDP handshake take place
inside a nested TLS transport. ([Microsoft Learn][1])

```text
Linux
  │
  │ Entra ID authentication / resource discovery
  ▼
Windows 365 / AVD control plane
  │
  │ signed connection configuration
  ▼
AVD Gateway / Broker
  │
  ├── TCP 443  Reverse Connect   ← always establishes first
  │
  ├── UDP 3478 STUN             ← attempts direct Shortpath
  │
  └── UDP 3478 TURN             ← relayed Shortpath fallback
  │
  ▼
Windows 365 Cloud PC / AVD session host
       RDP session
```

**RDP Shortpath** does not remove the bootstrap requirement: the session first establishes the TCP/443 Reverse
Connect transport, then exchanges STUN candidates and may move the RDP channels onto direct UDP, falling back to
TURN relay if direct connectivity fails. For ANC/private-network Cloud PCs there is also private Shortpath over
UDP/3390, but the public Reverse Connect path is still required initially. The TCP channel remains available
whenever UDP cannot be established. ([Microsoft Learn][1])

Microsoft's architecture description of the client flow: after Entra authentication, the client sends its token
to the AVD feed service, receives **digitally signed connection configurations**, stores them as `.rdp` files,
and uses one of those configurations to establish the gateway connection. The broker then causes the session
host to connect to the same gateway, after which the nested RDP/TLS session starts. ([Microsoft Learn][a2])

### 2.2 What is already solved (FreeRDP)

Modern FreeRDP implements enough of the AVD/Windows 365 ARM-gateway path to authenticate with Entra ID, resolve
the gateway, traverse the WebSocket/TCP 443 gateway, and establish an RDP session. Users demonstrably run this
against Windows 365 Cloud PCs, and upstream has continued fixing gateway compatibility through 2026.
([GitHub][a1]) The upstream FAQ documents the invocation pattern:

```text
<rdpw file> /gateway:type:arm /sec:aad
```

([GitHub][a3])

Build against **FreeRDP 3.30.0 or newer** (released July 16, 2026, including WebSocket fixes), not whatever
older version a distribution ships. ([GitHub][a4])

| Piece                                  | Status                                                  | Consequence                            |
| -------------------------------------- | ------------------------------------------------------- | -------------------------------------- |
| Entra authentication                   | **Implemented in FreeRDP**                              | Don't write this yourself              |
| AVD ARM/gateway resolution             | **Implemented in FreeRDP**                              | Reverse-connect bootstrap is viable    |
| TCP 443 gateway/WebSocket              | **Implemented**                                         | Native Windows 365 sessions work today |
| RDP/RDSTLS                             | **Implemented + protocol documented**                   | Normal FreeRDP territory               |
| Cloud PC enumeration                   | **Public Graph API**                                    | Easy application UI/discovery          |
| Direct browser launch                  | **Public/supported API surface**                        | Excellent fallback                     |
| Obtain signed `.rdpw` programmatically | **Weak spot**                                           | Main native-client integration problem |
| Shortpath/UDP                          | **Protocol documented, FreeRDP missing implementation** | Native client stays on TCP gateway     |

The two remaining gaps — programmatic acquisition of the signed `.rdpw` connection configuration (section 8) and
client-side UDP Shortpath in FreeRDP (section 11, Phase 2) — shape the roadmap. Do **not** spend engineering
effort reimplementing the gateway, broker, RDSTLS, or reverse-connect RDP stack; FreeRDP has crossed that
bridge. The code to own is the **session-acquisition/control-plane layer immediately above FreeRDP** plus the
application described in this document.

---

## 3. Functional requirements

### FR-1 — Sign-in and resource enumeration

The main area of the application lets the user sign into an Entra ID account and presents a unified list of that
user's remote resources from **both** providers:

- **Windows 365 Cloud PCs**, enumerated via Microsoft Graph `GET /me/cloudPCs` (v1.0, delegated
  `CloudPC.Read.All`). Each entry carries display name, Cloud PC `id`, status, and provisioning metadata.
- **AVD desktops and RemoteApps**, enumerated via the AVD workspace feed (feed discovery → workspace feed
  download). Graph **cannot** enumerate AVD resources for an end user; AVD objects live under ARM
  (`Microsoft.DesktopVirtualization`) and require Azure RBAC that ordinary end users lack. See section 8.

The list refreshes on sign-in, on account switch, on manual refresh, and periodically while the app is
foregrounded (section 7.3).

### FR-2 — Per-resource connection method (Native vs Web)

Every resource offers two launch methods:

- **Connect (Native)** — libfreerdp session over the ARM gateway path (section 5.2).
- **Connect (Web)** — system browser launched at the Microsoft web client via a direct-launch URL
  (section 5.3).

The user's last-used method is remembered **per resource** and becomes that resource's default. A method that is
currently unavailable is shown disabled with a reason (e.g., "Native mode unavailable: signed connection
configuration could not be acquired" during Phase 0, or "Web launch unavailable: workspace/resource ID unknown"
for AVD entries without admin-provided IDs).

### FR-3 — Multiple accounts and account switching

- The user can sign in additional accounts at any time ("Add account") without signing out existing ones.
- Exactly one account is **active** at a time. The resource list, all Graph calls, all feed queries, all
  management actions, and all connection launches are scoped to the active account.
- Switching accounts swaps the resource list and action context; it does not disturb the other accounts' cached
  tokens or live sessions.
- Each account can be signed out individually; signing out removes that account's tokens from the cache and its
  resources from the UI.
- Account state is isolated per account: token cache entries, per-resource connection preferences, and
  auth-state (section 6.4) are all keyed by account.
- Web launches carry the active account's UPN as `#loginHint=` so the browser-side web client lands on the same
  account (section 5.3).

### FR-4 — Token expiration handling

The application **expects** token expiration as a normal condition, not an error:

- Access tokens (lifetime ~60–90 minutes) are acquired **silently** from the per-account cache/refresh-token
  flow immediately before every Graph call, feed query, and connection launch. The app never schedules work off
  wall-clock lifetime assumptions.
- When silent acquisition fails (refresh token expired after 90 days of inactivity, revocation, password
  change/`invalid_grant`), the account enters a **ReauthRequired** state: a non-blocking banner on that account
  prompts interactive re-authentication (with `login_hint` prefilled). Other accounts are unaffected.
- Conditional Access / Continuous Access Evaluation claims challenges (`interaction_required`, `claims`) trigger
  interactive re-auth carrying the returned `claims` parameter.
- The full state machine is specified in section 6.4.

### FR-5 — Cloud PC management actions (Graph)

For each Cloud PC, the app exposes management controls via Microsoft Graph:

- **Restart** (`reboot`), **Rename**, **Troubleshoot** — self-service via `POST /me/cloudPCs/{id}/...`
  (Graph **beta**; see section 7.2).
- **Reset/Reprovision** (`reprovision`) — self-service via beta `/me` path; **destructive** (wipes the Cloud
  PC back to a fresh image), so the UI requires an explicit typed/checked confirmation before invoking it.
- **Restore to snapshot** and **Resize** — admin-context only (`/deviceManagement/virtualEndpoint/...`); shown
  only when the active account demonstrably has admin capability, otherwise hidden.

All actions require delegated `CloudPC.ReadWrite.All` (admin consent; section 6.2). Action invocations show
progress and surface the resulting status transition in the resource list.

---

## 4. User interface specification (toolkit-agnostic)

### 4.1 Main window

```text
┌────────────────────────────────────────────────────────────┐
│  [● user@contoso.com ▾]                        [⟳ Refresh] │
│    ├─ user@contoso.com        (active)                     │
│    ├─ admin@fabrikam.com      ⚠ reauthentication required  │
│    ├─ ──────────────                                       │
│    ├─ Add account…                                         │
│    └─ Sign out user@contoso.com                            │
├────────────────────────────────────────────────────────────┤
│  Windows 365                                               │
│   ┌──────────────────────────────────────────────────────┐ │
│   │ 🖥  Cloud PC — Standard 4vCPU   ● Running             │ │
│   │     [Connect ▾]  [⋮ Actions ▾]                       │ │
│   │        ├─ Connect (Native)        ├─ Restart          │ │
│   │        └─ Connect (Web browser)   ├─ Rename           │ │
│   │                                   ├─ Troubleshoot     │ │
│   │                                   └─ Reset (Reprovision)│
│   └──────────────────────────────────────────────────────┘ │
│  Azure Virtual Desktop                                     │
│   Workspace: Contoso Finance                               │
│   ┌──────────────────────────────────────────────────────┐ │
│   │ 🖥  Finance Desktop            [Connect ▾]            │ │
│   │ 📦  Excel (RemoteApp)          [Connect ▾]            │ │
│   └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

- **Account switcher** (top-left): dropdown listing every signed-in account with avatar/UPN, marking the active
  one, plus a per-account auth-state badge ("reauthentication required" when in ReauthRequired state).
  Menu items: switch (click an account), "Add account…", "Sign out <account>".
- **Resource list**: grouped by provider ("Windows 365", "Azure Virtual Desktop"), AVD further grouped by
  workspace. Each entry shows name, type (Cloud PC / desktop / RemoteApp), and live status where available
  (Cloud PC `status` from Graph).

### 4.2 Connect controls

Per-resource split button or menu: the primary action is the resource's remembered default method; the dropdown
offers "Connect (Native)" and "Connect (Web browser)". Disabled entries carry tooltips stating the reason
(FR-2). Launch feedback: spinner on the entry until the native session window appears or the browser is
spawned; failures surface per section 9.

### 4.3 Cloud PC action menu

Actions menu per Cloud PC entry: Restart, Rename, Troubleshoot, Reset (Reprovision). Reprovision opens a
destructive-action dialog requiring explicit confirmation and explaining that the Cloud PC is wiped and rebuilt.
Restore and Resize appear only for admin-capable accounts. Each invoked action shows progress and a
success/failure toast; the entry's status chip updates on the next refresh.

### 4.4 UI states

- **Loading** — skeleton list while enumeration is in flight.
- **Empty** — signed in but no resources ("No Cloud PCs or AVD resources are assigned to this account"), with a
  distinct message for missing license (Graph 404; section 9).
- **Offline** — network unreachable: cached resource list shown greyed with an offline banner; connect actions
  disabled.
- **Error** — enumeration failed: inline error with retry.
- **ReauthRequired banner** — per-account, non-blocking: "Your sign-in for <UPN> has expired. Sign in again."
  with an inline button that starts interactive auth for that account only.
- **Consent-required** — first-run in an unconsented tenant: guided admin-consent screen (section 9).

---

## 5. Architecture

### 5.1 Component model

```text
                        ┌──────────────────────────────┐
                        │            UI shell           │
                        │  account switcher · resource  │
                        │  list · action menus · states │
                        └──────┬───────────────┬───────┘
                               │               │
              ┌────────────────┴──┐     ┌──────┴──────────────┐
              │  Auth / Account   │     │  Resource layer      │
              │  Manager          │     │                      │
              │  · OAuth2+PKCE    │     │  ├ Graph CloudPC     │
              │  · per-account    │◄────┤  │ provider (v1.0)   │
              │    token cache    │token│  ├ AVD Feed provider │
              │  · state machine  │     │  └ Management Action │
              │  (section 6)      │     │    service (Graph)   │
              └────────┬──────────┘     └──────┬───────────────┘
                       │                       │
                       │              ┌────────┴─────────┐
                       │              │  Connection      │
                       │              │  launchers       │
                       │              │  ├ Native:       │
                       │              │  │  libfreerdp   │
                       │              │  │  wrapper      │
                       │              │  └ Web: browser +│
                       │              │     direct-launch│
                       │              │     URL builder  │
                       │              └────────┬─────────┘
                       │                       │
            OS keyring (libsecret /            │
            KWallet) for token storage         ▼
                                     Windows 365 / AVD
```

- **Auth/Account Manager** — owns interactive and silent token acquisition, the per-account encrypted token
  cache, and the per-account auth state machine (section 6).
- **Graph CloudPC provider** — `GET /me/cloudPCs` enumeration and status refresh.
- **AVD Feed provider** — feed discovery + workspace feed download; source of both AVD enumeration (FR-1) and
  signed `.rdpw` connection configurations (section 8). In Phase 0 this provider is replaced by an
  admin-provisioned static resource list (section 11).
- **Management Action service** — Graph Cloud PC actions (FR-5).
- **Native launcher** — thin wrapper around libfreerdp/FreeRDP 3.30.0+ invoking the AVD path
  (`<rdpw> /gateway:type:arm /sec:aad`); manages session windows and surfaces session errors.
- **Web launcher** — builds direct-launch URLs (section 5.3) and opens the system browser.

### 5.2 Native connection path

```text
Linux client application
       │
       ├──────── Microsoft Graph ── enumerate Cloud PCs, CloudPc.Id, actions
       │
       ├──────── AVD resource/feed layer
       │              └── signed connection configuration (.rdpw/.rdp)
       │                      ▲
       │                 MAIN GAP (section 8)
       │
       └──────── FreeRDP 3.x (3.30.0+)
                      ├── Entra/AAD authentication
                      ├── AVD ARM gateway resolution
                      ├── gateway/broker connection
                      ├── WebSocket / TCP 443
                      ├── Reverse Connect
                      ├── RDSTLS / RDP
                      └── UDP Shortpath   ✗ not implemented in FreeRDP today
```

Everything below the `.rdpw` in this stack (AAD auth, ARM gateway, TCP/443 reverse connect, RDP/RDSTLS) is
already handled by FreeRDP. The application's native path therefore consists of: acquire token (section 6) →
obtain signed `.rdpw` for the chosen resource (section 8) → hand it to libfreerdp with the ARM-gateway/AAD
options → manage the session lifecycle.

Until Phase 2 (section 11), native sessions run TCP-only:

```text
Windows 365 + FreeRDP today
        └── Reverse Connect TCP/443   ✓
Windows 365 + FreeRDP Shortpath
        ├── STUN/direct UDP           ✗
        └── TURN/relayed UDP          ✗
```

**Alternative RDP engines (watch list).** **IronRDP** (Devolutions) is a maintained pure-Rust RDP
implementation with .NET/WASM bindings — full RDP core, CredSSP/NLA, TLS 1.3, and the RDCleanPath WSS-bridging
extension; it is used by Devolutions Gateway, Cloudflare Access, and Teleport. It has **no AVD ARM-gateway
support** today, so it is not a current option for this project, but it is the most credible future alternative
engine and worth monitoring. ([GitHub][n1]) FreeRDP remains the only open-source stack with working
AVD/Windows 365 gateway support. Microsoft's own legacy Remote Desktop clients are end-of-support
(September 2025; extended to September 2026 for Gov clouds) with documentation moved to "previous-versions" —
everything consolidates on Windows App, which has no Linux version and no public SDK. ([Microsoft Learn][n2])

The properties appearing in AVD `.rdp`/`.rdpw` files are documented in Microsoft's **Supported RDP
properties** reference — useful when validating or synthesizing connection files handed to FreeRDP.
([Microsoft Learn][n3])

### 5.3 Web connection path

Direct-launch URLs open the Microsoft web client straight into a session, skipping portal navigation.
([Microsoft Learn][a6])

- **Windows 365** (Enterprise and Flex Dedicated):

  ```text
  https://windows.cloud.microsoft/webclient/ent/<CloudPc.Id>
  https://windows.cloud.microsoft/webclient/ent/<CloudPc.Id>?tenant=<tenantID>#loginHint=<UPN>
  ```

  `<CloudPc.Id>` is exactly the `id` Graph returns for the Cloud PC — the same identifier used for FR-5
  actions, so the web path needs no extra discovery.

  Additionally, Graph beta exposes `GET /me/cloudPCs/{id}/retrieveCloudPcLaunchDetail` (delegated
  `CloudPC.Read.All`), which returns a per-Cloud-PC `cloudPcLaunchUrl` of the form
  `https://rdweb-r0.wvdselfhost.microsoft.com/api/arm/weblaunch/tenants/<tenantId>/resources/<resourceId>`.
  ([Microsoft Learn][r7]) The web launcher should prefer this Microsoft-issued URL when available (it is the
  supported per-resource connect URL, and it carries the rdweb-namespace tenant/resource identifiers), falling
  back to constructed `windows.cloud.microsoft/webclient/ent/` URLs. Note: its predecessor
  `getCloudPcLaunchInfo` is deprecated and stops returning data **October 30, 2026** — use only
  `retrieveCloudPcLaunchDetail`.

- **AVD**:

  ```text
  https://windows.cloud.microsoft/webclient/avd/<workspaceID>/<resourceID>
  ```

  with the same optional `?tenant=` and `#loginHint=` parts. Caveats: the required workspace/resource
  ObjectIds are ARM-side identifiers (`Get-AzWvdWorkspace` etc.) that end users cannot self-discover — in
  Phase 0 they must be admin-provisioned (section 11) — and launching a second RemoteApp tab from the same host
  pool disconnects the first.

URL composition rules: query (`?tenant=`) before fragment; `#loginHint=` must be the **last** component. The
web launcher always appends `#loginHint=<active account UPN>` so the browser session follows the active account
(FR-3), and `?tenant=<tenantID>` for accounts whose home tenant differs from the browser's default.

### 5.4 Connectivity reference

Transport sequence (Microsoft-documented): TCP/443 Reverse Connect establishes first; RDP is established; the
client then tries direct UDP Shortpath via STUN, otherwise relayed UDP via TURN; TCP remains the fallback.
([Microsoft Learn][a8]) The UDP transport itself is publicly specified as the Remote Desktop UDP Transport
Extension, including **[MS-RDPEUDP2]**, with multitransport bootstrap in **[MS-RDPEMT]**. ([Microsoft
Learn][a7]) These specs are the basis for the Phase 2 FreeRDP work.

---

## 6. Identity and session lifecycle

### 6.1 Protocol

- **OAuth 2.0 authorization code flow with PKCE**, Microsoft identity platform v2 endpoints, as a **public
  client** (no client secret).
- Interactive sign-in uses the system browser (preferred) or an embedded webview, with a **loopback redirect
  URI registered as a native/public-client (mobile & desktop) redirect** — explicitly **not** an `spa`
  redirect: SPA-registered redirects cap refresh tokens at 24 hours, which would force daily re-auth.
  ([Microsoft Learn][r3])
- Candidate libraries (non-binding): MSAL (Python/.NET/Node/Java flavors run on Linux) or a minimal in-house
  OAuth 2.0 client following the MSAL cache and silent-acquisition patterns. ([Microsoft Learn][r4])

### 6.2 Scopes and consent

| Scope                   | Used for                              | Notes                                        |
| ----------------------- | ------------------------------------- | -------------------------------------------- |
| `openid profile offline_access` | Sign-in, ID token, refresh tokens | Standard                                     |
| `CloudPC.Read.All`      | FR-1 Cloud PC enumeration             | Delegated; **admin consent required**        |
| `CloudPC.ReadWrite.All` | FR-5 management actions               | Delegated; **admin consent required**; no lesser self-service scope exists |
| AVD feed audience       | FR-1 AVD enumeration, `.rdpw` (§8)    | **Open question** — token audience/scope for the feed service is undocumented for third parties |

Strategy: request `CloudPC.Read.All` at first sign-in; request `CloudPC.ReadWrite.All` via incremental consent
when the user first invokes an action. Both require tenant admin consent, so the app must handle the
unconsented case gracefully (section 9). App-only permissions and personal Microsoft accounts are **not
supported** by the Cloud PC APIs. ([Microsoft Learn][r1])

### 6.3 Token cache

- Per-account cache holding access token, refresh token, ID token, and the account object (home account ID,
  UPN, tenant) — modeled on the MSAL cache schema. ([Microsoft Learn][r4])
- Encrypted at rest via the OS keyring (libsecret/GNOME Keyring or KWallet); if no keyring is available, the
  app falls back to an encrypted file with a documented, weaker guarantee and warns the user.
- Multi-account: the cache stores any number of accounts; the app enumerates cached accounts at startup to
  rebuild the account switcher, and silent acquisition always names the specific account (MSAL
  `GetAccounts` → `AcquireTokenSilent(account)` pattern). ([Microsoft Learn][r5])
- Refresh tokens rotate on every use; the cache always persists the newest RT immediately.

### 6.4 Token expiry state machine (per account)

```text
                 ┌────────────┐
                 │ SignedOut  │
                 └─────┬──────┘
              Add account / re-auth
                       ▼
              ┌─────────────────┐   success   ┌────────────┐
              │ InteractiveAuth ├────────────►│   Active   │◄─────────────┐
              └───────┬─────────┘             └─────┬──────┘              │
                      │ user cancels /              │ access token       │ success
                      │ hard failure                │ expired/near expiry│ (new AT, rotated RT)
                      ▼                             ▼                    │
                 SignedOut                 ┌────────────────┐            │
                                           │  SilentRefresh ├────────────┘
                                           └───┬───────┬────┘
                 invalid_grant / RT expired    │       │  interaction_required /
                 (90-day inactivity,           │       │  claims challenge (CA/CAE)
                 revocation, pwd change)       │       ▼
                                               │   InteractiveAuth
                                               │   (carrying `claims`,
                                               │    login_hint=<UPN>)
                                               ▼
                                       ┌───────────────┐
                                       │ ReauthRequired│  (non-blocking banner;
                                       └───────────────┘   other accounts unaffected)

        Network failure during SilentRefresh → retry with backoff; keep using an
        unexpired cached AT if one exists; else treat account as Offline, not ReauthRequired.
```

Operating rules:

- Silent acquisition runs **before every Graph call, feed query, and connection launch** — never on a timer
  derived from assumed lifetimes. Access tokens last a randomized 60–90 minutes (CAE-capable clients may
  receive 20–28 h tokens); the cache layer, not the caller, knows when refresh is needed. ([Microsoft
  Learn][r2])
- A `claims` value returned in a challenge is passed through to the interactive request verbatim; a plain retry
  is never sufficient for a claims challenge.
- ReauthRequired affects only the one account: its resources grey out and its banner appears, while the active
  or other accounts continue working.

---

## 7. Microsoft Graph integration

### 7.1 Endpoint matrix

| Operation                | End-user path (delegated)                       | API version | Admin path (`/deviceManagement/virtualEndpoint/cloudPCs/{id}`) |
| ------------------------ | ----------------------------------------------- | ----------- | ---------------------------------------------------- |
| List Cloud PCs           | `GET /me/cloudPCs`                              | **v1.0**    | `GET .../cloudPCs` (all tenant Cloud PCs)            |
| Get Cloud PC             | `GET /me/cloudPCs/{id}`                         | v1.0        | `GET .../{id}`                                       |
| Retrieve launch detail   | `GET /me/cloudPCs/{id}/retrieveCloudPcLaunchDetail` | **beta** | `GET .../{id}/retrieveCloudPcLaunchDetail`           |
| Restart                  | `POST /me/cloudPCs/{id}/reboot`                 | **beta**    | `POST .../{id}/reboot` (v1.0)                        |
| Rename                   | `POST /me/cloudPCs/{id}/rename`                 | **beta**    | `POST .../{id}/rename` (v1.0)                        |
| Troubleshoot             | `POST /me/cloudPCs/{id}/troubleshoot`           | **beta**    | `POST .../{id}/troubleshoot` (v1.0)                  |
| Reset / Reprovision      | `POST /me/cloudPCs/{id}/reprovision`            | **beta**    | `POST .../{id}/reprovision` (v1.0)                   |
| Restore to snapshot      | — (none, even in beta)                          | —           | `POST .../{id}/restore` (+ snapshot enumeration)     |
| Resize                   | — (none)                                        | —           | `POST .../{id}/resize`                               |
| End grace period         | — (none)                                        | —           | `POST .../{id}/endGracePeriod`                       |

Permissions: enumeration and launch detail `CloudPC.Read.All`; every action `CloudPC.ReadWrite.All`; both
delegated-only, admin-consent. ([Microsoft Learn][r1])

`retrieveCloudPcLaunchDetail` returns `{cloudPcId, cloudPcLaunchUrl, windows365SwitchCompatible, ...}` and is
the successor to `getCloudPcLaunchInfo`, which is **deprecated and stops returning data October 30, 2026** —
the app must never ship a dependency on the deprecated function. ([Microsoft Learn][r7])

### 7.2 Beta-dependency risk

The `/me` action endpoints exist **only in Graph beta**, which Microsoft marks "subject to change; use in
production applications is not supported." Mitigation: the Management Action service isolates version selection
behind a feature flag; if a beta call fails with a shape/contract error the UI degrades to "Action unavailable —
Microsoft API change" rather than crashing; the project watches for v1.0 promotion of the `/me` action paths
and switches when available.

National-cloud note: Cloud PC Graph APIs are unavailable in US Gov L5 (DOD) and 21Vianet-operated clouds; the
app should detect these environments and disable FR-5 with an explanatory message.

### 7.3 Status refresh

Resource status (e.g., Cloud PC `status`) is polled while the window is foregrounded (suggested: 60 s interval,
paused in background), refreshed immediately after any invoked action, and on manual refresh. All Graph calls
honor `429` + `Retry-After` (section 9).

---

## 8. AVD feed integration — the `.rdpw` gap

This is the project's **critical-path unknown**, and it now carries two responsibilities: AVD **enumeration**
(FR-1) and **signed connection configuration acquisition** for the native path (FR-2).

What Microsoft documents publicly:

- The feed-discovery URL:

  ```text
  https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery
  ```

- That the AVD feed returns the user's remote resources and digitally signed connection configurations, in two
  stages: **initial feed discovery** and **workspace feed download**. ([Microsoft Learn][a5], [a2])

What is **not** documented: a supported third-party client contract — request/response schemas, token
audience/client-ID requirements — that would let an independent client do
`authenticate → enumerate resources → download current signed .rdpw` in a supported way. The service is not a
black box (endpoints and architecture are visible), but the client-facing feed protocol is not exposed as a
supported Graph-style API.

Several facts sharpen the constraint (re-verified August 2026):

- The manual bootstrap path is **effectively gone**: as of July 2026 the AVD web client URL
  (`client.wvd.microsoft.com/arm/webclient/`) redirects to the generic `windows.cloud.microsoft`, which offers
  **no `.rdpw` download option** at all ([GitHub][n4]); FreeRDP's own AVD FAQ has struck out the old
  "Download the rdp file" instruction. ([GitHub][a3])
- Microsoft has confirmed on Q&A (August 2026) that **no public API exists** to fetch `.rdpw` files; manual
  export from the web client (where still available) is the only supported method, and the file contains a
  `loadbalanceinfo` routing token. ([Microsoft Learn][n5])
- Microsoft's blessed automation alternative is the **URI schemes** `ms-avd:connect` and `ms-rd:subscribe` —
  but these are implemented only by the Windows Remote Desktop client and Windows App (≥ 2.0.804.0), so they
  do not help a Linux client. Their parameter shapes do document how Microsoft identifies a resource.
  ([Microsoft Learn][n6])
- A GitHub-wide code search for `api/arm/feeddiscovery` surfaces only documentation and subscription-config
  scripts — **no public client-side implementation of the ARM feed protocol exists**; the reverse-mapping in
  Phase 1 is original work.
- Graph's `CloudPc.Id` is **not** a substitute for the signed `.rdpw`; Graph is control-plane only. However,
  Graph beta `retrieveCloudPcLaunchDetail` (section 7) does return the **rdweb-namespace tenant and resource
  identifiers** for each Cloud PC inside its `cloudPcLaunchUrl` — a concrete, supported anchor for the feed
  reverse-mapping. ([Microsoft Learn][r7])

Reference material for the feed work: the classic RDWeb workspace webfeed XML is publicly specified as
**[MS-TSWP] (Terminal Services Workspace Provisioning Protocol)** — the AVD ARM feed is a different,
undocumented API, but MS-TSWP is its closest documented relative — and the community project **RAWeb**
implements a compatible feed *server* that Windows App can subscribe to, a useful reference for workspace-feed
semantics from the server side. ([Microsoft Learn][n7], [GitHub][n8])

Consequences:

- **Enumeration**: until the feed protocol is reverse-mapped (Phase 1), the app cannot self-enumerate AVD
  resources for an ordinary end user. Phase 0 AVD support is therefore limited to **admin-provisioned
  bookmarks** (workspace/resource IDs supplied by a tenant admin, launched via the web path).
- **Native connections**: Phase 1's first engineering validation stays deliberately small — obtain one valid
  Windows 365 `.rdpw` from a tenant by whatever mechanism is currently available, prove the target stack
  against FreeRDP 3.30.0+ with the upstream AVD flags, and only then invest in reverse-mapping feed
  acquisition. If TCP-only performance is acceptable, most protocol risk is eliminated before touching the
  feed.
- **Risk**: unknown token-audience / first-party-client-ID requirements raise both schedule risk and a
  support/ToS risk (section 12).

---

## 9. Error handling

| Domain | Error | Handling |
| ------ | ----- | -------- |
| Auth | AT expired | Normal: silent refresh (section 6.4), invisible to user |
| Auth | `invalid_grant` / RT expired / revoked | Account → ReauthRequired; per-account banner; other accounts unaffected |
| Auth | `interaction_required` / claims challenge | Interactive re-auth with `claims` + `login_hint`; never plain-retry |
| Auth | User cancels interactive auth | Return to prior state; no error dialog |
| Graph | `403` insufficient privileges / consent missing | Guided admin-consent flow: explain that a tenant admin must approve, show the admin-consent URL for forwarding |
| Graph | `404` on `/me/cloudPCs` (no license/assignment) | Empty state: "No Cloud PC is assigned to this account" |
| Graph | `429` throttled | Honor `Retry-After`; exponential backoff; no user-visible error unless persistent |
| Graph | Beta contract change (unexpected shape) | Disable affected action with "Microsoft API change" message; log for triage |
| Feed | Discovery/download failure | AVD section shows inline error + retry; Cloud PC section unaffected |
| Native session | Gateway rejects connection / token rejected mid-session | Toast with reason; offer retry (fresh silent token) and web-launch fallback |
| Native session | FreeRDP session drop | Standard reconnect prompt; if the signed config has expired, re-acquire before reconnecting |
| Web launch | Browser fails to spawn | Error toast with the URL offered for manual copy |
| Network | Offline | Offline banner; cached list greyed; connects disabled; auto-recover on connectivity |

Messaging rules: user-visible messages state what happened and what the user (or their admin) can do; raw error
codes go to logs, not dialogs; retries are automatic where safe (throttling, transient network) and manual where
the user should decide (session drop, feed failure).

---

## 10. Security considerations

- **Token storage**: tokens only in the OS keyring-backed encrypted cache (section 6.3); never in plain files,
  logs, or environment variables.
- **Public client**: no client secret is embedded; PKCE protects the auth-code exchange; the loopback redirect
  binds the response to the local listener.
- **Signed connection configuration**: the `.rdpw` is Microsoft-signed; the client treats it as opaque and
  passes it to FreeRDP unmodified. Acquired configs are held in memory or short-lived files with `0600`
  permissions and deleted after session start.
- **Browser handoff**: direct-launch URLs place the UPN in the `#loginHint` fragment. Fragments are not sent to
  servers in the HTTP request, but they do enter browser history on a shared desktop — acceptable for a UPN
  (low sensitivity), documented here as a deliberate trade-off.
- **Destructive actions**: reprovision requires explicit confirmation (section 4.3); the app never batches or
  auto-retries destructive Graph actions.
- **Logging**: tokens, `Authorization` headers, and full `.rdpw` contents are redacted from all log levels;
  UPNs are redacted at default log level.

---

## 11. Roadmap

UDP Shortpath is a distinct **Phase 2**, not part of the MVP.

**Phase 0 — Web-first client (ship first)**
- Sign-in, multi-account UI and switching (**FR-3 complete**), token lifecycle (**FR-4 complete**).
- **FR-1 partial**: Windows 365 enumeration via Graph v1.0 `GET /me/cloudPCs`. AVD entries limited to
  admin-preconfigured workspace/resource IDs (end users cannot discover them; section 8).
- **FR-2 web-only**: direct-launch `ent/` and `avd/` URLs with `?tenant=` / `#loginHint=`; native option shown
  disabled with reason.
- **FR-5** via beta `/me` actions behind a "beta API" feature flag (section 7.2).
- Lowest risk; entirely on supported/public API surface except the flagged beta actions.

**Phase 1 — Native MVP (TCP-only)**
1. Validate the target hardware/software stack against **FreeRDP 3.30.0+** using upstream-supported AVD flags
   (`<rdpw file> /gateway:type:arm /sec:aad`) with one manually obtained `.rdpw`. **Caveat:** the web-client
   download option was removed in July 2026 (section 8), so even this one-off validation artifact may require
   capturing the file from a Windows client's cache (`%TEMP%`-cached `.rdp` from the Windows App/RD client
   subscription) or from a tenant where the portal still offers export.
2. If TCP-only performance is acceptable, reverse-map the **AVD feed protocol → programmatic signed `.rdpw`
   acquisition** (feed discovery → workspace feed download; section 8).
- Landing the feed work simultaneously unlocks proper AVD enumeration, **completing FR-1** and enabling the
  **FR-2 native** option for both providers. Everything below `.rdpw` acquisition in the stack is already
  handled by FreeRDP.

**Phase 2 — UDP Shortpath (separate project)**
- Implement **[MS-RDPEMT]** / **[MS-RDPEUDP2]** client-side UDP multitransport in FreeRDP: direct Shortpath via
  STUN, relayed fallback via TURN. FreeRDP's long-standing upstream UDP issue remains open — enabling
  `+multitransport` today negotiates and then falls back because client UDP is unimplemented. ([GitHub][a9])
- Until this lands, native sessions stay on the TCP/443 reverse-connect gateway. Explicitly optional and
  decoupled from Phase 1 delivery.

---

## 12. Risks and open questions

1. **Admin-consent wall.** Both CloudPC delegated scopes require tenant admin consent; the app is unusable in a
   tenant until an admin approves, and personal Microsoft accounts are unsupported entirely. A first-run
   "request admin approval" UX (section 9) is mandatory, not optional.
2. **Beta-only self-service actions.** All `/me/cloudPCs/{id}/*` actions are Graph beta — "production use not
   supported." Feature-flagged with graceful degradation (section 7.2); watch for v1.0 promotion.
3. **"Reset" is only partially self-service.** Restore-to-snapshot and resize have no `/me` path even in beta
   and require admin virtualEndpoint APIs (plus snapshot enumeration). Oddly, destructive **reprovision is**
   self-service in beta — hence the strong confirmation gate. The UI must distinguish end-user vs admin
   capability per account.
4. **AVD feed protocol undocumented for third parties.** Both AVD enumeration and `.rdpw` acquisition depend on
   reverse-mapping it; unknown token-audience/first-party-client-ID requirements create schedule risk and a
   potential support/ToS risk. This is the roadmap's critical path — and it hardened in July 2026: the manual
   `.rdpw` download was removed from the web client, Microsoft confirmed no public API exists, and no public
   client-side implementation of the ARM feed protocol could be found anywhere on GitHub (section 8). Even the
   Phase 1 one-off validation `.rdpw` is now nontrivial to obtain.
5. **Phase-0 AVD discovery gap.** Direct-launch AVD IDs are ARM-derived (admin RBAC); end users cannot
   self-enumerate AVD until the feed work lands, so Phase 0 AVD support is admin-provisioned bookmarks only.
6. **Redirect-registration trap.** Registering the redirect URI as `spa` silently caps refresh tokens at 24
   hours; the registration must be a native public-client loopback redirect to get standard (90-day-inactivity)
   refresh-token behavior. ([Microsoft Learn][r3])
7. **URL/API churn.** `windows.cloud.microsoft` direct-launch URLs are a documented end-user feature (updated
   July 2026), not a versioned API contract; Cloud PC Graph APIs are absent in DOD/21Vianet clouds. Concrete
   evidence of churn: `getCloudPcLaunchInfo` is deprecated with a hard stop on **October 30, 2026**
   (successor: `retrieveCloudPcLaunchDetail`), and the AVD web client's `.rdpw` export disappeared with the
   July 2026 `windows.cloud.microsoft` migration.
8. **CAE/claims challenges.** Graph may issue claims challenges mid-session; the token state machine must
   round-trip the `claims` parameter (section 6.4) — a plain retry loop will spin forever.
9. **Second-RemoteApp caveat.** Launching a second AVD RemoteApp web tab from the same host pool disconnects
   the first; the web launcher should warn when this applies.

---

## 13. References

### Microsoft — connectivity and product
[1]: https://learn.microsoft.com/en-us/windows-365/enterprise/understanding-remote-desktop-protocol-traffic "Understanding Network Flows - Remote Desktop Protocol | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/windows-365/end-user-access-cloud-pc "Accessing Cloud PCs | Microsoft Learn"
[a2]: https://learn.microsoft.com/en-us/azure/virtual-desktop/network-connectivity "Understanding Azure Virtual Desktop network connectivity | Microsoft Learn"
[a5]: https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-remotedesktop "RemoteDesktop Policy CSP | Microsoft Learn"
[a6]: https://learn.microsoft.com/en-us/windows-app/direct-launch-urls "Access desktops and apps using direct launch URLs for Windows App in a web browser | Microsoft Learn"
[a7]: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpeudp2/e7c5f83e-6574-4f10-90b0-f51b6f72c752 "[MS-RDPEUDP2]: Introduction | Microsoft Learn"
[a8]: https://learn.microsoft.com/en-us/windows-365/enterprise/understanding-remote-desktop-protocol-traffic "Understanding Network Flows | Microsoft Learn"

### Microsoft — Graph and identity
[4]: https://learn.microsoft.com/graph/api/resources/cloudpc-api-overview?view=graph-rest-beta "Working with Windows 365 Cloud PCs using the Microsoft Graph API | Microsoft Learn"
[r1]: https://learn.microsoft.com/en-us/graph/api/user-list-cloudpcs?view=graph-rest-1.0 "List cloudPCs for user (v1.0) | Microsoft Learn"
[r1b]: https://learn.microsoft.com/en-us/graph/api/cloudpc-reboot?view=graph-rest-beta "cloudPC: reboot (beta, /me path) | Microsoft Learn"
[r1c]: https://learn.microsoft.com/en-us/graph/api/cloudpc-restore?view=graph-rest-beta "cloudPC: restore (admin path only) | Microsoft Learn"
[r1d]: https://learn.microsoft.com/en-us/graph/api/resources/cloudpc?view=graph-rest-v1.0 "cloudPC resource type (v1.0) | Microsoft Learn"
[r2]: https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens "Microsoft Entra access tokens | Microsoft Learn"
[r3]: https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens "Microsoft Entra refresh tokens | Microsoft Learn"
[r3b]: https://learn.microsoft.com/en-us/entra/identity-platform/configurable-token-lifetimes "Configurable token lifetimes | Microsoft Learn"
[r4]: https://learn.microsoft.com/en-us/entra/identity-platform/msal-acquire-cache-tokens "Acquire and cache tokens with MSAL | Microsoft Learn"
[r5]: https://learn.microsoft.com/en-us/entra/msal/dotnet/acquiring-tokens/acquire-token-silently "AcquireTokenSilent / multi-account pattern | Microsoft Learn"
[r6]: https://learn.microsoft.com/en-us/graph/permissions-reference "Microsoft Graph permissions reference | Microsoft Learn"
[r7]: https://learn.microsoft.com/en-us/graph/api/cloudpc-retrievecloudpclaunchdetail?view=graph-rest-beta "cloudPC: retrieveCloudPcLaunchDetail (beta) | Microsoft Learn"
[r7b]: https://learn.microsoft.com/en-us/graph/api/cloudpc-getcloudpclaunchinfo?view=graph-rest-beta "cloudPC: getCloudPcLaunchInfo (deprecated, stops Oct 30 2026) | Microsoft Learn"

### August 2026 research pass — SDKs and connection documentation
[n1]: https://github.com/Devolutions/IronRDP "IronRDP — Rust implementation of RDP | GitHub"
[n2]: https://learn.microsoft.com/en-us/previous-versions/remote-desktop-client/overview "Remote Desktop client overview (previous-versions) | Microsoft Learn"
[n3]: https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties "Supported RDP properties — Azure Virtual Desktop | Microsoft Learn"
[n4]: https://github.com/FreeRDP/FreeRDP/issues/13094 "RDPW file can no longer be downloaded · Issue #13094 · FreeRDP/FreeRDP"
[n5]: https://learn.microsoft.com/en-us/answers/questions/5562259/is-there-a-public-api-to-download-rdpw-files-for-a "Is there a public API to download .rdpw files for AVD? | Microsoft Q&A"
[n6]: https://learn.microsoft.com/en-us/azure/virtual-desktop/uri-scheme "URI schemes with the Remote Desktop client (ms-avd / ms-rd) | Microsoft Learn"
[n7]: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-tswp/1fc83092-67b5-4091-bd6f-256ce6658e80 "[MS-TSWP]: Terminal Services Workspace Provisioning Protocol | Microsoft Learn"
[n8]: https://github.com/kimmknight/raweb "RAWeb — web interface / workspace feed server for RemoteApps | GitHub"

### FreeRDP
[a1]: https://github.com/FreeRDP/FreeRDP/issues/11951 "Azure Cloud PC/AVD RD Gateway sometimes rejects FreeRDP · Issue #11951"
[a3]: https://github.com/freerdp/freerdp/wiki/FAQ "FAQ · FreeRDP/FreeRDP Wiki"
[a4]: https://github.com/FreeRDP/FreeRDP/releases "Releases · FreeRDP/FreeRDP"
[a9]: https://github.com/FreeRDP/FreeRDP/issues/4978 "UDP support for FreeRDP · Issue #4978"
