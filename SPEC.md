# lightning-mesh-islands — Partition & Heal Visualizer

**Status:** Design spec, ready to build (2026-11-02). Two-day hackathon, 3–8 people (§8).
**Upstream:** [`WorldTreeNetwork/lightning-mesh`](https://github.com/WorldTreeNetwork/lightning-mesh), checked out beside this repo at `../lightning-mesh`.

## 1. Purpose

Make Lightning Mesh's core claim visible: **symmetric nodes with no authority
split apart, keep working, and agree again when they reconnect.** The viewer
shows every node's *own view* of mesh state side by side, so divergence during
a partition and convergence after heal are visible, timed, and explainable.

Secondary purpose: this is a learning vehicle for the builder. Every feature
should force contact with a real mechanism (CRDT merge, HLC ordering, gossip
anti-entropy, liveness, babel route withdrawal) rather than paint over it.

### Terminology — read this first

Upstream `docs/network-coordination/island-formation.md` uses **"island"** for a
*different* thing: co-located nodes sharing one client `/24` for IP-keeping
roam. That feature is **out of scope** here. In this repo:

| Term | Meaning here |
|---|---|
| **partition** | A set of nodes that can currently gossip with each other. A healthy mesh is one partition. |
| **view** | One node's local projection of mesh state (its CRDT books + derived facts). |
| **divergence** | Two nodes' views of the same book differ. |
| **convergence** | All nodes' views of every book are equal (same digest). |
| **heal** | Links restored; time from heal to convergence is the headline metric. |

Use "partition" in code and UI. If the repo name's "islands" shows up in UI
copy, define it as "partition" on screen.

## 2. Facts about upstream this design depends on

Verified against `lightning-mesh@2639fd7`. Re-verify anything marked ⚠ before
relying on it.

**CRDT layer** — `crates/mjolnir-mesh/src/crdt/`
- Gossip message variants (`crdt/gossip.rs`, `GossipMessage`): `LeaseUpdate`,
  `LeaseRelease`, `DnsUpdate`, `ServiceUpdate`, `SubnetClaimUpdate`,
  `SubnetClaimRelease`, `PeerAddrUpdate`, `UserUpdate`, `ServicePublishV2`,
  `ServiceUnpublishV2`, `LivenessBeacon`, `NodeNameAnnounce`,
  `LeasedNamePublish`, `CoordinateAnnounce`.
- Wire encoding: `crdt::sync::{encode, decode}` (postcard).
- HLC (`crdt/hlc.rs`) has explicit-time entry points `tick_at(physical_ms)` and
  `observe_at(other, physical_ms)` → **virtual time is possible.** ⚠ The global
  helpers `tick_hlc` / `observe_hlc` read `SystemTime`; check that the apply
  functions you call don't use them internally.
- Liveness (`crdt/liveness.rs`, `LivenessTracker`) takes `now_ms` on every call
  → virtual time works. Daemon constants (`bin/mjolnir-meshd.rs`):
  `ANTI_ENTROPY_INTERVAL = 20s`, `LIVENESS_STALE_THRESHOLD = 60s`,
  `LIVENESS_HARD_EXPIRY = 3600s`, `RADIO_INTERVAL = 10s`.
- Service-name conflicts: `apply_service_publish_v2_tracking_loss` records the
  loser in a `LostNameMap`; the daemon projects it as `lost_names`.

**Reference harness:** `crates/mjolnir-mesh/tests/service_mesh_convergence.rs`
is a *synchronous* multi-node sim: real apply functions, real postcard wire,
messages queued per `(from, to)` link so a test can **hold a link (partition)
and flush in any order (heal)**. The simulator here generalises that pattern.
Read it before writing any sim code.

**Per-node state on a real node**
- meshd writes `/var/run/mjolnir/directory.json` (`DirectorySnapshot`, schema
  v1, about every 20s): `node {node_id, subnet, backhaul_addr, name, ula, …}`,
  `neighbors[] {node_id, addrs, subnet, name}`, `identities[]`, `services[]`,
  `lost_names[] {name, winner_node_id}`, `gateways[] {node_id, cost_hint}`,
  `https_names[]`.
- meshd writes `/var/run/mjolnir/radio.json` (`RadioSnapshot`, v1, 10s):
  `backhaul_addr, mesh_if, mesh_mac, channel, freq_mhz, collected_at_unix,
  stations[] {mac, signal_dbm, expected_throughput_mbps, inactive_ms},
  mpaths[] {dst, next_hop, metric}`.
- `mjolnir-hello` serves these files as `GET /api/directory`, `/api/node` and
  `/api/radio` on each node's **LAN address** `10.42.<x>.1:80` (not the
  overlay address). CORS is on for GET `/api/*`.
- meshd control API: `127.0.0.1:5380` (`GET /v0/directory`,
  `POST /v0/publish`, `/v0/unpublish`). Localhost only, so use SSH to reach it.
- babeld has **no** control socket configured. Read routes with
  `ip route show proto babel` over SSH.
- The directory is a **projection**, not the raw CRDT. There is no digest or
  version vector in it. The collector computes digests from the projection
  (§5.2).

**Sim lab** — `../lightning-mesh/deploy/sim/README.md`
- Runs only on a **Linux x86_64 KVM host** (libvirt/QEMU OpenWrt guests,
  vwifi + `mac80211_hwsim`). **Not runnable on macOS.**
- Mgmt network `10.99.0.0/24` (node-a `.10`, node-b `.11`, …); SSH survives a
  partition.
- Partition lever: `killall vwifi-server` (or `vwifi-ctrl set <id> …`).
  Overlay `10.254.x` dies while mgmt SSH still works.
- Known red tests: `fade` (distance doesn't drop the overlay), `dhcp-crdt`,
  `keep-ip`. Don't build a demo on those mechanisms.

**`tests/two_site_netns.rs`** is three `#[ignore]` stubs that `panic!`. They
describe the right scenario but give you no harness.

**License:** upstream is AGPL-3.0-or-later (or commercial). This repo links
`mjolnir-mesh` as a library, so license it AGPL-3.0-or-later.

## 3. Scope

### In scope (MVP — must ship)
1. **Simulator backend** (`islands-sim`): N in-process nodes (default 5, up to
   12) built on the real `mjolnir-mesh` CRDT lib, with per-link message queues,
   virtual clock, partition/heal levers, scripted scenarios. Runs on macOS.
2. **Viewer** (web UI): topology graph, per-node view panels, divergence
   matrix, event timeline, controls.
3. **Three scripted scenarios** (§6) that run start to finish, unattended, with
   a pass/fail convergence assertion.
4. **NodeView contract** (§5) shared by every backend.

5. **Live-fleet collector** (read-only): poll `/api/directory` and `/api/radio`
   on the venue mesh's LAN gateways. Promoted to MVP because a mesh is
   installed at the venue. The viewer laptop must be joined to the mesh SSID
   to reach the `10.42.<x>.1` LAN gateways. Partitions are physical (unplug or
   power off a node), and only with the mesh owner's OK.

### Stretch (in order)
6. **Sim-lab collector**: SSH to `10.99.0.x`, read `directory.json`,
   `radio.json` and `ip route show proto babel`; partition by stopping vwifi.
   Real babel routes and real timings. Needs a Linux box with KVM (§9).
7. **Replay**: record an event log; scrub the timeline after the fact.
8. **Upstream digest proposal** (§9.4).

### Out of scope / non-goals
- Island formation (shared-subnet roaming), from upstream.
- **Any write to real nodes.** The live collector is read-only. No `mjolnir-apply`,
  no UCI, no publish. (Sim-lab writes are fine; it's a lab.)
- Changes to upstream `lightning-mesh` in the MVP. If an upstream endpoint
  would help (e.g. a per-book digest in `directory.json`), write it up as a
  proposal (§9), don't patch it.
- Running real `babeld` inside the simulator. Sim routes are **modeled** and
  must be labeled that way in the UI.
- Radio/RF modelling beyond link up/down.

## 4. Architecture

```
            ┌──────────────── viewer (browser) ────────────────┐
            │ topology · view panels · divergence matrix ·     │
            │ timeline · controls                              │
            └──────────────▲───────────────────┬───────────────┘
                 WS: events│+ snapshots        │HTTP: commands
            ┌──────────────┴───────────────────▼───────────────┐
            │            islands-server (Rust, axum)           │
            │   Backend trait: snapshot() · events() · act()   │
            └──────┬──────────────────┬───────────────────┬────┘
                   │                  │                   │
         ┌─────────▼──────┐  ┌────────▼────────┐  ┌───────▼────────┐
         │ SimBackend     │  │ SimLabBackend   │  │ FleetBackend   │
         │ in-process,    │  │ SSH 10.99.0.x   │  │ GET /api/* on  │
         │ mjolnir-mesh   │  │ vwifi levers    │  │ LAN gateways   │
         │ lib, virtual t │  │ (stretch)       │  │ read-only      │
         └────────────────┘  └─────────────────┘  └────────────────┘
```

### Repo layout
```
Cargo.toml                  # workspace
crates/islands-core/        # NodeView types, digest, divergence calc (no I/O)
crates/islands-sim/         # SimBackend: nodes, links, clock, scenarios
crates/islands-server/      # axum HTTP + WS; picks backend by flag
web/                        # Svelte + Vite viewer
scenarios/*.json            # scripted scenarios (§6)
SPEC.md                     # this file
```

- Depend on upstream by path: `mjolnir-mesh = { path = "../lightning-mesh/crates/mjolnir-mesh", default-features = false }`.
  **Do not enable `daemon`** (it pulls in iroh and Linux-only deps). ⚠ The first
  task is confirming the lib builds on macOS without it.
- Web stack: Svelte + Vite, to match upstream `hello-mesh-web`.
  `hello-mesh-web/src/lib/components/RadioGraph.svelte` is prior art for drawing
  the topology. Read it, but don't import across repos.
- Use JSON everywhere (scenarios, config, wire). No YAML.

## 5. Contracts

### 5.1 NodeView v1 (backend → viewer)

```json
{
  "version": 1,
  "source": "sim | simlab | fleet",
  "node_id": "a1b2…",
  "label": "node-a",
  "t_ms": 123456,
  "reachable": true,
  "books": {
    "subnet_claims": { "digest": "9f3c…", "entries": 4, "detail": [ … ] },
    "peer_addrs":    { "digest": "…", "entries": 5, "detail": [ … ] },
    "services":      { "digest": "…", "entries": 2, "detail": [ … ] },
    "node_names":    { "digest": "…", "entries": 5, "detail": [ … ] },
    "users":         { "digest": "…", "entries": 0, "detail": [] },
    "liveness":      { "digest": "…", "entries": 5, "detail": [ { "node_id": "…", "stale": false, "last_seen_ms": 120000 } ] }
  },
  "lost_names": [ { "name": "jukebox", "winner_node_id": "…" } ],
  "routes": { "modeled": true, "entries": [ { "dst": "10.42.3.0/24", "via_node": "…" } ] },
  "links": [ { "peer": "…", "up": true } ]
}
```

- `t_ms` is virtual time for `sim` and wall time for the other sources.
- `routes.modeled` is `true` for `sim` and `false` when the routes come from
  `ip route show proto babel`.
- `detail` holds the book's rows, sorted canonically, for the drill-down panel.

### 5.2 Digest rule
`digest = hex(blake3(canonical_json(sorted detail rows)))[..12]`. Exclude
fields that are naturally per-node (the node's own `link_local_*`,
`collected_at`, liveness `last_seen_ms`). For `liveness`, compute the digest
over `(node_id, stale)` only. Compute digests in `islands-core` so every
backend agrees.

### 5.3 Events (WS stream)
```json
{ "t_ms": 0, "kind": "link_down|link_up|msg_sent|msg_delivered|msg_dropped|book_changed|diverged|converged|lost_name|scenario_step|assert_pass|assert_fail", "node": "…", "peer": "…", "book": "…", "note": "…" }
```
`converged` fires when every reachable node's digests match for every book, and
carries `heal_to_converge_ms` when a heal preceded it.

### 5.4 Commands (HTTP, `POST /act`)
`cut {a, b}`, `heal {a, b}`, `isolate {node}`, `rejoin {node}`,
`split {groups: [[…], […]]}`, `heal_all`, `publish {node, name, port}`,
`unpublish {node, name}`, `kill {node}`, `revive {node}`,
`tick {ms}` (sim only), `speed {x}` (sim only), `run_scenario {file}`.
A backend returns `501` for a command it can't do. The live fleet backend
returns `501` for every mutating command.

## 6. Scenarios (MVP acceptance)

Each is a JSON file of timed steps plus assertions; `islands-server
--scenario <file> --headless` exits non-zero if any assertion fails.

1. **split-brain-name** — 5 nodes. `split [[a,b],[c,d,e]]`. `a` publishes
   `jukebox`; `d` publishes `jukebox`. Assert both sides diverge on `services`.
   `heal_all`. Assert convergence within one anti-entropy flush, **the same
   winner on every node**, and the loser in `lost_names` on the losing node.
   Shows: owner-bound service CRDT plus deterministic conflict resolution.
2. **node-death** — `kill c`. Advance virtual time. Assert `c` goes `stale` in
   every survivor's liveness view at about 60s (`LIVENESS_STALE_THRESHOLD`),
   and that its modeled routes are withdrawn. `revive c`. Assert it rejoins and
   everything converges. Shows: positive presence, not "absence of an error".
3. **order-independence** — 4 nodes, partition into pairs, generate mutations
   on both sides, then heal with three different link-flush orders (A-first,
   B-first, interleaved). Assert an identical final digest every time. Shows:
   merge is commutative, which is the thing that makes "no authority" work.
   (Port this directly from `service_mesh_convergence.rs`.)

## 7. Viewer requirements

- **Topology:** nodes as circles; links solid when up and dashed red when cut.
  Messages in flight animate along links. Clicking a link toggles it.
- **Partition coloring:** each connected component gets a hue. Node border =
  its component.
- **Divergence matrix:** rows = nodes, columns = books, cell color = digest
  (same digest = same color). Converged = one color per column. This is the
  key visual. Make it big.
- **View panel:** click a node to see its `detail` for each book, with the
  rows that differ from the majority highlighted.
- **Timeline:** horizontal event strip. Mark partition, divergence, heal and
  converge. Label "heal → converge: N ms".
- **Controls:** play/pause, speed (1×, 10×, 100×; sim only), step, run
  scenario, reset.
- A **"modeled routes"** badge whenever `routes.modeled = true`.

## 8. Plan: two days, 3–8 people, parallel lanes

The **contracts in §5 are the seams between lanes.** Freeze NodeView v1 and the
event schema in the first hour. After that, no lane waits on another.

### Hour 0 (everyone, ≤1h)
- Agree on §5. Commit `fixtures/nodeviews/*.json` (hand-written NodeViews for
  5 nodes: converged, split, conflicting) and `fixtures/events/*.jsonl` (a
  hand-written partition → heal stream).
- Lane A starts M0 immediately; it's on the critical path.

### Lanes

| Lane | People | Owns | Starts from | Done when |
|---|---|---|---|---|
| **A — Sim core** | 1–2 | `islands-core`, `islands-sim` | upstream lib + `service_mesh_convergence.rs` | Scenarios 1–3 pass headless |
| **B — Viewer** | 1–2 | `web/` | `fixtures/` (no backend needed) | Topology, matrix, timeline and controls render from fixtures, then from live WS |
| **C — Server + scenarios** | 1 | `islands-server`, `scenarios/` | §5 contracts; a mock backend that replays `fixtures/events` | `curl`/WS serves NodeViews; `--headless` runner exits non-zero on failed asserts |
| **D — Venue mesh** | 1–2 | `FleetBackend` | §2 facts; the installed mesh | Live venue mesh renders read-only in the viewer |
| **E — Sim lab** *(if ≥6 people)* | 1 | `SimLabBackend` | `../lightning-mesh/deploy/sim/README.md` | Real `proto babel` routes visible; vwifi partition shows in the viewer |

With 3 people, merge lanes: A+C, B, D. Lane E is the first thing to drop.

### Milestones

| # | Deliverable | Lane | Target |
|---|---|---|---|
| M0 | Workspace builds on macOS against `../lightning-mesh` lib, without `daemon`; first CRDT apply called from `islands-sim` | A | Day 1, morning |
| M1 | Sim: nodes, link queues, virtual clock, `services` + `subnet_claims` + `liveness` books; scenario 3 passes | A | Day 1 |
| M2 | Server: NodeView + digest over WS/HTTP, mock backend, headless runner | C | Day 1 |
| M3 | Viewer runs from fixtures, then from the live server; cut/heal by click | B | Day 1 → Day 2 morning |
| M4 | Scenarios 1 and 2 pass headless and play visibly | A + C | Day 2 midday |
| M5 | Venue mesh in the viewer (read-only) | D | Day 2 midday |
| M6 | Sim-lab collector | E | Day 2, if time |
| — | **Integration freeze, demo rehearsal** | all | Day 2, last 3h |

**Demo arc:** simulator (scenario 1, split-brain name, at 10×) → divergence
matrix goes multicolor → heal → one color, with "heal → converge: N ms"
on screen → switch the backend to the live venue mesh and show the same
views on real hardware.

Pre-hackathon prep, if the rules allow it: M0, the fixtures, and reading the
upstream files in §2.

## 9. Decisions and open questions

Resolved 2026-11-02:
1. **Length and team:** two days; 3–8 people working in small teams or
   independently. This is why §8 is organised as lanes.
2. **Hardware:** a Lightning Mesh is installed at the venue, so the live-fleet
   collector is MVP (lane D). ⚠ Confirm how many nodes it has. A visible
   partition needs ≥2, and ≥3 makes the demo better. Also confirm who may
   unplug them.
3. **Linux box:** one should be available. The sim lab needs: Linux x86_64,
   hardware virtualization (KVM, so `/dev/kvm` exists), root, libvirt/QEMU,
   about 4 GB free RAM for 5 small OpenWrt VMs, and an upstream checkout to
   run `deploy/sim/build-*.sh`. Setting it up from scratch takes hours, so
   provision it before the event or drop lane E.

Open:

4. **Upstream digest proposal.** A real node's `directory.json` is a
   *summary* of its CRDT state, so the fleet view computes digests from that
   summary (§5.2). Two nodes could differ in a field the summary leaves out
   and still look converged. The fix is a small additive upstream change to
   `lightning-mesh`: meshd writes one digest per book (and its HLC
   high-water mark) into `directory.json`. Not needed for the MVP. Reaching
   the venue mesh would also mean redeploying its routers, which the mesh
   owner must approve.

## 10. First steps for the building agent

1. Read upstream: `crates/mjolnir-mesh/tests/service_mesh_convergence.rs`,
   `src/crdt/{gossip,sync,hlc,liveness,service_apply,subnet}.rs`, and
   `docs/network-coordination/{gossip-and-crdt,babel-routing,p2p-resilience}.md`.
2. Check the ⚠ items in §2 (macOS build without `daemon`; apply functions
   that read `SystemTime`).
3. Write the §8 hour-0 fixtures so lanes B and C can start without lane A.
4. Lane A: build M0 → M1 → M4 in order. Keep scenario 3 green from M1 onward.
