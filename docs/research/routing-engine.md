# Research: which routing engine honours Dutch moped tags

Resolves [issue #2](https://github.com/AarjanM/scooterplanner/issues/2). Researched 2026-10-21 against engine
source, docs and issue trackers at these versions: GraphHopper 11.1 (2026-09-29), Valhalla 3.9.1 (2026-10-06),
BRouter 1.7.10 (2026-07-17). Vocabulary follows [GLOSSARY.md](../../GLOSSARY.md): the three Vehicle classes are
Fiets, Snorfiets and Bromfiets (a Speed pedelec routes as a Bromfiets).

## Recommendation

**Build on GraphHopper.** Its custom models are the only mechanism of the three that expresses every Dutch rule in
one declarative JSON file per Vehicle class: hard bans (`priority` × 0) and soft penalties (`priority` × 0..1) over
encoded values, plus GeoJSON `areas` (`in_<id>`) that express a Rijbaan zone as "Snorfiets: cycleway inside this
polygon → priority 0" without touching the graph. It also ships an in-tree `navigation` module that emits
Mapbox Directions v5 JSON with voice and banner instructions, supports alternative routes under Contraction
Hierarchies, and is Apache-2.0.

The one real cost: GraphHopper has **no `moped`/`mofa` tag parser out of the box**. ScooterPlanner has to write a
small Java `ImportRegistry` extension that produces `moped_road_access` and `mofa_road_access` encoded values
(a copy of `OSMRoadAccessParser` with the restriction-key chain `moped → motor_vehicle → vehicle → access`, and
`mofa → moped → …`). That is roughly one class plus a config line, and it is a one-off; every later rule change is
a JSON edit.

**Runner-up: Valhalla.** It is the only engine that already parses `moped=*`/`mofa=*` and has a dedicated
`motor_scooter` costing, and it has the best navigation output (OSRM format with `voiceInstructions` /
`bannerInstructions`, `nl-NL` narration). It lost because (a) `moped` and `mofa` are folded into **one** access bit
(`kMopedAccess = 512`), so Snorfiets and Bromfiets cannot be told apart without forking the C++ graph format;
(b) `moped=use_sidepath` is not in its Lua moped table, so a Rijbaan tagged `moped=use_sidepath` stays open to
mopeds; (c) the only area-based control is `exclude_polygons` (excludes *everything* inside), so a Rijbaan zone
cannot be expressed; and (d) each of those fixes is a Lua/C++ change followed by a full tile rebuild, not a
request-time or config-time change.

**BRouter** is a credible fallback for *route planning only*: its `lookups.dat` now carries
`moped`, `mofa` and `speed_pedelec` with `no|yes|designated|use_sidepath`, its profile language does hard bans and
penalties, and it runs in 128 MB of heap. It is not fit for Turn-by-turn navigation: its turn instructions are
"voice hints" styled for Locus/OsmAnd/Orux rather than an SDK-consumable OSRM/Mapbox structure, it has no speed-up
preprocessing (query time scales with distance), and the shipped `moped.brf` is marked "DO NOT USE FOR ACTUAL
NAVIGATION" and does not reference the `moped` tags at all.

## Comparison table

| # | Criterion | GraphHopper 11.1 | Valhalla 3.9.1 | BRouter 1.7.10 |
|---|---|---|---|---|
| 1 | Dutch legal rules per Vehicle class from OSM tags | **Expressible, after a one-off Java parser.** Custom model gives hard ban (`priority` `multiply_by: 0`) and penalty per encoded value; `areas` → `in_<id>` for Rijbaan zones; `country`, `road_class`, `road_environment == FERRY` built in. No `moped`/`mofa` parser; `traffic_sign` not imported. | **Partly out of the box, rest needs fork + rebuild.** Lua maps `moped`/`mofa` (+ `motor_vehicle` fallback, directional and `oneway:moped` variants) to one access bit; `motorroad=yes` strips moped access on trunk/primary; `use_sidepath` and `designated` not distinguished for moped; Snorfiets ≠ Bromfiets impossible without C++ change; no per-area costing. | **Expressible for a planner.** `lookups.dat` v11.1 has `moped`, `mofa`, `speed_pedelec` × `no|yes|designated|use_sidepath`, `motorroad`, `oneway:bicycle`, `route=ferry`; profile `costfactor ≥ 10000` = hard ban, else penalty. Shipped `moped.brf` ignores these tags. Way-level `traffic_sign` absent. |
| 2 | Turn-by-turn output for a navigation SDK | `/route` instructions (`sign` codes, `text`, `interval`) **plus** in-tree `/navigate/directions/v5/gh/{profile}/…` = Mapbox Directions v5 with `voice_instructions`, `banner_instructions`. No OSRM-native format. | **Best.** `format=osrm` (+ `voice_instructions`, `banner_instructions`, `polyline6`), Valhalla trip JSON with `verbal_pre/post_transition_instruction`, `begin/end_shape_index`, 43 maneuver types, `nl-NL`. | `turnInstructionMode` 0–7 (locus/osmand/comment/gpsies/orux styles) embedded in GPX/GeoJSON. Not OSRM/Mapbox shaped. |
| 3 | RAM/disk for NL graph, import, rebuild | **No NL-specific numbers published** (see §3). Documented planet: ~60 GB RAM / ~3 h base import, CH +~120 GB. NL PBF is 1.41 GB (≈1.7 % of planet). Profile/EV changes need a full re-import; CH/LM can be added later. Serving RAM ≈ graph-cache size. | **No numbers published.** Scripted image rebuilds tiles on `docker restart` when PBF hashes change; `server_threads` tuning "if valhalla gets killed when building tiles". Any Lua change = full tile rebuild. | **Smallest.** Server default `JAVA_OPTS=-Xmx128M`; NL is covered by two public segment files, `E0_N50.rd5` (79 MB) + `E5_N50.rd5` (182 MB), rebuilt on brouter.de; no local import if you use them. Own `lookups.dat` changes require regenerating segments. |
| 4 | Alternatives, via points, re-route latency | `algorithm=alternative_route` (`max_paths`, `max_weight_factor`, `max_share_factor`) **works with CH**; multiple `point`, `pass_through`, `heading`, `curbside`. CH = "very fast and lightweight"; LM "an order of magnitude faster than flexible". Per-request custom models only in LM/flex mode. | `alternates=n`, **not on multipoint routes**; location types `break|through|via|break_through`, `heading`; bidirectional A*, no CH (latency not benchmarked here). | `alternativeidx` (example shows 0; range not documented); multiple `lonlats`; two-pass algorithm whose "processing time scales linear with distance"; server `maxRunningTime=300` s guard. |
| 5 | Licence vs AGPL-3.0; container images | **Apache-2.0** (compatible). No official image; community `israelhikingmap/graphhopper` (nightly + version tags, maintainers "looking for someone to take over"). | **MIT** (compatible). Official `ghcr.io/valhalla/valhalla`, `valhalla-scripted`, `valhalla-dev`. | **MIT** (compatible). In-tree Dockerfile, published to `ghcr.io/abrensch/brouter` by `docker-publish.yml`. |

Bold marks the cells that decided the recommendation.

## Evidence per criterion

### 1. Expressing the Dutch legal rules

What a Legal route needs, in OSM terms (tag counts from [Taginfo](https://taginfo.openstreetmap.org/api/4/key/values?key=moped),
fetched 2026-10-21: `moped=no` 164 k, `designated` 72 k, `yes` 51 k, `use_sidepath` 47 k; `mofa=yes` 109 k,
`no` 85 k, `use_sidepath` 74 k, `designated` 70 k):

- Bromfietspad (G12a): `moped=designated` (+ `mofa=designated`) on `highway=cycleway`.
- Verplicht fietspad (G11): `highway=cycleway` with `moped=no` (Snorfiets allowed → `mofa=yes`).
- Onverplicht fietspad (G13): cycleway with `mofa=no`, `moped=no`.
- Rijbaan with a parallel Bromfietspad: `moped=use_sidepath` → ban or heavy penalty for Bromfiets.
- Autoweg/autosnelweg (G3/G1): `highway=motorway|trunk` + `motorroad=yes` → always banned.
- Oneway exceptions `oneway:bicycle=no`, `cycleway=opposite*`; ferries `route=ferry`.
- Rijbaan zone: polygon where Snorfiets must leave cycle paths.

> Could not verify today: the OSM wiki pages `Key:moped`, `Key:mofa` and `NL:Nederlandse_Verkeersborden` returned
> HTTP 503 on every attempt (direct, `action=raw`, and `api.php`). The G-sign → tag mapping above is from project
> knowledge and Taginfo value counts, not a wiki quote. Ticket candidate (see §Open questions).

**GraphHopper**

- Custom model semantics, hard ban vs penalty, `areas`/`in_<id>`, `distance_influence`, `turn_penalty`:
  [docs/core/custom-models.md](https://github.com/graphhopper/graphhopper/blob/master/docs/core/custom-models.md).
  Hard ban = `{"if": "...", "multiply_by": "0"}` in `priority`. Built-in encoded values include `road_class`,
  `road_environment` (`FERRY`, `TUNNEL`…), `road_access`, `bike_road_access`, `foot_road_access`, `hgv_road_access`,
  `country`, `cycleway`, `max_speed` (full list in
  [`DefaultImportRegistry`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/ev/DefaultImportRegistry.java)).
- Restriction keys are hard-wired per `TransportationMode` in
  [`OSMRoadAccessParser.toOSMRestrictions`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/util/parsers/OSMRoadAccessParser.java):
  FOOT, VEHICLE, BIKE, CAR, MOTORCYCLE (`motorcycle, motor_vehicle, vehicle, access`), HGV, PSV, BUS, HOV; any other
  mode throws `IllegalArgumentException`. A GitHub code search for `moped` or `mofa` in the repo returns 0 hits.
- `bicycle=use_sidepath` is modelled: [`BikeRoadAccess`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/ev/BikeRoadAccess.java)
  has `USE_SIDEPATH`, and [`BikeCommonPriorityParser`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/util/parsers/BikeCommonPriorityParser.java)
  maps it to `REACH_DESTINATION` priority. This is the template for the moped equivalent.
- Extension point: `GraphHopper.setImportRegistry(ImportRegistry)` ([GraphHopper.java](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/GraphHopper.java),
  interface [`ImportRegistry.createImportUnit(String name)`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/ev/ImportRegistry.java)).
  New encoded values are then listed in `graph.encoded_values` ([config-example.yml](https://github.com/graphhopper/graphhopper/blob/master/config-example.yml))
  and usable in custom models and `details`.
- Oneway exceptions for cyclists (`oneway:bicycle=no`, `cycleway=opposite|opposite_lane|opposite_track`,
  `cycleway:left/right:oneway=no|-1`) are handled in
  [`BikeCommonAccessParser`](https://github.com/graphhopper/graphhopper/blob/master/core/src/main/java/com/graphhopper/routing/util/parsers/BikeCommonAccessParser.java).
  `motorway` and `motorroad=yes` are skipped for bikes there.
- Country defaults exist for NLD (foot: `BUSWAY|BRIDLEWAY → NO`, `CYCLEWAY → YES`; bike: `BUSWAY|BRIDLEWAY → NO`)
  in `OSMRoadAccessParser`; there is no NL `CountryRule`
  ([custom-areas-and-country-rules.md](https://github.com/graphhopper/graphhopper/blob/master/docs/core/custom-areas-and-country-rules.md)).
- `traffic_sign=*` is not imported by any parser; `custom_areas.directory` can add polygon-derived values at import.

**Valhalla**

- Tag transform [`lua/graph.lua`](https://github.com/valhalla/valhalla/blob/master/lua/graph.lua): the `moped`
  table maps `yes, designated, private, permissive, destination, delivery, dismount, permit, residents → true` and
  `no, unknown, agricultural → false`; **`use_sidepath` is absent** (falls through to `motor_vehicle`/highway
  default, i.e. allowed). `moped_tag = any_in(moped) or any_in(mofa) or motor_vehicle_access`, so `mofa` is only
  read when `moped` is missing and both feed the same `moped_forward/backward`. Directional `moped:forward`,
  `mofa:backward`, `oneway:moped`, `oneway:mofa` (incl. `-1`) are handled. Highway defaults: `motorway → moped
  false`, `trunk → moped true`, `cycleway|path|footway|pedestrian → moped false`, `living_street|residential → true`.
  `traffic_sign` never appears; `use_sidepath` is only in the bicycle/foot/access/cycleway tables.
- Access bits [`baldr/graphconstants.h`](https://github.com/valhalla/valhalla/blob/master/valhalla/baldr/graphconstants.h):
  `kMopedAccess = 512`, `kMotorcycleAccess = 1024`, `kAllAccess = 4095` (12 bits, all used), so a separate Snorfiets
  bit is a graph-format change.
- `motorroad=yes` → [`countryaccess.cc`](https://github.com/valhalla/valhalla/blob/master/src/mjolnir/countryaccess.cc)
  / [`graphenhancer.cc`](https://github.com/valhalla/valhalla/blob/master/src/mjolnir/graphenhancer.cc):
  "motorroad defaults remove ped, wheelchair, moped, and bike access" for road class ≤ primary. Correct for G3.
- Costing [`sif/motorscootercost.cc`](https://github.com/valhalla/valhalla/blob/master/src/sif/motorscootercost.cc):
  `DynamicCost(costing, TravelMode::kDrive, kMopedAccess)`; options `top_speed` (default 45, 20–120),
  `use_hills`, `use_primary` (0–1); road-class factors motorway 1.0 … residential 0.0; surfaces rougher than
  `kDirt` hard-excluded; ferries costed via `ferry_factor_`, not banned. Trunk/motorway are *penalised*, not
  banned, by costing; bans come only from access bits.
- Request-time hard exclusions: `exclude_highways` is **motorway-only**
  ([`sif/dynamiccost.h` `CheckExclusions`](https://github.com/valhalla/valhalla/blob/master/valhalla/sif/dynamiccost.h)),
  requires `service_limits.allow_hard_exclusions`; `exclude_polygons` (GeoJSON) and `exclude_ferries` exist
  ([API reference](https://github.com/valhalla/valhalla/blob/master/docs/docs/api/route/api-reference.md)).
- Lua is swappable without recompiling: `mjolnir.graph_lua_name`, "Location of the lua file to use for graph
  customization during tile building instead of default one"
  ([scripts/valhalla_build_config](https://github.com/valhalla/valhalla/blob/master/scripts/valhalla_build_config)).
  Still a full tile rebuild per change.
- Bicycle `use_sidepath` is a stored edge flag set only from `bicycle=use_sidepath`
  ([`pbfgraphparser.cc`](https://github.com/valhalla/valhalla/blob/master/src/mjolnir/pbfgraphparser.cc))
  and penalised in [`bicyclecost.cc`](https://github.com/valhalla/valhalla/blob/master/src/sif/bicyclecost.cc)
  (`sidepath_factor_ = 3.0 * (1 - use_roads)`); nothing equivalent for moped.

**BRouter**

- [`misc/profiles2/lookups.dat`](https://github.com/abrensch/brouter/blob/master/misc/profiles2/lookups.dat)
  (`---lookupversion:11`, `minorversion:2`) contains `moped;… no|yes|designated|use_sidepath`,
  `mofa;… no|yes|designated|use_sidepath`, `speed_pedelec;… yes|designated|use_sidepath|no`, `motorroad;… yes`,
  `oneway:bicycle;… no|yes`, `route;… ferry`. These entries are absent at commit `e66468b0` (2023-07-06) and
  present at `743f65b6` "new version 11.1 for lookups" (2025-08-26). `traffic_sign` exists only in the node context
  (`highway=traffic_sign`, `traffic_sign:direction`).
- [Issue #767](https://github.com/abrensch/brouter/issues/767) "Moped profile seems to ignore moped tagging"
  (closed 2026-04-22), maintainer: "the tag `moped` never made it into the `lookups.dat`. There is now an addition
  to the list for `moped` and `mofa`." Companion [issue #94](https://github.com/abrensch/brouter/issues/94).
- [`moped.brf`](https://github.com/abrensch/brouter/blob/master/misc/profiles2/moped.brf): header "Moped-Routing is
  experimental !!! DO NOT USE FOR ACTUAL NAVIGATION. Turn restrictions are missing"; access is derived from
  `motor_vehicle`/`motorcar`/`motorcycle` only; `grep -i moped` finds no tag use. `considerTurnRestrictions`
  exists in `trekking.brf`, so the "missing" is a profile choice, not an engine limit.
- Semantics: "Ways with costfactor >= 10000 are considered as if they did not exist at all"; costfactor must be ≥ 1
  ([profile_developers_guide.md](https://github.com/abrensch/brouter/blob/master/docs/developers/profile_developers_guide.md)).
  Profiles can only use keys present in `lookups.dat` at segment-generation time ("a profile key missing from the
  data sees empty values").

### 2. Turn-by-turn instructions

- GraphHopper `/route`: per instruction `text`, `street_name`, `distance`, `time`, `interval` (point indices),
  `sign` (−7 KEEP_LEFT … 7 KEEP_RIGHT, 6 USE_ROUNDABOUT with `exit_number`, `turn_angle`)
  ([docs/web/api-doc.md](https://github.com/graphhopper/graphhopper/blob/master/docs/web/api-doc.md)).
  In-tree [`navigation` module](https://github.com/graphhopper/graphhopper/tree/master/navigation):
  `@Path("navigate")` → `GET /navigate/directions/v5/gh/{profile}/{coordinates}` with `voice_instructions`,
  `banner_instructions` query params; "Provides an endpoint that is consumable with the Mapbox Navigation SDK …
  Mapbox API v5" ([NavigateResource.java](https://github.com/graphhopper/graphhopper/blob/master/navigation/src/main/java/com/graphhopper/navigation/NavigateResource.java)).
- Valhalla: `format` ∈ `json|gpx|osrm|pbf`; with `osrm`: `shape_format` (`polyline6` default, `geojson`),
  `banner_instructions`, `voice_instructions`; `directions_type` `none|maneuvers|instructions`; trip JSON maneuvers
  carry `instruction`, `verbal_transition_alert_instruction`, `verbal_pre_transition_instruction`,
  `verbal_post_transition_instruction`, `begin_shape_index`/`end_shape_index`, `travel_type: motor_scooter`,
  43 maneuver types; `nl-NL` in the language list
  ([API reference](https://github.com/valhalla/valhalla/blob/master/docs/docs/api/route/api-reference.md)).
- BRouter: `turnInstructionMode` 0–7 (`locus-style`, `osmand-style`, `comment-style`, `gpsies-style`,
  `orux-style`), `turnInstructionCatchingRange` 40 m, `turnInstructionRoundabouts`; hints derive from
  `priorityclassifier` in the profile
  ([profile_developers_guide.md](https://github.com/abrensch/brouter/blob/master/docs/developers/profile_developers_guide.md)).
  Output formats `gpx|kml|geojson|csv` via `/brouter?lonlats=…&profile=…&alternativeidx=…&format=…`
  ([http_server.md](https://github.com/abrensch/brouter/blob/master/docs/developers/http_server.md)).

### 3. Memory, disk, import, rebuild

Input: Geofabrik `netherlands-latest.osm.pbf` = **1 407 494 620 bytes (1.41 GB)**, dated 2026-10-20 (HTTP headers).

- GraphHopper ([docs/core/deploy.md](https://github.com/graphhopper/graphhopper/blob/master/docs/core/deploy.md)):
  planet base import "~60GB RAM and takes ~3h"; `MMAP` lowers RAM to ~31 GB at the cost of ~3 days; CH preparation
  ~120 GB RAM; serving `-Xmx` ≈ graph-cache size minus geometry. "Adding or modifying profiles is not possible and
  you need to run a new import instead"; CH/LM can be added to existing profiles later. Rebuild strategy on one small
  box: `import` into a new `graph.location`, then restart `server` on it (two directories, swap). No NL figure is
  published; by PBF share (1.7 % of planet) a single-digit-GB heap is the expectation. **Unverified; needs a
  measurement ticket.**
- Valhalla ([docker/README.md](https://github.com/valhalla/valhalla/blob/master/docker/README.md)): no RAM/disk
  numbers; `server_threads` "If valhalla gets killed when building tiles, lower this number"; tiles rebuild
  automatically on `docker restart` when PBF hashes change, `force_rebuild=True` forces it; pre-built
  `valhalla_tiles.tar` can be dropped in with `use_tiles_ignore_pbf=True`. No NL figure found in docs or issues.
  **Unverified.**
- BRouter: server start script `JAVA_OPTS="-Xmx128M -Xms128M -Xmn8M -DmaxRunningTime=300"`
  ([http_server.md](https://github.com/abrensch/brouter/blob/master/docs/developers/http_server.md)); NL lies in
  5°×5° segments `E0_N50.rd5` (79 469 208 B) and `E5_N50.rd5` (182 399 774 B) on
  [brouter.de/brouter/segments4](https://brouter.de/brouter/segments4/) (Last-Modified 2026-10-21). No import step
  when using public segments; a custom `lookups.dat` means regenerating segments yourself
  ([build_segments.md](https://github.com/abrensch/brouter/blob/master/docs/developers/build_segments.md)).

### 4. Alternatives, via points, re-calculation

- GraphHopper: `algorithm=alternative_route`, `alternative_route.max_paths` (2), `max_weight_factor` (1.4),
  `max_share_factor` (0.6); "works without `ch.disable=true`", i.e. under CH. Via points: repeated `point`,
  `pass_through`, `heading`/`heading_penalty`, `curbside`, `snap_prevention`
  ([api-doc.md](https://github.com/graphhopper/graphhopper/blob/master/docs/web/api-doc.md)). Modes: speed (CH)
  "very fast and lightweight (less RAM)"; hybrid (LM) "an order of magnitude faster than the flexible mode"
  ([README](https://github.com/graphhopper/graphhopper/blob/master/README.md)). Request-time `custom_model` only
  "with flex- and hybrid mode", hybrid limited to `multiply_by` ∈ [0, 1]
  ([profiles.md](https://github.com/graphhopper/graphhopper/blob/master/docs/core/profiles.md)). So per-Vehicle-class
  rules should live server-side and be CH-prepared; rider toggles (e.g. avoid ferries) go through LM.
- Valhalla: `alternates` "not yet supported on multipoint routes … nor time dependent routes"; locations
  `break|through|via|break_through`, `heading`, `heading_tolerance` 60°
  ([API reference](https://github.com/valhalla/valhalla/blob/master/docs/docs/api/route/api-reference.md)).
- BRouter: `alternativeidx` parameter; algorithm doc: two-pass search whose "processing time scales linear with
  distance", with an adaptive cutoff and a recalculation timeout
  ([algorithm.md](https://github.com/abrensch/brouter/blob/master/docs/developers/algorithm.md)).
- **Not verified:** absolute re-route latencies on a small instance for any engine; no primary benchmark exists.

### 5. Licence and containers

- GraphHopper: Apache-2.0 ([LICENSE.txt](https://github.com/graphhopper/graphhopper/blob/master/LICENSE.txt), GitHub
  SPDX). Compatible with ScooterPlanner's AGPL-3.0 (permissive licence; the combined work can be AGPL).
  Images: community [`israelhikingmap/graphhopper`](https://github.com/IsraelHikingMap/graphhopper-docker-image-push)
  (nightly `latest`, upstream version tags, amd64/arm64, default `JAVA_OPTS=-Xmx1g -Xms1g`; README: GraphHopper
  "chose not to publish a Docker image", maintainers looking for a successor). GraphHopper's repo has no Dockerfile.
- Valhalla: MIT ([COPYING](https://github.com/valhalla/valhalla/blob/master/COPYING)); official images
  `ghcr.io/valhalla/valhalla`, `ghcr.io/valhalla/valhalla-scripted`, `ghcr.io/valhalla/valhalla-dev`
  ([docker/README.md](https://github.com/valhalla/valhalla/blob/master/docker/README.md)); the former
  `gis-ops/docker-valhalla` is archived and moved upstream.
- BRouter: MIT ([LICENSE](https://github.com/abrensch/brouter/blob/master/LICENSE)); in-tree `Dockerfile`
  (Temurin 17), published by `.github/workflows/docker-publish.yml` to `ghcr.io/${{ github.repository }}`
  (= `ghcr.io/abrensch/brouter`).

## What the GraphHopper build looks like

1. `ImportRegistry` extension registering `moped_road_access` (keys `moped, motor_vehicle, vehicle, access`) and
   `mofa_road_access` (keys `mofa, moped, motor_vehicle, vehicle, access`), both mapping `use_sidepath`,
   `designated`, `no` etc. like `BikeRoadAccess`; optionally a boolean `motorroad`.
2. Three server-side profiles with `custom_model_files`: `fiets.json` (= upstream `bike.json`), `snorfiets.json`,
   `bromfiets.json`. Bans: `road_class == MOTORWAY || road_class == TRUNK` (or `motorroad`),
   `moped_road_access == NO`, `in_rijbaan_zone && road_class == CYCLEWAY` (Snorfiets). Penalties:
   `moped_road_access == USE_SIDEPATH` × 0.1, etc. `road_environment == FERRY` stays a penalty.
3. `profiles_ch` for all three; LM for the rider-toggle variants. `/navigate` for the SDK, `/route` for the map.

## Flags: not verified

- OSM wiki tagging guidance for NL G-signs (wiki returned 503 all day).
- RAM/disk/import time for a Netherlands graph on GraphHopper and Valhalla (no primary figures exist).
- Re-route latency numbers for any engine.
- BRouter `alternativeidx` value range (only `0` appears in docs).
- Whether the public brouter.de segments were already regenerated with lookups v11.x containing `moped` (the file
  date suggests yes, but the lookup version is not exposed over HTTP).

## Open questions that should become tickets

1. Confirm the G11/G12a/G13/G3 → `moped`/`mofa`/`motorroad`/`traffic_sign` convention from the OSM wiki and measure
   NL coverage with Overpass (how many `highway=cycleway` lack `moped=*`?).
2. Specify `moped_road_access` / `mofa_road_access` semantics: inheritance order, is `use_sidepath` a ban or a
   penalty per Vehicle class, how `speed_pedelec=*` is read.
3. Rijbaan zone polygons: data source (e.g. Amsterdam A10 ring) and shipping them as custom-model `areas`.
4. Benchmark the NL import on a small instance (heap, graph-cache size, import + CH time) and pick MMAP vs RAM.
5. Turn restrictions for mopeds: `turn_costs.vehicle_types` and OSM `restriction` relations with `except=moped`.
6. Navigation SDK that consumes Mapbox v5 JSON (open forks / MapLibre navigation), pairing with `/navigate`.
