# Boston Uncommon — SOC Simulation

**Live:** https://jsj-ai.github.io/Boston-Uncommon  
**Repo:** https://github.com/JSJ-AI/Boston-Uncommon

A real-time Security Operations Center simulation game centered on Boston Common. Units patrol the road network, detect a deployed "bad guy" via field-of-view sensors, and coordinate pursuit and capture.

---

## Stack

| Layer | Technology |
|-------|-----------|
| Map | MapLibre GL JS 4.7.1 |
| Tiles | CARTO Positron (OSM-based, matches Overpass geometry) |
| Road Network | OpenStreetMap via Overpass API (multi-mirror fallback) |
| Hosting | GitHub Pages — JSJ-AI/Boston-Uncommon |
| Fonts | Inter + JetBrains Mono (Google Fonts) |

---

## Operational Area

- **Center:** Boston Common (42.3551°N, 71.0659°W)
- **Map bounds:** ~5-mile radius, zoom 12–18
- **Deployment radius:** 0.5 mile from center (cars), 2000ft (bikes/officers)
- **Road network:** ~15,000+ OSM nodes including park paths and cycleways

---

## Units (Auto-deployed on load)

| Unit | Count | FOV Angle | FOV Distance | Speed | Roads |
|------|-------|-----------|--------------|-------|-------|
| 🚁 Drone | 3 | 360° | 138m (~450ft) | 25 km/h | Aerial (unrestricted) |
| 🚓 Vehicle | 10 | 130° forward | 68m (~225ft) | 20 km/h | Primary/secondary/residential |
| 🚲 Bike | 10 | 130° forward | 35m (~115ft) | 15 km/h | Side roads + park paths |
| 👮 Officer | 10 | 130° forward | 35m (~115ft) | 5 km/h | Side roads + footways + park paths |

**Total: 33 units**

---

## Patrol Behavior

- All units deploy snapped to actual OSM road nodes (never mid-segment or off-road)
- Units follow **OSM way IDs** — stay on the same named road segment through intersections
- Direction changes: minimum 30 seconds, then 25% random chance per intersection
- No reversing — units never go backward on a road
- 3-second pause at intersections (realistic city behavior)
- Patrol radius: 1 mile from initial placement
- Bikes and officers use Boston Common internal paths (footway/pedestrian/cycleway tags)
- Cars restricted to drivable roads only

---

## Incident Deployment

1. Click **Deploy Bad Guy** → choose Red Car, Red Bike, or Red Person
2. Click map → bad guy snaps to nearest valid road node for that type
3. Incident timer starts immediately
4. Bad guy is **static by default**
5. Use the **attribute panel** (bottom-left) to set speed (0–30 km/h) and compass direction

---

## Detection & Pursuit

- Each tick (500ms): every unit checks if the bad guy falls within its FOV polygon
- **Car/Bike/Officer:** 130° forward wedge from current bearing
- **Drone:** full 360° circle
- On first detection:
  - **Car is always dispatched** (guaranteed, closest vehicle)
  - Closest bike, officer, and drone also dispatched
  - **Drone** flies to scene at 3× speed, then orbits 40m above target (5°/tick ≈ 18s/orbit)
  - Ground units get **50% speed boost** in pursuit mode
  - Ground units skip intersection pauses during pursuit
  - All highway restrictions relaxed during pursuit (emergency override)
  - Alert panel fires: "Unit X HAS VISUAL on Red Car. Dispatching..."

---

## Capture Logic

Two-unit capture with 30-second containment window:

1. **First unit on scene** → "Awaiting backup..." alert
2. **Car + non-drone backup both within 8m** → 30-second countdown begins
3. Status panel counts down "CAPTURE IN 29s..."
4. At zero → **"TARGET CAPTURED!"** — incident clears, timer records response time
5. Bad guy stays active the full 30 seconds — operator can attempt evasion

Drones do **not** count as backup — coordination only.

---

## Controls

| Control | Action |
|---------|--------|
| **Reset** | Stops movement, clears all units, re-deploys fresh random placement |
| **Deploy Bad Guy** | Choose type → click map to place |
| **Speed slider** | Set bad guy movement speed (0–30 km/h) |
| **Compass grid** | Set bad guy direction (NW/N/NE/W/E/SW/S/SE) |

---

## Architecture

Single `index.html` file (~1,200 lines). No build system. No dependencies beyond CDN links.

**Key classes/functions:**
- `Agent(id, type, lat, lon, pt)` — base class for all police units
- `BadGuy(type, lat, lon)` — incident object
- `buildGraph(data)` — constructs OSM node graph with way IDs and highway types
- `randomRoadPoint(lat, lon, radiusM, agentType)` — snaps to valid road node by type
- `_pickTarget()` — way-ID navigation with 30s turn timer
- `_pickPursuitTarget()` — minimizes distance to bad guy at each intersection
- `inFOV(agent)` — point-in-wedge/circle detection check
- `dispatchPursuit(detector)` — triggers car + backup + drone pursuit
- `checkCapture()` — two-unit 30-second countdown capture

**FOV rendering:** MapLibre GeoJSON source (`fov-fill` + `fov-outline` layers), updated every 2 ticks.

---

## Related Projects (same stack)

| Project | URL | Status |
|---------|-----|--------|
| MBTA Live | jsj-ai.github.io/mbta-live | ✅ Fixed (CARTO tiles) |
| MA Traffic Twin | jsj-ai.github.io/ma-traffic-twin | ✅ Fixed (CARTO tiles) |
| MA NWSS Wastewater | jsj-ai.github.io/MA-NWSS-Wastewater | Needs CARTO tile fix |
| SNAP Dashboard | jsj-ai.github.io/SNAP | Static, no map |

---

## Known Issues / Next Steps

- [ ] Bad guy road-following movement when operator sets direction (currently free movement)
- [ ] Multiple bad guy support (architecture ready, UI capped at 1)
- [ ] Individual agent attribute panel (click on agent to see/edit its properties)
- [ ] NWSS site needs same OpenFreeMap → CARTO tile fix
- [ ] Pursuit path visualization (line showing route to target)

---

*Built with J2Analytics · Boston Uncommon v0.1 · May 2026*
