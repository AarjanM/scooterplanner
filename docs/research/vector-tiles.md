# Research: self-hosted vector tiles for the Netherlands

Resolves [issue #6](https://github.com/AarjanM/scooterplanner/issues/6) (part of the wayfinder map, #1).
Researched 2026-10-21 against primary sources only (vendor docs, specs, source code, live tool output).
Vocabulary follows [GLOSSARY.md](../../GLOSSARY.md).

## TL;DR recommendation

1. **Serve PMTiles straight from Scaleway Object Storage** with HTTP range requests; no tile server
   process. Put Scaleway **Edge Services (Starter, EUR 0.99/month)** in front for a custom domain,
   TLS and an edge cache. Keep the single instance for the routing engine, not for tiles.
2. **Basemap**: `pmtiles extract` of the Netherlands from the daily Protomaps planet build.
   Measured today: **1.9 GB for z0-15 (158,737 tiles), 783 MB for z0-14** (dry run, see §2.1).
   Style it with `@protomaps/basemaps` flavors.
3. **Moped-infrastructure overlay**: neither the Protomaps nor the OpenMapTiles schema carries
   `moped`/`mofa`/`traffic_sign`, so Bromfietspad (G12a) vs Verplicht fietspad (G11) cannot be
   drawn from a standard basemap. Build a **separate small overlay tileset with a Planetiler
   YAML (custommap) profile** over the Geofabrik Netherlands extract (1.3 GB `.osm.pbf`),
   classifying `highway=cycleway` ways by `traffic_sign`/`moped`/`mofa` into
   `bromfietspad | verplicht_fietspad | onverplicht_fietspad | onbekend`. Draw it as its own
   MapLibre line layers above the basemap roads.
4. **Fonts and sprites**: copy `protomaps/basemaps-assets` (Noto Sans glyphs, OFL; sprites, MIT)
   into the same bucket as static files.
5. **Versioned, immutable file names** (`nl-basemap-20261020.pmtiles`), long `Cache-Control`
   on the archives, a tiny short-lived `style.json` that points at the current versions. Rebuild
   the overlay weekly and the basemap monthly from CI; never overwrite an archive in place.
6. **Egress**: 75 GB/month free, then EUR 0.01/GB. At the measured ~12 KB average tile, 75 GB is
   roughly 60,000 map sessions of 100 tiles. Costs stay in single-digit euros per month well
   past the first thousand users.

Everything marked **[unverified]** below needs a hands-on test before an ADR is written.

---

## 1. PMTiles from Object Storage vs a tile server

### 1.1 What PMTiles is and what it costs per tile

- PMTiles is a single-file archive read remotely over HTTP range requests; the docs contrast it
  with MBTiles, which "is meant to be read from disk by a running server process". A tile lookup
  needs "at most two cacheable intermediate requests" (header+root directory, optionally one leaf
  directory) before the tile bytes themselves. The archive is read-only: "If you need
  transactional updates, use a database like SQLite or PostgreSQL."
  Source: https://docs.protomaps.com/pmtiles/
- Spec v3: 127-byte header; the root directory "MUST be contained in the first 16,384 bytes
  (16 KiB)"; leaf directories are optional and the spec discourages more than one level; tiles
  are Hilbert-ordered and the archive should be `clustered` so tile data is contiguous. Internal
  and tile compression may be gzip/brotli/zstd.
  Source: https://github.com/protomaps/PMTiles/blob/main/spec/v3/spec.md
- The live Protomaps planet build `20261020.pmtiles` reports spec v3, mvt, z0-15, clustered,
  gzip internal and tile compression, 178,326,964 tile entries, built by Planetiler 0.10.2 from
  OSM replication `2026-10-20T04:00:00Z` (`pmtiles show`, run 2026-10-21).

### 1.2 Serving directly from S3-compatible storage

- "designed to work on any S3-compatible cloud storage platform that supports HTTP Range
  Requests." CORS is mandatory when the page/app origin differs from the bucket: allow `GET`
  and `HEAD`, allow request headers `range` and `if-match`, expose `etag`. "Each `Range` tile
  request counts as a GET." The provider list notes **"Scaleway supports only HTTP/1.1"**
  (R2/GCS support HTTP/2; S3, Azure, DigitalOcean, B2 are HTTP/1.1 too).
  Source: https://docs.protomaps.com/pmtiles/cloud-storage
- HTTP/1.1 means browsers cap parallel tile requests at ~6 per host (standard browser
  behaviour, not from a fetched source); MapLibre already throttles tile fetches so in
  practice this matters most on first load. **[unverified on Scaleway specifically]**
- Scaleway Object Storage docs: the `GetObject` sample response carries `Accept-Ranges: bytes`
  but the page does not document the `Range` header or `206 Partial Content` explicitly.
  Source: https://www.scaleway.com/en/docs/object-storage/api-cli/object-operations/
  Protomaps listing Scaleway as a working (HTTP/1.1) provider is the stronger signal that range
  requests work. **[unverified: run `curl -r 0-16383 -I` against a real bucket]**
- CORS on Scaleway is set with the AWS CLI: `aws s3api put-bucket-cors --bucket NAME
  --cors-configuration file://cors.json`; the docs' example uses `AllowedHeaders: ["*"]` and
  `ExposeHeaders: ["Etag"]`; requires the `ObjectStorageBucketsWrite` or `ObjectStorageFullAccess`
  IAM permission.
  Source: https://www.scaleway.com/en/docs/object-storage/api-cli/setting-cors-rules/
- Scaleway Object Storage pricing (Paris, before tax): Standard Multi-AZ EUR 0.000022/GB/hour
  (~EUR 0.016/GB/month), Standard One Zone EUR 0.000011/GB/hour (~EUR 0.008/GB/month);
  **requests included; ingress included; egress "75GB free every month then EUR 0.01/GB"**.
  Source: https://www.scaleway.com/en/pricing/storage/
  The 1.9 GB basemap + a few hundred MB of overlay/fonts therefore costs about EUR 0.02-0.04
  per month to store.
- Scaleway Edge Services is "a feature that allows you to create a cache and a custom domain for
  your buckets" (Object Storage concepts page). Cache behaviour: you set a **Lifetime**; an
  object's own caching directive "always takes precedence over any lifetime setting"; purge is
  either the whole cache or up to 5 named paths in the console (the API bypasses the 5-path
  limit); directories cannot be purged.
  Sources: https://www.scaleway.com/en/docs/object-storage/concepts/ ,
  https://www.scaleway.com/en/docs/edge-services/how-to/configure-cache/
  Pricing: Starter EUR 0.99/month (1 pipeline, 100 GB cache egress), Professional EUR 12.99
  (10 pipelines, 1 TB), Advanced EUR 109.99 (100 pipelines, 10 TB); extra cache egress
  EUR 0.0135/GB; extra pipeline EUR 4. "Cache" is defined as the "sum of cached data fetched from
  Scaleway cache infrastructure".
  Source: https://www.scaleway.com/en/pricing/network/
  **[unverified: whether Edge Services caches `206` range responses; the docs do not say. If it
  does not, Edge Services still gives TLS + custom domain but no egress saving.]**

### 1.3 Tile servers on the instance

All three below can read the same `.pmtiles` from the bucket, so choosing PMTiles-in-a-bucket
does not close the door on a server later.

| Server | Reads PMTiles from | Fonts / sprites | Caching | Source |
|---|---|---|---|---|
| `pmtiles serve` (go-pmtiles) | local dir, `--bucket=s3://...` or HTTP URL; serves Z/X/Y + TileJSON; `--public-url` required for TileJSON | no | `--cache-size` (default 64 MB) caches **headers and directories only**, not tile bytes; `--cors=` is "suited to development" | https://docs.protomaps.com/pmtiles/cli |
| Caddy + `pmtiles_proxy` | `bucket` URL | no | `cache_size` covers "intermediate PMTiles headers and directories" only; auto-HTTPS | https://docs.protomaps.com/deploy/server |
| Martin (Rust) | local path, `file://`, watched dirs, `http(s)://` single files, `s3://bucket/path` with S3-compatible `endpoint`; "Remote files are read with byte-range requests"; remote prefixes re-listed every `10m` | glyphs generated on the fly from `otf/ttf/ttc` at `/font/{name}/{start}-{end}` with fallback stacks; sprites generated from SVG dirs at `/sprite/{id}.json|.png|@2x`, SDF variant | per-source `cache_control`; font and sprite caches (`size_mb`) | https://maplibre.org/martin/sources-pmtiles/ , https://maplibre.org/martin/sources-fonts/ , https://maplibre.org/martin/sources-sprites/ |
| tileserver-gl (Node) | `--file` accepts "MBTiles or PMTiles file (local path, http(s)://, s3://, pmtiles://, or mbtiles:// URL)"; `--fetch-timeout` default 15000 ms, a timed-out remote tile "is treated as missing" | serves fonts from `paths.fonts` (`serveAllFonts`), sprites local or remote | not documented | https://tileserver.readthedocs.io/en/latest/usage.html , https://tileserver.readthedocs.io/en/latest/config.html |

Protomaps' own deployment comparison: "Static PMTiles: No Z/X/Y support, no edge caching, but
scales to zero"; Caddy and `pmtiles serve` give Z/X/Y "with CDN-based caching and SSL via a
reverse proxy. They don't scale to zero." Edge cache hits "can respond in 100 ms or less, while a
single-region bucket may take multiple seconds for distant users" (irrelevant for a NL-only
audience served from `nl-ams`/`fr-par`).
Source: https://docs.protomaps.com/deploy/

Instance cost for comparison (PAR-1, before tax; "List prices include egress and IPv6"):
STARDUST1-S 1 vCPU/1 GB/100 Mbps EUR 0.0006/h (~EUR 0.43/month); DEV1-S 2 vCPU/2 GB/200 Mbps
EUR 0.00898/h (~EUR 6.55/month); DEV1-M 3 vCPU/4 GB/300 Mbps ~EUR 14.74/month.
Source: https://www.scaleway.com/en/pricing/virtual-instances/
Instance egress is thus flat-rate but bandwidth-capped; bucket egress is metered but
uncapped.

### 1.4 Client support for `pmtiles://`

- MapLibre GL JS: `npm install pmtiles`; `const protocol = new Protocol(); maplibregl.addProtocol("pmtiles", protocol.tile);`
  then a vector source with `"url": "pmtiles://https://example.com/example.pmtiles"`. "Using the
  `pmtiles://` protocol will automatically derive a `minzoom` and `maxzoom` for your `Source`."
  `addProtocol` "works best if it is only called once".
  Source: https://docs.protomaps.com/pmtiles/maplibre
- MapLibre Native: Android 11.8.0 release notes "Add PMTiles support (#2882)"; iOS 6.10.0 "Add
  support for PMTiles with `pmtiles://` URL scheme (#2882)".
  Sources: https://github.com/maplibre/maplibre-native/releases/tag/android-v11.8.0 ,
  https://github.com/maplibre/maplibre-native/releases/tag/ios-v6.10.0
- MapLibre React Native 11.5.0 (2026-10-04) pins native Android 13.6.1
  (`package/android/gradle.properties`: `org.maplibre.reactnative.nativeVersion=13.6.1`) and iOS
  6.31.0 (`package/MapLibreReactNative.podspec`: `$MLRN_NATIVE_VERSION ||= "6.31.0"`), both
  above the versions that added PMTiles. Its example app has `PMTilesVectorSource.tsx` using
  `<VectorSource url="pmtiles://https://..."/>`; the changelog records a fix for "usage of
  PMTiles on iOS 17 through VectorSource with NSString (#625)".
  Source: https://github.com/maplibre/maplibre-react-native (files cited above, read 2026-10-21)
- So **no tile server is needed for either web or React Native**. A Z/X/Y endpoint
  (`pmtiles serve`/Martin) is only needed for third-party clients that cannot register a
  protocol. **[unverified: MapLibre Native's on-device tile cache behaviour for range responses;
  matters for data use during Turn-by-turn navigation.]**

### 1.5 Verdict for §1

Direct-from-bucket wins on cost (requests free, storage cents, scale-to-zero), on operations
(nothing to keep running or patch) and on the budget constraint (one instance stays free for
routing). A server only pays off if (a) Z/X/Y URLs are needed, (b) fonts/sprites must be
generated dynamically (Martin), or (c) measured latency of HTTP/1.1 range requests from Scaleway
turns out poor. Keep Martin as the named fallback: it reads the identical files from the bucket.

---

## 2. Generating a Netherlands extract

### 2.1 Option A: `pmtiles extract` from the Protomaps daily build (recommended for the basemap)

- Daily planet builds (~120 GB, z0-15, "Version 4 channel") at maps.protomaps.com/builds; the
  bucket keeps a week of builds and the latest per patch version, with BLAKE3 hashes. Licence:
  ODbL Produced Work, OSM attribution required. Each extra zoom level roughly doubles size.
  Source: https://docs.protomaps.com/basemaps/downloads
- Build pipeline is `protomaps/basemaps` (Java/Planetiler profile; `java -jar ... --download
  --area=monaco`, or Docker image `protomaps/basemaps`). The daily build's metadata confirms
  `planetiler:version 0.10.2`.
  Source: https://github.com/protomaps/basemaps/blob/main/tiles/README.md ; `pmtiles show`
- `pmtiles extract <input> <output>` takes `--bbox`, `--region` (GeoJSON Polygon/MultiPolygon),
  `--minzoom`, `--maxzoom`, `--download-threads` (default 4), `--overfetch` (default 0.05),
  `--dry-run`. Full sub-pyramid extraction "makes minimal I/O or network requests to the source
  archive"; `--minzoom` partial extracts "may require many more requests".
  Source: https://docs.protomaps.com/pmtiles/cli ; `pmtiles extract --help` (go-pmtiles 1.31.2)
- **Measured 2026-10-21** with go-pmtiles 1.31.2 against `https://build.protomaps.com/20261020.pmtiles`,
  bbox `3.31,50.75,7.23,53.60` (whole NL incl. Wadden, some BE/DE border spill):

  | Extract | Tile entries | Archive size | Requests to source | Transfer |
  |---|---|---|---|---|
  | z0-15 (full) | 158,737 | **1.9 GB** | 65 | 2.0 GB |
  | z0-14 | 41,895 | **783 MB** | 60 | 822 MB |

  Dry run only; wall-clock time depends on download bandwidth (the tile planning phase itself
  took ~5 s). A `--region` polygon of the national border will be somewhat smaller than the
  bbox. **[unverified: actual download time; polygon extract size]**
- Build time on our side: none. Memory: none. This is why it is the recommended basemap route.

### 2.2 Option B: Planetiler, own build (needed for the overlay; optional for the basemap)

- Planetiler quick start: `planetiler --download --area=monaco --output=monaco.pmtiles`
  (`--area=` names a Geofabrik extract, so `--area=netherlands`). Output format follows the
  `--output` extension (`.mbtiles` default, `.pmtiles` supported); MVT default, MLT via
  `--tile-format=mlt`. Jar needs **Java 25+**; the single-file launcher bundles its own runtime.
  Hardware: "At least 1GB of free SSD space, plus 5-10x the size of the `.osm.pbf` file";
  "At least 0.5x the `.osm.pbf` size in free RAM"; give the JVM ~0.5x the pbf size and leave the
  rest for memory-mapped files.
  Source: https://github.com/onthegomap/planetiler/blob/main/README.md
- Planet benchmarks (OpenMapTiles profile) from the same README, for scale: planet (92 GB pbf)
  on 192 cpu/720 GB in 19 m; on 64 cpu/128 GB in 42 m (73 GB pbf); with
  `--building-merge-z13=false` a 2 cpu/8 GB machine did the planet in 18 h 18 m; 8 cpu/16 GB in
  3 h 35 m. **No country-sized timing is documented.**
- PLANET.md adds: RAM ≥ 1.5x pbf → `--nodemap-type=array --storage=ram`; otherwise
  `--storage=mmap`; disk ~10x input.
  Source: https://github.com/onthegomap/planetiler/blob/main/PLANET.md
- Geofabrik `netherlands-latest.osm.pbf` is **1.3 GB**, "normally updated every day", ODbL.
  Sources: https://download.geofabrik.de/europe/netherlands.html , https://download.geofabrik.de/
- Applying the README rules to NL: ≥ 0.65 GB RAM for the JVM (so a 2 GB DEV1-S is plausible,
  4 GB comfortable), 7-14 GB free disk. **[unverified: real NL build time/RAM; expect minutes,
  not hours, on a laptop. This should be measured once, see follow-ups.]**
- Full-basemap profiles you could run yourself:
  - `openmaptiles/planetiler-openmaptiles`: `docker run --rm -v "$(pwd)/data":/data
    openmaptiles/planetiler-openmaptiles:latest --force --download --area=monaco`; Java 25+;
    `--only-layers=`/`--exclude-layers=` to trim. No size/time figures given.
    Source: https://github.com/openmaptiles/planetiler-openmaptiles/blob/main/README.md
  - `protomaps/basemaps` (see §2.1) for an identical schema to the daily build.
- Custom YAML profile (`planetiler-custommap`): schema files are YAML 1.2 with `sources`
  (`type: osm`, `url: geofabrik:<area>`), `layers[].features[]` with `source`, `geometry: line`,
  `include_when` / `exclude_when` (tag maps; lists are alternatives; `%` wildcard; `__any__`;
  `__all__`/`__not__` for boolean logic), `min_zoom` (may be an expression), and `attributes[]`
  with `key` + `tag_value` (optionally `type`, per-attribute `include_when`). Run with
  `planetiler schema.yml --download --output=out.pmtiles` or
  `java -jar planetiler.jar generate-custom --schema=schema.yml`; `verify schema.yml` runs the
  schema's embedded test cases.
  Sources: https://github.com/onthegomap/planetiler/blob/main/planetiler-custommap/README.md ,
  https://github.com/onthegomap/planetiler/blob/main/README.md

### 2.3 Option C: classic OpenMapTiles (PostGIS + imposm)

Not examined in depth: it needs PostgreSQL/PostGIS and a multi-container pipeline, which does not
fit one small instance, and the OpenMapTiles organisation itself publishes the Planetiler profile
above. **[unverified beyond the existence of `openmaptiles/planetiler-openmaptiles`]**

### 2.4 Verdict for §2

Basemap via `pmtiles extract` (zero build cost, daily freshness available). Overlay via a
Planetiler custommap profile on the Geofabrik NL extract (small, fast, fully under our control,
and the same tool chain as the basemap should we ever need to self-build it).

---

## 3. Making Bromfietspad vs Verplicht fietspad visible

### 3.1 What the standard schemas carry

- **OpenMapTiles `transportation`**: `class` includes `path`, `subclass` includes `cycleway`,
  `path`, `footway`; per-mode fields are only `bicycle`, `foot`, `horse`, `mtb_scale`
  ("Original value of the ... tag (highways only)"), plus `access` (only `no`/`private`),
  `surface`, `oneway`, `brunnel`, `toll`, `expressway`, `layer`, `level`, `indoor`, `official`.
  **No `moped` or `mofa` field.**
  Source: https://openmaptiles.org/schema/#transportation ; the layer SQL selects exactly
  `class, subclass, network, oneway, ramp, brunnel, service, access, toll, expressway, layer,
  level, indoor, bicycle, foot, horse, mtb_scale, official, surface`
  (https://github.com/openmaptiles/openmaptiles/blob/master/layers/transportation/transportation.yaml);
  the Planetiler port sets `Fields.BICYCLE`, `FOOT`, `ACCESS` from zoom 9
  (https://github.com/openmaptiles/planetiler-openmaptiles/blob/main/src/main/java/org/openmaptiles/layers/Transportation.java).
- **Protomaps `roads`**: `kind` ∈ highway, major_road, minor_road, **path**, aerialway, ferry,
  pier, rail, aeroway; `kind_detail` includes **cycleway**, path, footway, sidewalk, crossing...
  (https://docs.protomaps.com/basemaps/layers). In `Roads.java` `highway=cycleway` maps to
  `pm:kind=path` and, having no explicit kindDetail, falls back to `kind_detail = highway value`
  (`feat.setAttr("kind_detail", highway)`), so **`kind=path`, `kind_detail=cycleway`** is emitted
  and `access` only from zoom 15
  (https://github.com/protomaps/basemaps/blob/main/tiles/src/main/java/com/protomaps/basemap/layers/Roads.java).
  The live build's `roads` layer fields (from `pmtiles show --metadata`, 2026-10-21) are
  `access, is_bridge, is_link, is_tunnel, kind, kind_detail, min_zoom, name*, network*, oneway,
  ref, route, service, shield_text*, sort_rank` — **no `bicycle`, `moped` or `mofa`**. The
  default styles treat `kind` path and other together and have no cycleway-specific rule
  (`styles/src/base_layers.ts`, filters on `["in", "kind", "other", "path"]`).
- Conclusion: **a cycle path is drawable, its legal class is not.** The G11/G12a/G13 distinction
  must come from the OSM tags directly.

### 3.2 How the distinction is tagged in OSM

From the Dutch tagging guideline (Wayback snapshot 2026-07-21 of
https://wiki.openstreetmap.org/wiki/NL:The_Netherlands_roads_tagging ; the live wiki returned
HTTP 503 throughout this research):

| Glossary term | Sign | OSM tags (quoted) |
|---|---|---|
| Verplicht fietspad | G11 | `highway=cycleway traffic_sign=NL:G11 mofa=yes moped=no` |
| Bromfietspad | G12a | `highway=cycleway traffic_sign=NL:G12a mofa=designated moped=designated` |
| Onverplicht fietspad | G13 | `highway=cycleway traffic_sign=NL:G13 mofa=no moped=no` |
| Bromfiets on/off ramp | D103 / D104 | `highway=cycleway moped=designated traffic_sign=NL:D103` (resp. `NL:D104`) |

- `Key:moped` (snapshot 2026-09-10): "legal access restriction for mopeds. A moped is a low
  powered motorcycle typically restricted to 45 km/h"; Class B, licence plate required — i.e.
  the Bromfiets / Speed pedelec class.
  https://wiki.openstreetmap.org/wiki/Key:moped
- `Key:mofa` (snapshot 2026-10-02): "maximum design speed of around 25 km/h ... In Dutch mofas
  would be translated as 'snorfietsen'"; Netherlands section: snorfietsen "follow largely the same
  access restrictions (including compulsory use of cycleways) as bicycles" and "are banned from
  the majority of cyclepaths within the ringway A10 in Amsterdam" (the Rijbaan zone).
  https://wiki.openstreetmap.org/wiki/Key:mofa
- Fallback defaults for NL (snapshot 2026-08-25 of
  https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing/Access_restrictions): for
  `highway=cycleway` → `moped=dismount`, `mofa=yes`, `bicycle=designated`. The page stresses these
  "are NOT the legal default access restrictions when a road in the real world is not
  signposted" but fallbacks for data users. So an untagged cycleway should be rendered as
  **unknown**, not as a Verplicht fietspad.
- Tag usage (taginfo API, global counts, 2026-10-21): `traffic_sign=NL:G11` 76,303 ways;
  `NL:G12a` 65,347 ways; `NL:G13` 21,118 ways; key `moped` on 329,171 ways (values: `no` 48.8 %,
  `designated` 21.5 %, `yes` 15.1 %, `use_sidepath` 13.9 %); key `mofa` on 334,204 ways.
  Only ~8 % of `highway=cycleway` ways worldwide carry `moped`, so **coverage inside NL must be
  checked** before relying on it. Source: https://taginfo.openstreetmap.org/api/4/
  **[unverified: NL-only coverage; taginfo is not country-filtered]**

### 3.3 Overlay design (Planetiler custommap)

One layer, `fietspad`, line geometry, from `highway=cycleway` (optionally also
`highway=path` + `bicycle=designated`), min_zoom ~11, attributes:

- `kind`: derived in the style or, better, in the profile with ordered `include_when` rules:
  1. `traffic_sign` matches `%G12a%` **or** `moped` ∈ {`designated`, `yes`} → `bromfietspad`
  2. `traffic_sign` matches `%G11%` **or** (`moped=no` and `mofa` ∈ {`yes`, `designated`}) →
     `verplicht_fietspad`
  3. `traffic_sign` matches `%G13%` **or** (`moped=no` and `mofa=no`) → `onverplicht_fietspad`
  4. otherwise → `onbekend` (render distinctly so data gaps are visible to mappers)
- pass-through: `traffic_sign`, `moped`, `mofa`, `bicycle`, `oneway`, `name`, `surface`.
- `traffic_sign` is often a `;`-separated list, hence the `%` wildcards (custommap supports `%`).
  Equivalent, more precise logic can be written in a Java `Profile` if YAML gets awkward
  (`./planetiler Roads.java --download --output=roads.pmtiles`, README).
- Expected size: a fraction of the 783 MB z14 basemap because only cycleways are included.
  **[unverified: build it and measure]**

Alternative (not recommended): fork `protomaps/basemaps` to add `moped`/`mofa` to `roads`. It
couples us to a Java build of the whole basemap for a single attribute and loses the free daily
planet builds.

### 3.4 Styling in MapLibre

Style spec essentials (https://maplibre.org/maplibre-style-spec/layers/ and `/root/`):
`source-layer` is required for vector sources; `filter` "Only features that match the filter are
displayed"; `line-color`, `line-width`, `line-gap-width` (casing), `line-offset`,
`line-dasharray` ("lengths of the alternating dashes and gaps", in line-width units, must be a
`["literal", [...]]`), `line-pattern` ("Name of image in sprite", power-of-two width, disables
color/dasharray). Example:

```json
{
  "sources": {
    "basemap": { "type": "vector", "url": "pmtiles://https://tiles.example.nl/nl-basemap-20261020.pmtiles" },
    "fietspad": { "type": "vector", "url": "pmtiles://https://tiles.example.nl/nl-fietspad-20261020.pmtiles" }
  },
  "layers": [
    { "id": "bromfietspad", "type": "line", "source": "fietspad", "source-layer": "fietspad",
      "filter": ["==", ["get", "kind"], "bromfietspad"],
      "paint": { "line-color": "#d97706", "line-width": 2.5 } },
    { "id": "verplicht-fietspad", "type": "line", "source": "fietspad", "source-layer": "fietspad",
      "filter": ["==", ["get", "kind"], "verplicht_fietspad"],
      "paint": { "line-color": "#2563eb", "line-width": 2, "line-dasharray": ["literal", [2, 1]] } }
  ]
}
```

Base layers come from `layers("basemap", namedFlavor("light"), { lang: "nl" })` of
`@protomaps/basemaps` (https://docs.protomaps.com/basemaps/maplibre); the overlay layers are
appended after the basemap `roads` layers so they draw on top. Colour/dash choices above are
placeholders for a design ticket.

---

## 4. Fonts and sprites

- Style spec: `glyphs` is "A URL template for loading signed-distance-field glyph sets in PBF
  format" with `{fontstack}` and `{range}` (256 code points per range), must be absolute;
  `sprite` is a URL string or an array of `{id, url}`; "All the URLs are internally extended to
  load both .json and .png files". https://maplibre.org/maplibre-style-spec/root/
- Protomaps assets: `https://protomaps.github.io/basemaps-assets/fonts/{fontstack}/{range}.pbf`
  and `https://protomaps.github.io/basemaps-assets/sprites/v4/light`; fonts are Noto Sans
  Regular/Medium/Italic generated with maplibre `font-maker` (SIL OFL); sprites light/dark at 1x
  and 2x (MIT-derived); "For self-hosting or offline use, the glyph and sprite assets can be
  downloaded as ZIP files".
  Sources: https://docs.protomaps.com/basemaps/maplibre , https://github.com/protomaps/basemaps-assets
- Alternatives: `openmaptiles/fonts` (Noto Sans, Open Sans, PT Sans, Roboto, Metropolis via
  fontnik, OFL/Apache; https://github.com/openmaptiles/fonts); `maplibre/font-maker` web app or
  CLI to convert any TTF/OTF (https://github.com/maplibre/font-maker). Martin can generate
  glyphs/sprites on the fly (§1.3) if a server is ever introduced.
- Recommendation: upload the `basemaps-assets` `fonts/` and `sprites/v4/` trees to the tile
  bucket under a versioned prefix (`assets/v4/...`), `Cache-Control: public, max-age=31536000,
  immutable`. Add a small custom sprite (G11/G12a sign icons) produced with the same tool
  Protomaps uses (`spreet`, per the basemaps-assets README) and reference it as a second sprite
  `{ "id": "sp", "url": ".../assets/sp/sprite" }`. GitHub Pages hosting is fine for prototyping
  but is a third party on the request path, so not for production.

---

## 5. Update cadence, cache headers, egress

### 5.1 Cadence

- Upstream freshness: Geofabrik NL "normally updated every day"; Protomaps publishes daily and
  keeps a week of builds; the archive metadata exposes `planetiler:osm:osmosisreplicationtime`.
- Proposed: **overlay weekly** (it carries the legally relevant tags and benefits from mapper
  fixes quickly; it is small), **basemap monthly** (large, cosmetic). Both from a scheduled CI
  job: `pmtiles extract --region nl.geojson`, `planetiler fietspad.yml --download
  --output=...`, `pmtiles verify`, then `pmtiles upload --bucket=s3://...` or `aws s3 cp`, then
  publish a new `style.json`. The daily cadence Protomaps offers is not needed for a basemap.
  **[unverified: GitHub Actions runner disk/time for the 1.9 GB extract; the 783 MB z14 variant
  is the safer first target]**

### 5.2 Cache headers and versioning

- Protomaps' own Cloudflare deployment defaults `CACHE_CONTROL` to `public, max-age=86400`
  (1 day) and warns that purge is zone-wide. https://docs.protomaps.com/deploy/cloudflare
- A PMTiles archive is fetched by byte ranges, so **replacing a file in place while clients hold
  cached directories would mix bytes of two archives**. The required CORS config exposes `etag`
  and allows `if-match` precisely so clients can detect this (https://docs.protomaps.com/pmtiles/cloud-storage).
  **[unverified: exact client behaviour on ETag mismatch]** Avoid the problem entirely:
  - archives: date-stamped names, `Cache-Control: public, max-age=31536000, immutable`;
  - `style.json` (and TileJSON if any): `Cache-Control: public, max-age=300`; it is the only
    mutable object and the only one that needs an Edge Services purge;
  - fonts/sprites: versioned prefix, immutable;
  - keep the previous archive for a week so running app sessions finish cleanly.
- On Scaleway the object's own `Cache-Control` metadata governs Edge Services ("always takes
  precedence over any lifetime setting"), so set it at upload time.

### 5.3 Egress estimate

- Measured average compressed tile: 1.9 GB / 158,737 ≈ **12 KB** (z0-15 NL extract). A map
  session that loads ~100 tiles ≈ 1.2 MB plus ≤ 32 KB of header/directories.
- Object Storage: 75 GB free ≈ **60,000 such sessions per month at zero egress cost**; beyond
  that EUR 0.01/GB, so 1 TB ≈ EUR 10. Requests are free, which matters because PMTiles makes one
  GET per tile.
- Edge Services Starter adds EUR 0.99/month for 100 GB of cache egress and EUR 0.0135/GB after,
  slightly dearer per GB than raw bucket egress; its value is TLS + custom domain + origin
  offload, not cost. Professional (1 TB) at EUR 12.99 is the next step if traffic grows.
- Serving from the instance instead would make egress flat (included in the instance price) but
  capped by its link (200 Mbps on DEV1-S) and would compete with the routing engine.
- Mobile: if MapLibre Native caches tiles on device, repeat rides over the same area cost nothing;
  **[unverified, see §1.4]**.

---

## 6. Recommendation (consolidated)

| Concern | Decision | Why |
|---|---|---|
| Delivery | PMTiles in Scaleway Object Storage, Edge Services Starter in front; no tile server | requests free, ~EUR 1-2/month total, nothing to operate; web and RN both speak `pmtiles://` |
| Basemap | `pmtiles extract` of NL from daily Protomaps build, z0-15 (1.9 GB) or z0-14 (783 MB) | zero build infrastructure, measured sizes, Protomaps styles ready-made |
| Moped overlay | Planetiler custommap profile over Geofabrik NL → `fietspad` layer with `kind` ∈ bromfietspad / verplicht_fietspad / onverplicht_fietspad / onbekend | standard schemas drop `moped`/`mofa`/`traffic_sign`; overlay is small and independently refreshable |
| Fonts/sprites | copy `basemaps-assets` into the bucket, versioned + immutable | OFL/MIT, same origin, no third party on the path |
| Cadence | overlay weekly, basemap monthly, from CI | matches upstream daily availability without churn |
| Caching | versioned archives, `immutable`; short-lived `style.json` | avoids mixed-archive range reads; single purge point |
| Fallback | Martin on the instance reading the same bucket | only if Z/X/Y or dynamic fonts become necessary |

Reasons against the alternatives: a tile server adds an always-on process to the one instance
and solves no problem we have; a forked Protomaps profile ties a one-attribute need to a full
basemap build; OpenMapTiles classic needs PostGIS.

## 7. Open points to turn into tickets

1. Build the overlay once and measure: Planetiler time/RAM/output size for NL on a laptop and on
   a DEV1-S; check NL coverage of `moped`/`mofa`/`traffic_sign` on cycleways (taginfo is global).
2. Verify Scaleway end to end with a throwaway bucket: range requests (`206`), CORS preflight,
   Edge Services caching of range responses, HTTP/1.1 parallelism effect on first paint.
3. Fix the overlay classification rules (and `onbekend` handling) together with the routing-rule
   work in #14; decide whether D103/D104 ramps and the Amsterdam Rijbaan zone need their own
   kinds.
4. MapLibre Native on-device caching of PMTiles during Turn-by-turn navigation (data usage,
   offline tolerance).
5. Visual design of Bromfietspad vs Verplicht fietspad vs Onverplicht fietspad (colour, dash,
   sign icons) and colour-blind check.
6. Re-check the three OSM wiki pages against the live wiki once it is reachable again (all
   quotes above are from Wayback snapshots dated 2026-07-21, 2026-08-25, 2026-09-10, 2026-10-02).

## Sources (all accessed 2026-10-21)

- Protomaps: https://docs.protomaps.com/pmtiles/ , https://docs.protomaps.com/pmtiles/cloud-storage ,
  https://docs.protomaps.com/pmtiles/cli , https://docs.protomaps.com/pmtiles/maplibre ,
  https://docs.protomaps.com/basemaps/downloads , https://docs.protomaps.com/basemaps/layers ,
  https://docs.protomaps.com/basemaps/maplibre , https://docs.protomaps.com/deploy/ ,
  https://docs.protomaps.com/deploy/server , https://docs.protomaps.com/deploy/cloudflare ,
  https://docs.protomaps.com/guide/getting-started ,
  https://github.com/protomaps/PMTiles/blob/main/spec/v3/spec.md ,
  https://github.com/protomaps/basemaps/blob/main/tiles/README.md ,
  https://github.com/protomaps/basemaps/blob/main/tiles/src/main/java/com/protomaps/basemap/layers/Roads.java ,
  https://github.com/protomaps/basemaps/blob/main/styles/src/base_layers.ts ,
  https://github.com/protomaps/basemaps-assets , https://github.com/protomaps/go-pmtiles (v1.31.2 binary)
- Planetiler: https://github.com/onthegomap/planetiler/blob/main/README.md ,
  https://github.com/onthegomap/planetiler/blob/main/PLANET.md ,
  https://github.com/onthegomap/planetiler/blob/main/planetiler-custommap/README.md ,
  https://github.com/openmaptiles/planetiler-openmaptiles/blob/main/README.md ,
  https://github.com/openmaptiles/planetiler-openmaptiles/blob/main/src/main/java/org/openmaptiles/layers/Transportation.java
- OpenMapTiles: https://openmaptiles.org/schema/ ,
  https://github.com/openmaptiles/openmaptiles/blob/master/layers/transportation/transportation.yaml ,
  https://github.com/openmaptiles/fonts
- Tile servers: https://maplibre.org/martin/ , https://maplibre.org/martin/sources-pmtiles/ ,
  https://maplibre.org/martin/sources-fonts/ , https://maplibre.org/martin/sources-sprites/ ,
  https://tileserver.readthedocs.io/en/latest/usage.html , https://tileserver.readthedocs.io/en/latest/config.html
- MapLibre: https://maplibre.org/maplibre-style-spec/root/ , https://maplibre.org/maplibre-style-spec/layers/ ,
  https://github.com/maplibre/maplibre-native/releases/tag/android-v11.8.0 ,
  https://github.com/maplibre/maplibre-native/releases/tag/ios-v6.10.0 ,
  https://github.com/maplibre/maplibre-react-native (v11.5.0: `package/android/gradle.properties`,
  `package/MapLibreReactNative.podspec`, `examples/shared/src/examples/protocols/PMTilesVectorSource.tsx`, `CHANGELOG.md`),
  https://github.com/maplibre/font-maker
- Scaleway: https://www.scaleway.com/en/pricing/storage/ , https://www.scaleway.com/en/pricing/network/ ,
  https://www.scaleway.com/en/pricing/virtual-instances/ ,
  https://www.scaleway.com/en/docs/object-storage/api-cli/setting-cors-rules/ ,
  https://www.scaleway.com/en/docs/object-storage/api-cli/object-operations/ ,
  https://www.scaleway.com/en/docs/object-storage/concepts/ ,
  https://www.scaleway.com/en/docs/edge-services/how-to/configure-cache/ ,
  https://www.scaleway.com/en/docs/edge-services/reference-content/understanding-pricing/
- OSM data: https://download.geofabrik.de/europe/netherlands.html , https://download.geofabrik.de/ ,
  https://taginfo.openstreetmap.org/api/4/ , and Wayback snapshots of
  https://wiki.openstreetmap.org/wiki/NL:The_Netherlands_roads_tagging ,
  https://wiki.openstreetmap.org/wiki/Key:moped , https://wiki.openstreetmap.org/wiki/Key:mofa ,
  https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing/Access_restrictions (live wiki HTTP 503 on 2026-10-21)
