# WindowsAppForLinux

A Linux desktop client for **Windows 365 Cloud PCs** and **Azure Virtual Desktop** — analogous to Microsoft's
Windows App, which is not available for Linux.

Planned capabilities:

- Entra ID sign-in with multiple accounts and account switching
- Unified enumeration of the user's Cloud PCs (Microsoft Graph) and AVD desktops/RemoteApps (AVD workspace feed)
- Per-resource choice of connection method: native (FreeRDP 3.30.0+ ARM-gateway path) or web (direct-launch URLs)
- Cloud PC management actions via Graph: restart, rename, troubleshoot, reset/reprovision
- Resilient token lifecycle: silent refresh, re-auth prompts, per-account encrypted token cache

See [spec.md.txt](spec.md.txt) for the full product and architecture specification, including the delivery
roadmap (Phase 0 web-first client → Phase 1 native TCP-only MVP → Phase 2 UDP Shortpath).
