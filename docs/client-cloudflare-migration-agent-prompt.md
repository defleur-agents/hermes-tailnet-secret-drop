# Client migration prompt: Tailscale to Cloudflare Access

Use this prompt with the agent on an existing WizardOS/Hermes client VPS. Replace only the angle-bracket inputs before sending it.

```text
Migrate this WizardOS/Hermes client's user-facing web access from Tailscale to Cloudflare Access without disrupting working services. This run must keep Tailscale installed as the rollback/admin path; retirement of Tailscale is a later, separately approved operation.

Stop at the first failed verification in any phase and report the precise blocker. Continue only after that gate passes, and never retry a destructive step blindly.

Owner inputs:
- Cloudflare account ID: <ACCOUNT_ID>
- Cloudflare zone ID: <ZONE_ID>
- Client hostname: <CLIENT_HOSTNAME>
- Approved operator email(s): <ALLOWED_EMAILS>
- Cloudflare API token file: <OWNER_ONLY_LOCAL_FILE>

The API token must be supplied through an owner-only local file or the client's existing secure credential intake, never pasted into chat or placed directly in a command line. Scope it to this client's account and zone with account permissions Cloudflare Tunnel: Edit and Access: Apps and Policies: Edit; account permissions Access: Organizations, Identity Providers, and Groups: Read; and DNS: Edit only for the selected zone. Never copy it out of that file. After acceptance, tell the owner to rotate or delete the migration token. Use the client's own Cloudflare account, zone, hostname, and identity policy.

Execute and verify these phases:

1. Inventory and preserve rollback
- Confirm Linux, Hermes Agent, Python 3.10+, git, systemd user services, cloudflared, and the active Hermes profile/home.
- Capture the complete current `tailscale serve status --json`, Tailscale node state, user-service inventory, loopback listeners, app route table, current user-facing URLs, Cloudflare Tunnel routes, DNS records, and Access applications/policies.
- Probe every current app locally. Classify each route as live web UI, API-only, WebSocket, stale, or privileged/admin-only. Keep stale routes in the inventory and label them honestly.
- Save owner-readable rollback evidence before any mutation. Never reset all Tailscale Serve routes and never replace resources not proven to belong to this installation.

2. Build Cloudflare ingress
- Reuse the client's reviewed Cloudflare installer or route adapter when present. Create Cloudflare Access policy before public DNS/ingress.
- Permit only the approved operator email addresses. Keep every application origin on IPv4 loopback; expose no public host listener.
- Present the exact DNS records, Tunnel routes, and Access applications/policies to the owner and obtain confirmation before creating them.
- Route the exact client hostname through a remotely managed Cloudflare Tunnel. Put exact/special routes before any catch-all. Preserve WebSocket upgrades and original target-path rewriting.
- Verify the cloudflared service is enabled, active, and configured to survive reboot.
- Verify unauthenticated requests stop at Cloudflare Access. Verify authenticated browser access using the owner's normal browser. A generic provider 403 is not proof of Access.

3. Migrate user-facing applications one route at a time
- For each live web route, add the Cloudflare mapping, update its canonical URL plus exact Host/Origin/CORS allowlists, and verify local health, authenticated page load, assets, API calls, and WebSockets where used.
- Keep privileged control managers, raw TCP services, SSH, and other non-browser admin surfaces on Tailscale unless their security boundary is explicitly redesigned and approved.
- Do not remove or rewrite the existing Tailscale route after a Cloudflare check fails. Repair and retest that route first.

4. Upgrade and migrate Hermes Secret Drop
- Use only `https://github.com/humanitylabs-org/hermes-tailnet-secret-drop` tag `v1.3.2`, commit `480976354bea4bfc2354193d0736206ba0c9bbc4`. Verify the local tag, peeled tag, and remote tag all match that exact commit.
- Choose a free loopback port, using 8805 when available, and use it consistently for the Tunnel origin and installer.
- Create an exact Cloudflare Access-protected non-root path prefix, using `https://<CLIENT_HOSTNAME>/secret-drop`, and point its Tunnel origin to `http://127.0.0.1:<SECRET_DROP_PORT>`. Present these phase-4 Access/Tunnel/DNS mutations for the same owner confirmation required in phase 2.
- Before running Secret Drop setup, prove the Access application protects that exact path: an unauthenticated request must stop at Cloudflare Access, while an authenticated request may return the expected unavailable-origin response because Secret Drop is not installed on the new loopback port yet. A generic Cloudflare error is not proof. Do not continue until this exact gate is verified.
- Run `./scripts/prereq-check.sh --mode cloudflare-access`, then `./scripts/setup.sh --access-protected-public-base-url https://<CLIENT_HOSTNAME>/secret-drop --http-port <SECRET_DROP_PORT>`.
- The installer may remove only the exact Secret Drop–owned Tailscale listener after the new loopback service passes. It must leave every unrelated Tailscale route and the Tailscale client intact. If ownership or cleanup cannot be proven, fail closed and preserve the recorded recovery state.
- Verify `hermes-secret-drop health`, prefixed loopback health, bare loopback `/health` returning 404, the public Access gate, a complete demo form, CSP pinning of the inline-script hash, plaintext JSON rejection without request consumption, a successful encrypted demo-envelope submission, discarded fake value, single-use replay rejection, and sanitized status.
- Migration invalidates pending Secret Drop links. Issue fresh links afterwards and tell the owner that earlier links are retired.

5. Acceptance and handoff
- Re-probe every migrated route through Cloudflare with real Access credentials and compare the final Tailscale Serve configuration with the preserved snapshot.
- Report: reviewed repository/tag/commit, Cloudflare account/zone/hostname identifiers without secrets, Access application and Tunnel IDs, migrated routes, deferred/admin-only routes, stale routes, user services, loopback binds, health results, WebSocket results, Secret Drop demo result, exact rollback commands, and any blocker.
- Completion means every named live route works through Cloudflare and all unrelated Tailnet routes remain available. Stop with a precise blocker if any acceptance check fails.

Never print credentials, expose an origin publicly, enable Tailscale Funnel, weaken Access policy, delete Tailscale itself, reset all Serve routes, invent success for a dead route, or claim migration complete from configuration alone.
```

## Later Tailscale retirement

Retire Tailscale only through a second approved run after the Cloudflare inventory is complete and every required user-facing route has passed authenticated verification. That run must identify the remaining admin/non-browser dependencies, replace or explicitly retain each one, then remove client and server membership without disturbing Cloudflare services.
