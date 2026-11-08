# hop-gui

A static cluster dashboard for Hop, built with the shared Haasstyle components.

`index.html`, `app.js`, `appearance.js`, `style.css` and `vendor/`: no build
step, no CDN, no frontend framework, and no server of its own.

## Hosting

Cloudflare Pages serves this repository as it is: no build command, the
repository root as the output directory. Any static host will do; for a
quick local look, `python3 -m http.server` in this directory (Web Crypto,
needed for a cluster with a key, works on `localhost`).

## Connecting to a cluster

Open the dashboard, choose **New server** below the server list, and enter
a name, an agent address (`http://NODE:8080`) and the cluster's API key, if
it has one. `?name=NODE&ip=IP` prefills the form.

**The browser talks to the agent directly.** The page is static, and every API call goes from the browser to the agent address
you entered: that address must be reachable from the browser, and the agent
must allow the cross-origin request. Hop agents do (on the agent port,
8080 by default, on every answer, including answers that come from the
leader and streams):

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, DELETE, PATCH, OPTIONS
Access-Control-Allow-Headers: Content-Type, X-Hop-Auth
```

The preflight (`OPTIONS`) needs no signature. When a public origin asks
for a LAN address, Chrome's Private Network Access preflight is answered
with `Access-Control-Allow-Private-Network: true`. The leader port (9080)
is not meant for browsers and sends no CORS headers.

With an API key, every request carries
`X-Hop-Auth: hex(HMAC-SHA256(key, METHOD + "\n" + PATH + "\n" + hex(sha256(body))))`,
computed with Web Crypto, so the key itself never travels. Web Crypto only
works in a secure context: serve the dashboard on `localhost` or over HTTPS
(Cloudflare does), or use a cluster without a key (`hopos.insecure=1`).

The calls the dashboard makes, all on the agent, which hands the `/v1/`
ones to the leader:

| Call | For |
| --- | --- |
| `GET /leader` | the leader address in the header |
| `GET /v1/status` | `agents`, `jobs`, `total_placed`, `settling`, `placed` |
| `GET /v1/agents` | the agents table |
| `GET /v1/agents/{id}/capacity` | CPU, memory, tasks and attributes per agent |
| `GET /v1/tasks` | the tasks of an open agent (`tasks_by_agent`), the kernel and Hop as `system` |
| `GET /v1/jobs`, `POST /v1/jobs` | the jobs table, a new job, a redeploy |
| `DELETE /v1/jobs/{name}` | delete a job |
| `PATCH /v1/jobs/{name}/priority` | the drag order (`{"priority": index}`) |
| `GET /v1/jobs/{name}/status` | the task table (`tasks_by_agent`) |
| `GET /v1/agents/{id}/logs/{task}/{stdout,stderr}` | the live log (SSE; without a query it follows, `?follow=0` is a snapshot) |
| `GET /v1/events` | live updates (SSE) |

The hop repository checks that every one of these exists, with this
shape: its test `api/src/tests_gui.rs` greps this `app.js` when this
repository sits next to it (`../hop-gui`, or `HOP_GUI_DIR`).

## Dashboard

- Saved clusters, live updates over SSE, agent failover and fallback polling.
- Agent CPU, memory, temperature and task counts.
- Jobs with draggable priority, creation, redeployment and deletion.
- Job details, task status and live stdout/stderr logs, also of a node's
  system tasks: `kern` (the console of HopOS itself) and `hop` (Hop's own lines).
- Direct links to `#agents`, `#agents/<encoded-id>`, `#jobs` and `#jobs/<encoded-name>`.
- `?name=NODE&ip=IP` prefills the cluster form without connecting automatically.
- Responsive tables and keyboard-accessible forms, confirmations and dialogs.

## Appearance

Use **Appearance** at the bottom of the left sidebar, in the same place as EasyACP. Choose these independently:

- Theme: Senior / Classic 95, Medior / Glossy 08, Junior / Tactile Matte.
- Accent colour: purple, red, blue, petrol or green.
- Mode: dark or light.

Glossy 08, purple and dark are the defaults. Appearance is stored as `hop-appearance` in this browser, applied before the stylesheet loads, and synchronized across open tabs. Existing `hop-clusters`, `hop-active-cluster` and endpoint pool settings are retained.

## Maintaining the styles

`style.css` handles Hop's layout. The material, controls, spinner, select and decision dialogs come from `vendor/`; see [vendor/README.md](vendor/README.md). Keep these shared files in sync with Haasstyle instead of recreating their visuals in app-specific CSS.

`appearance.js` manages the picker and theme tokens. `app.js` handles the Hop API and renders the same shared components for dynamic content.

Saved servers use the shared vertical `t-nav` / `t-choice` navigation in the
left sidebar. **New server** sits immediately below the list; **Appearance**
sits in the sidebar footer. Up/Down and Home/End navigate the server list.
The dashboard's delete icon removes the selected server from this browser,
after confirmation; its workloads keep running. On narrow screens the sidebar
stacks above the dashboard, keeping the server list vertical.

Icon actions remain at least 36 × 36 px; tags keep the shared 32 px minimum
height and tables vertically center text, status labels and actions.
