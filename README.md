# Boston Uncommon — SOC Simulation

**Live:** https://jsj-ai.github.io/Boston-Uncommon
**Repo:** https://github.com/JSJ-AI/Boston-Uncommon

A real-time Security Operations Center simulation game centered on Boston Common. A deployed "bad guy" is detected through a layered sensor mesh — patrolling units, fixed traffic cameras, public surveillance cameras, and an acoustic microphone grid — then pursued and captured.

---

## Stack

| Layer | Technology |
|-------|-----------|
| Map | MapLibre GL JS 4.7.1 |
| Tiles | CARTO Positron (OSM-aligned) |
| Road network | OpenStreetMap via Overpass API (3-mirror fallback) |
| Traffic camera locations | OSM `highway=traffic_signals` nodes |
| Public surveillance cameras | OSM `man_made=surveillance` (public-zone only) |
| Street-level imagery | Mapillary Graph API v4 (client token, read scope) |
| Aerial imagery | Esri World Imagery (non-commercial use) |
| Hosting | GitHub Pages — JSJ-AI/Boston-Uncommon |

Single `index.html`, no build step, no framework.

---

## Operational Area

- **Center:** Boston Common (42.3551°N, 71.0659°W)
- **BBOX:** `42.338,-71.090,42.373,-71.042` — ~3.9 km × 3.9 km, ~5 sq mi
- **Road network:** ~15,000+ OSM nodes incl. park paths, cycleways, side streets

---

## Patrol Units

| Type | Count | Icon | Color | FOV | Range | Speed |
|------|-------|------|-------|-----|-------|-------|
| Drone | 3 | 14px circle | Blue `#0284c7` | 360° | 138 m | 25 km/h |
| Vehicle | 10 | 10x14 px car | Green `#16a34a` | 130° | 68 m | 20 km/h |
| Bike | 10 | 14px wheels | Orange `#ea580c` | 130° | 35 m | 15 km/h |
| Officer | 10 | 12x14 px person | Purple `#7c3aed` | 130° | 35 m | 5 km/h |

Units patrol on OSM roads using way-ID navigation (no reversing, 30s turn minimum, 3s intersection pause). Cars stay on roads; bikes and officers can use park paths + cycleways. Drones move line-of-sight.

---

## Sensor Mesh

### Traffic cameras
- Placed at every OSM `highway=traffic_signals` node in the BBOX (~50–80 in Boston Common area)
- Tiny 8px amber dots to minimize clutter
- **Per-road-approach FOV**: each camera emits a 90° wedge along every connecting road (derived from road-graph neighbours). 4-way intersections get 4 wedges, T-junctions 3, dead-ends 1 — coverage flows along streets instead of radiating cardinal directions
- Wedge turns red when bad guy enters coverage → triggers all-unit dispatch

### Surveillance cameras
- OSM `man_made=surveillance` (public/traffic/town zones only — no private home cams)
- Larger CCTV icon (red, 22px) so the rarer red ones stand out from the amber traffic dots
- 4-wedge omni coverage (no street-direction context for these)

### Acoustic microphone mesh
- 20 mics in a 4×5 interior grid with light jitter
- **Collision avoidance**: each mic checks distance to nearest camera/agent and shifts ~70m away if within 50m (up to 6 push attempts)
- Listening radius: 700m so 3+ mics hear any point in the operational area (triangulation overlap)
- Mesh visualization: semi-transparent pink fills naturally stack to deeper hue in overlap zones
- Toggleable in the left panel ("Mics" button)

### Hover snapshots (real imagery, no synthetic placeholders)
- **Drones** → Esri aerial tile of the drone's coordinates (synchronous, "AERIAL VIEW" label)
- **Traffic cams, surveillance, vehicles, bikes, officers** → Mapillary street-level photo of the sensor's location (async, "STREET VIEW" label)
- Mapillary cache: 30-minute TTL by rounded lat/lon (~10m precision)
- Fallback when Mapillary has no coverage: Esri aerial with honest label "AERIAL (no street coverage)"

---

## Right-Panel Sensor Feed

Pinned snapshot from whichever camera-bearing object is closest to the active bad guy. Refreshes every 2s; image refetches only when the closest sensor changes. Shows sensor ID + distance in feet. Idle state when no incident.

---

## Bad Guy + Controls

Click **Deploy Bad Guy** → choose Red Car / Red Bike / Red Person → click map. Snaps to nearest valid road for that mode.

**Control panel** (after deployment):
- **Mode swap**: Car ↔ Bike ↔ Foot. Preserves position (re-snaps to nearest valid road for new mode), updates icon, caps speed to new mode max (Car 30, Bike 18, Foot 6 km/h). Timer keeps running.
- **Joystick**: 130×130 round draggable joystick. Direction = bearing, radial distance = speed. Release leaves stick where dropped (bad guy keeps moving). Touch + mouse supported.
- **Fine controls** (collapsible): max-speed slider, 8-way compass buttons.
- **Fire Weapon**: triggers acoustic event.

---

## Gunshot / Acoustic Dispatch

When the bad guy fires:
- Every mic within its 700m listening radius "hears" the shot (no decibel attenuation — geometric)
- Status: ≥3 mics → "Triangulation confirmed" (validated). 1–2 mics → "Confidence: low" (still dispatches). 0 mics → "Event unconfirmed" (no dispatch)
- **Persistent triangulation viz**: hearing mics' listening circles render at 18% opacity (overlap zones stack to deeper pink), hearing-mic icons get glowing pink halos, a pulsing pink source marker drops at the shot location
- Viz **stays pinned to the shot coordinates** even if the bad guy moves — clears on capture or scenario reset
- Dispatches closest car + bike + officer + drone (no FOV requirement, passive acoustic trigger)

---

## Detection → Dispatch → Capture Pipeline

Three independent detection paths, all routing to the same dispatch logic:

1. **Visual (agents)** — bad guy in agent's FOV polygon → dispatch
2. **Camera (intersection)** — bad guy in any traffic/surveillance cam wedge → dispatch
3. **Acoustic (gunshot)** — bad guy fires, mics hear → dispatch

`dispatchPursuit(detector)` for visual/camera; `dispatchAcoustic()` for gunshot.

**Capture**: closest car + non-drone backup both within 8m → 30-second countdown → "TARGET CAPTURED!" alert, response time recorded, units return to patrol.

---

## Alert Feed (right panel)

Slide-in animation with color-coded glow flash per alert type (info=cyan, warn=amber, bad=red, ok=green). Bad and Ok alerts get a gradient background tint. Newest at top, capped at 50 entries.

---

## Left Panel — Asset Grid

| Asset | Default count |
|------|---------------|
| Drones | 3 |
| Vehicles | 10 |
| Bikes | 10 |
| Officers | 10 |
| Cameras | dynamic (from OSM) |
| Mics | 20 |

Two compact toggle buttons below: **Cams** (show/hide cameras + FOV wedges) and **Mics** (show/hide mic icons + mesh).

---

## Architecture Notes

- Single `index.html`, ~2,200 lines
- TICK_MS = 500ms simulation loop
- FOV re-rendered every 2 ticks for performance
- Camera FOV uses `roadGraph.nodes[cam.id].neighbors` to derive per-road bearings
- Race-condition guards on async image fetches: `sensorPopupSeq` and `sensorFeedSeq` counters invalidate in-flight fetches when the user moves on

---

## Related Projects (same stack)

| Project | URL | Status |
|---------|-----|--------|
| MBTA Live | jsj-ai.github.io/mbta-live | ✅ Working |
| MA Traffic Twin | jsj-ai.github.io/ma-traffic-twin | ✅ Working |
| Boston Uncommon | jsj-ai.github.io/Boston-Uncommon | ✅ Working (this project) |
| MA NWSS Wastewater | jsj-ai.github.io/MA-NWSS-Wastewater | Needs CARTO tile fix |
| SNAP Dashboard | jsj-ai.github.io/SNAP | Static, no map |

---

## Known Open Items / Backlog

- [ ] Pursuit-path visualization (line from each pursuer to bad guy)
- [ ] Multiple bad guys (architecture ready, UI capped at 1)
- [ ] Click-to-edit individual agent properties (per-unit attribute panel)
- [ ] Score / response-time history / leaderboard
- [ ] NWSS site CARTO tile fix
- [ ] Mapillary token rotation guidance if rate-limited

---

*Built with J2Analytics · Boston Uncommon · May 2026*
