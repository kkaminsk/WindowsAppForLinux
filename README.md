# WindowsAppForLinux

A Linux desktop client for **Windows 365 Cloud PCs** and **Azure Virtual Desktop** — analogous to Microsoft's
Windows App, which is not available for Linux.

> **Status:** specification and design stage — no implementation code yet.

Planned capabilities:

- Entra ID sign-in with multiple accounts and account switching
- Unified enumeration of the user's Cloud PCs (Microsoft Graph) and AVD desktops/RemoteApps (AVD workspace feed)
- Per-resource choice of connection method: native (FreeRDP) or web (direct-launch URLs)
- Cloud PC management actions via Graph: restart, rename, troubleshoot, reset/reprovision
- Resilient token lifecycle: silent refresh, re-auth prompts, per-account encrypted token cache

## Native connectivity

FreeRDP 3.30.0+ already implements everything below the connection configuration for Windows 365/AVD — Entra
token acquisition, the ARM gateway negotiation, the RDP session itself. The remaining gap is that the
connection-critical `.rdpw` config comes only from the undocumented AVD feed-discovery endpoint, which FreeRDP
does not call. This project closes that gap with a small Connection-Config Provider and aims to upstream feed
support into FreeRDP, targeting a distributable product (with the web client as the always-available fallback).

Delivery follows a gate-first staged plan:

1. **Stage 0** — app-identity feasibility spike (can our own Entra app registration get an AVD feed token?)
2. **Stage 1** — feed schema capture (pin down the undocumented feed contract)
3. **Stage 2** — Connection-Config Provider (token → feed discovery → generated `.rdpw`)
4. **Stage 3** — FreeRDP handoff (native session end to end, TCP-only)
5. **Stage 4** — upstream feed/`.rdpw` acquisition to FreeRDP

## Documentation

- [spec.md.txt](spec.md.txt) — full product and architecture specification
- [Windows 365 / AVD Native Connectivity on Linux — Design](docs/superpowers/specs/2026-08-18-windows365-native-connectivity-design.md)
  — approved design for the native path; the current execution plan
