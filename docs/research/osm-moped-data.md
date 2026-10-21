# OSM moped tagging quality in the Netherlands and NDW open data

Research note for issue [#3](https://github.com/AarjanM/scooterplanner/issues/3) (part of the wayfinder map, #1).
Written 2026-10-21. Terms follow `GLOSSARY.md`: Bromfiets (OSM key `moped`), Snorfiets (OSM key `mofa`), Fiets (`bicycle`), Verplicht fietspad (G11), Bromfietspad (G12a), Onverplicht fietspad (G13), Rijbaan, Rijbaan zone.

## Method and source caveats

- Counts come from two independent sources dated the same day: the Geofabrik regional taginfo for the Netherlands (data until 2026-10-20T20:23Z, <https://taginfo.geofabrik.de/europe:netherlands/>) and bounded `out count;` queries against the public Overpass API (`area["ISO3166-1"="NL"]`, data timestamp 2026-10-21T09:18Z, <https://overpass-api.de/api/interpreter>). The two agree within 0.1% on `highway=cycleway` (260,302 vs 260,048), so the small difference between the Geofabrik clip polygon and the NL boundary relation is immaterial.
- The OSM wiki (wiki.openstreetmap.org) returned HTTP 503 for the whole session. Wiki quotes below come from Wayback Machine captures; the capture date is given with each URL. Treat anything dated earlier than 2025-11 as **unverified against the live wiki**.
- overpass-api.de was at capacity ("All query workers are busy"; the kumi.systems and private.coffee mirrors timed out), so the combination counts were run one by one with retries; the Amsterdam/Utrecht breakdown and a few national combinations could not be obtained and are marked **unverified** with the intended query.

## 1. Coverage counts

### 1a. Netherlands-wide, `highway=cycleway` (260,302 ways)

| Tag on `highway=cycleway` | Ways | Share | Source |
|---|---:|---:|---|
| any `traffic_sign=*` | 164,404 | 63.2% | taginfo NL [combinations](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=cycleway#combinations) |
| `traffic_sign=NL:G11` | 76,013 | 29.2% | taginfo NL |
| `traffic_sign=NL:G12a` | 65,118 | 25.0% | taginfo NL |
| `traffic_sign=NL:G13` | 20,845 | 8.0% | taginfo NL |
| any `moped=*` | 163,991 | 63.0% | taginfo NL |
| `moped=no` | 91,903 | 35.3% | taginfo NL |
| `moped=designated` | 69,270 | 26.6% | taginfo NL |
| `moped=yes` | 2,697 | 1.0% | Overpass |
| any `mofa=*` | 159,679 | 61.3% | taginfo NL |
| `mofa=designated` | 68,716 | 26.4% | taginfo NL |
| `mofa=yes` | 66,182 | 25.4% | taginfo NL |
| `mofa=no` | 24,568 | 9.4% | taginfo NL |
| **no `moped`, no `mofa`, no `traffic_sign`** | **83,897** | **32.3%** | Overpass |
| no `moped` and no `traffic_sign` | 85,948 | 33.0% | Overpass |
| `traffic_sign~G12a` without `moped=*` | 315 | 0.5% of G12a | Overpass |
| `traffic_sign~G12a` with `moped=*` but not `designated` | 593 | 0.9% of G12a | Overpass |
| `traffic_sign~G12a` with `mofa=no` | 24 | | Overpass |
| `moped=designated` without `traffic_sign` | 4,275 | 6.2% of `moped=designated` | Overpass |
| `traffic_sign~G11` with `mofa=no` (Rijbaan-zone pattern) | 1,631 | 2.1% of G11 | Overpass |
| `traffic_sign~G11` with `mofa=designated` (wiki says `yes`) | 14,927 | 19.6% of G11 | Overpass |
| `traffic_sign~G11` without `mofa=*` | 5,359 | 7.1% of G11 | Overpass |
| `traffic_sign~G11` with `moped` present and not `no` (contradiction) | 181 | 0.2% of G11 | Overpass |
| `traffic_sign~G13` with `mofa=no` | 20,495 | 98.3% of G13 | Overpass |
| `moped=designated` + `mofa=no` (Bromfiets yes, Snorfiets no) | 154 | | Overpass |
| `mofa=no` with `moped` present and not `no` | 174 | | Overpass |
| `mofa=no` without any `moped` tag | 4,628 | 18.8% of `mofa=no` | Overpass |
| `oneway=*` | 132,019 | 50.7% | taginfo NL |
| `electric_mofa=yes` (all highway types) | 362 | | taginfo NL |
| `speed_pedelec=yes` (all highway types) | 372 | | taginfo NL |

Reading: the three sign tags (G11+G12a+G13 = 161,976) and the `moped`/`mofa` tags cover almost the same 62-63% of cycleways, which is what the Dutch cycleway preset produces (sign plus derived `mofa`/`moped`, see section 2). The remaining third of all cycleways, 83,897 ways, carry none of the three and are where a router must fall back on defaults. The 2017 forum complaint that short connector pieces at crossings and roundabouts are the typical untagged ways ([community.openstreetmap.org/t/81017](https://community.openstreetmap.org/t/opletten-cycleway-de-kleine-stukjes-met-missende-mofa-en-moped-tags/81017)) is consistent with this; whether the 83,897 are mostly short connectors was **not measured** (would need a length sum, not a count).

### 1b. `use_sidepath` on carriageways

| Tag (all highway types) | Ways | Source |
|---|---:|---|
| `bicycle=use_sidepath` | 132,797 | taginfo NL |
| `mofa=use_sidepath` | 72,526 | taginfo NL |
| `moped=use_sidepath` | 46,005 | taginfo NL |
| `bicycle=use_sidepath` without any `moped=*` | 83,945 (63.2%; 132,797 minus 48,852 with `moped`) | taginfo NL [combinations](https://taginfo.geofabrik.de/europe:netherlands/tags/bicycle=use_sidepath#combinations) |
| `bicycle=use_sidepath` without any `mofa=*` | 60,156 (45.3%; minus 72,641 with `mofa`) | taginfo NL |
| `bicycle=use_sidepath` + `moped=yes` (Rijbaan open to Bromfiets, i.e. sidepath is G11) | 1,850 | taginfo NL |
| `moped=use_sidepath` without any `mofa=*` | 323 (0.7%; 46,005 minus 45,682) | taginfo NL [combinations](https://taginfo.geofabrik.de/europe:netherlands/tags/moped=use_sidepath#combinations) |
| `moped=use_sidepath` + `bicycle=use_sidepath` | 45,113 (98.0% of `moped=use_sidepath`) | taginfo NL |
| `oneway:bicycle` / `oneway:mofa` / `oneway:moped` | 38,748 / 26,396 / 22,395 | taginfo NL |

Per road class (taginfo NL, share of ways in that class carrying the tag):

| `highway=` | Ways | `bicycle=*` | `bicycle=use_sidepath` | `bicycle=no` | `moped=*` | `moped=use_sidepath` | `moped=no` | `mofa=*` | `mofa=use_sidepath` |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| primary | 49,440 | 97.6% | 33.9% | 63.4% | 60.5% | 22.6% | 37.1% | 64.6% | 26.8% |
| secondary | 75,643 | 88.3% | 57.0% | 30.7% | 33.0% | 19.9% | 11.0% | 43.7% | 31.2% |
| tertiary | 121,293 | 50.6% | 36.1% | 13.1% | 14.2% | 9.6% | n/a (<5%) | 23.2% | 18.3% |

Sources: [primary](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=primary#combinations), [secondary](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=secondary#combinations), [tertiary](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=tertiary#combinations).

Reading: `bicycle` is tagged far more consistently than `moped`, and `moped` more than `mofa`. On primaries 60% have a `moped` value; on tertiaries only 14%. A Bromfiets router that only looks at `moped=*` will therefore treat most tertiary roads as open even where a parallel Bromfietspad exists. The gap is quantified above: 63% of the 132,797 `bicycle=use_sidepath` ways carry no `moped` tag at all. Some of those are legitimately G11 sidepaths (where the Bromfiets stays on the Rijbaan; only 1,850 say so explicitly with `moped=yes`), but given 65k G12a paths against 46k `moped=use_sidepath` roads, a large share are G12a sidepaths whose Rijbaan is missing the Bromfiets restriction. The reverse direction is clean: 98% of `moped=use_sidepath` ways also have `bicycle=use_sidepath` and 99.3% have `mofa`. In July 2024 the community did a reviewed bulk edit adding `mofa=use_sidepath` where `bicycle` and `moped` were both `use_sidepath` and later batches adding `mofa=no` on primaries/secondaries/tertiaries (changesets 153867659, 153885152, 154002540, 154002522; <https://community.openstreetmap.org/t/bicycle-and-moped-tags-hetzelfde-mofa-ook/115534>). Before that edit about 20% of `moped=use_sidepath` ways lacked `mofa` (emvee's count in the same thread).

### 1c. Other highway types and signs relevant to the Legal route

| Item | Ways | Source |
|---|---:|---|
| `highway=path` | 240,918 | taginfo NL |
| `highway=path` with `bicycle=*` / `bicycle=no` / `bicycle=yes` | 17.6% / 10.0% / 4.0% | taginfo NL [path](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=path#combinations) |
| `highway=path` with `moped=*` / `moped=no` | 2,671 (1.1%) / 1,517 | taginfo NL |
| `highway=path` with `mofa=*` / `mofa=no` | 2,430 (1.0%) / 1,421 | taginfo NL |
| `highway=path` with `motor_vehicle=no` | 12,146 (5.0%) | taginfo NL |
| `highway=path` with any `traffic_sign` | 1,640 (0.7%), of which 879 are `traffic_sign=none`; no G11/G12a/G13 value reaches the taginfo combination list, so cycle-path signs on `highway=path` are rare (exact count **not verified**, Overpass query failed) | taginfo NL |
| `highway=footway` | 658,924 | taginfo NL |
| `highway=footway` with `bicycle=*` | 3.6% | taginfo NL [footway](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=footway#combinations) |
| `highway=footway` with `moped=*` / `mofa=*` | 1,093 (0.17%) / 1,013 (0.15%), mostly `no` | taginfo NL |
| `highway=footway` with `traffic_sign=NL:G7` or `NL:G07` | 5,735 + 2,784 = 8,519 (1.3%) | taginfo NL |
| `highway=motorway` | 13,797 | taginfo NL |
| `highway=trunk` | 8,625; `motorroad=*` on 99.4%, `motorroad=yes` on 81.2% | taginfo NL [trunk](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=trunk#combinations) |
| `highway=trunk` with `motorroad=no` (non-Autoweg trunk, open by default) / without `motorroad` | 1,567 (18.2%) / 53 (0.6%) | taginfo NL |
| `highway=trunk_link` | 3,594; `motorroad=yes` 2,359 (65.6%), `motorroad=no` 1,069 (29.7%), untagged 166 (4.6%) | taginfo NL [trunk_link](https://taginfo.geofabrik.de/europe:netherlands/tags/highway=trunk_link#combinations) |
| `motorroad=yes` (all classes) | 9,885 | taginfo NL |
| `motorroad=yes` with `moped=*` (almost all `no`) | 384 (3.9%); 96% rely on the motorroad implication | taginfo NL [motorroad=yes](https://taginfo.geofabrik.de/europe:netherlands/tags/motorroad=yes#combinations) |
| `traffic_sign=NL:G3` (Autoweg sign on a way) | 60 | taginfo NL |
| `traffic_sign=NL:C9` (closed to slow traffic incl. Bromfiets/Snorfiets/Fiets) | 8,194 | taginfo NL |
| `traffic_sign=NL:C12` (closed to motor vehicles, Bromfiets/Snorfiets excepted) | 1,042 | taginfo NL |
| `traffic_sign=NL:C13` (closed to Bromfiets/Snorfiets with running engine) | 170 | taginfo NL |
| `traffic_sign=NL:C15` (closed to Fiets and Bromfiets, Snorfiets included) | 4,455 | taginfo NL |
| `traffic_sign=NL:D103` / `NL:D104` (Bromfiets lane-change signs at bromfietsdoorsteek) | 330 / 280 | taginfo NL |
| `cycleway=lane` / `cycleway:both=*` / `cycleway:right=*` / `cycleway:left=*` | 12,874 / 35,019 / 11,695 / 3,616 | taginfo NL |
| `moped:conditional` / `mofa:conditional` | 351 / 362 | taginfo NL |
| `route=ferry` | 580 | taginfo NL |
| `route=ferry` with `moped=*` / `mofa=*` / `bicycle=*` | **not verified**: the taginfo combination list for `route=ferry` returns no access keys at all and the Overpass count failed on server capacity; access is probably tagged on the `type=route` relations rather than the ways | taginfo NL / Overpass |

Context: the Netherlands holds about 80% of all `moped=*` tagging in OSM worldwide (264,482 of 329,171 ways; global taginfo <https://taginfo.openstreetmap.org/keys/moped>, NL taginfo above) and 85% of `mofa=*` (285,695 of 334,204). The tag semantics are therefore effectively defined by Dutch practice.

## 2. Default access a router must assume for untagged ways

### What the Dutch wiki says mappers should do

NL:Cycleway (Wayback 2025-11-11, <https://web.archive.org/web/20251111230604/https://wiki.openstreetmap.org/wiki/NL:Cycleway>):

- "Voor een cycleway geldt per definitie al bicycle=designated." Fiets access on `highway=cycleway` is implied.
- "De tags voor mofa (snorfiets) en moped (bromfiets) volgen uit het type fietspad (op basis van het geplaatste verkeersbord)." The three types and their tagging:
  - Verplicht fietspad G11: `highway=cycleway` + `traffic_sign=NL:G11` + `mofa=yes` + `moped=no`.
  - Fiets/bromfietspad G12a: `highway=cycleway` + `traffic_sign=NL:G12a` + `mofa=designated` + `moped=designated`.
  - Onverplicht fietspad G13: `highway=cycleway` + `traffic_sign=NL:G13` + `moped=no` + `mofa=no` + `electric_mofa=yes` ("Fietsen en elektrisch aangedreven snorfietsen mogen dit pad gebruiken. Brommers en snorfietsen met verbrandingsmotor zijn verboden.").
- "Paden zonder een van deze borden worden niet getagd met highway=cycleway, maar bijvoorbeeld highway=path." So by convention an untagged `highway=cycleway` is still one of the three signed types, with the sign unrecorded.
- Onderbord exceptions are tagged explicitly, e.g. G11 + "Speed-pedelecs toegestaan" gets `speed_pedelec=yes`.
- The adjacent Rijbaan of a G11/G12a path gets `bicycle=use_sidepath` ("Eventueel aangevuld met vergelijkbare tags voor mofa en moped").
- Oneway: "Volgens de Nederlandse wet is dit voor fietspaden langs wegen de standaard ... In OSM wordt echter bij het ontbreken van een tag voor 'oneway' aangenomen dat in twee richtingen mag worden gereden." Explicit `oneway=yes` is recommended; only 50.7% of cycleways have any `oneway` tag.
- The page itself warns: "Er zijn echter maar weinig routeerders die landspecifieke standaardwaarden hanteren."

The same mapping is given in the sign overview NL:Overzicht Nederlandse Verkeersborden (Wayback 2025-12-25, <https://web.archive.org/web/20251225032705/https://wiki.openstreetmap.org/wiki/NL:Overzicht_Nederlandse_Verkeersborden>), which also gives: G3 autoweg = `highway=trunk` + `motorroad=yes` + `traffic_sign=NL:G3`; C9 = `moped=no & mofa=no & bicycle=no & ...`; C12 = `motor_vehicle=no & moped=yes & mofa=yes`; C13 = `moped=no & mofa=no`; C14 = `bicycle=no` (Snorfiets with running engine is NOT covered by C14, per RVV art. 2b and the C13 definition); C15 = `bicycle=no & moped=no & mofa=no`; D103/D104 = Bromfiets lane-change at a bromfietsdoorsteek. It notes "Er is binnen OSM geen tag specifiek voor snorfietsen zonder in werking zijnde motor. Bij mofa=* gaan we er in het algemeen van uit dat de snorfietsmotor is ingeschakeld."

NL:The_Netherlands_roads_tagging (Wayback 2026-02-01, <https://web.archive.org/web/20260201035401/https://wiki.openstreetmap.org/wiki/NL:The_Netherlands_roads_tagging>) adds the "Bromfietsdoorsteek" pattern: a short `highway=cycleway` + `moped=designated` ramp between Rijbaan and G12a path at the built-up-area boundary, with the Rijbaan split so that the part where Bromfiets must use the path has `moped=use_sidepath` and an explicit C13/C15 ban has `moped=no`. It also says "highway=trunk motorroad=no [impliceert] geen toegangsrestricties".

### What the routing-defaults table says a data consumer may fall back on

OSM tags for routing/Access restrictions, Netherlands section (Wayback 2026-01-04, <https://web.archive.org/web/20260104180215/https://wiki.openstreetmap.org/wiki/OSM_tags_for_routing/Access_restrictions#Netherlands>). Fallback values relevant to the three Vehicle classes:

| `highway=` | moped (Bromfiets) | mofa (Snorfiets) | bicycle (Fiets) | foot |
|---|---|---|---|---|
| motorway, motorway_link | no | no | no | no |
| trunk(_link) with `motorroad=yes` | no | no | no | no |
| trunk(_link) without motorroad, primary, secondary, tertiary, unclassified, residential, service, road, track | yes | yes | yes | yes |
| living_street | yes | yes | yes | designated |
| path | yes | yes | yes | yes |
| busway | no | no | no | no |
| bridleway | no | no | no | no |
| **cycleway** | **dismount** [NL 16] | **yes** [NL 17] | designated | yes |
| footway, pedestrian | dismount | dismount | dismount | designated |

Footnotes quoted: [NL 16] "Mopeds ('bromfietsen') are not allowed to use cycleways indicated by the compulsory cycleway sign [G11], or the optional cycleway sign [G13]. There is a sign that allows both bicycles and mopeds [G12a], but in that case moped=designated is tagged explicitly." [NL 17] "Mofas ('snorfietsen' < 25 km/h) must use cycleways indicated by the compulsory cycleway signs [G11] and [G12a], but are not allowed to use cycleways indicated by the optional cycleway sign [G13]. In the latter case, mofa=no is tagged explicitly."

So: **an untagged `highway=cycleway` is to be treated as G11** (Fiets yes, Snorfiets yes, Bromfiets no). That is the only defensible fallback: it is the most common signed type (76k G11 vs 65k G12a), the wiki documents G12a as always explicitly tagged, and it is the conservative choice for a Bromfiets (a false "no" costs a detour, a false "yes" produces an illegal route).

The same page's caveats, which matter for the Rijbaan side:

- "the actual legal default value for moped, mofa, bicycle, horse & foot on a carriageway (such as on a residential, tertiary etc.) is NOT yes but use_sidepath; only when a designated side path is not present these road users may use the carriageway."
- "a highway=primary will seldom be allowed to be used with a bicycle: as per 2022-06-18 Taginfo shows that more than 95% of these ways in OSM in the Netherlands has a negative value tagged."
- "a highway=path will seldom be allowed to be used with a moped of mofa; access for bicycle is very different between regions and operators."
- "highway=footway is used in OSM for mixed purposes in the Netherlands" (formal G7 footways, sidewalks, permissive paths, private paths).
- "As per 2024, no mainstream routing engine is known to use this page directly."

Consequences for ScooterPlanner:

- `highway=path` default `moped=yes`/`mofa=yes` from the table is contradicted by the page's own caveat. The forum is split: emvee argues explicit tags are unnecessary because the default allows all three; PeeWee32 argues defaults are a consumer fallback, not a mapping licence (<https://community.openstreetmap.org/t/bicycle-and-moped-tags-hetzelfde-mofa-ook/115534>). Only 17.6% of paths carry any `bicycle` tag, so a decision is needed (section "Tag combinations").
- `highway=footway` default is `dismount` for all three; the Snorfiets-engine-off case has no tag.
- For Rijbaan ways, a Bromfiets router cannot rely on `moped=use_sidepath` being present (section 1b): a geometric "is there a parallel G12a path?" check or a `bicycle=use_sidepath` proxy must be considered.

### Community consensus state (forum)

- The G11 `mofa=yes` vs `mofa=designated` debate (Jan 2024, <https://community.openstreetmap.org/t/mofa-yes-of-designated-bij-g11-bord/109438>) ended without a decision; preset and wiki stayed at `mofa=yes`. For a router both values mean "allowed"; the only safe reading of `designated` is "allowed and probably preferred", not "mandatory".
- The G13 thread (Jan 2024, <https://community.openstreetmap.org/t/g13-onverplicht-fietspad-electric-mofa-cycleway-both-etc/108408>) disagreed on whether `mofa=no` on G13 excludes electric Snorfietsen; `electric_mofa=yes` was added to the wiki (362 uses so far).
- `moped=use_sidepath` never went through a proposal; the wiki asks routers to "treat it as a negative access value, though may be weaker than moped=no" (Wayback 2026-04-16, <https://web.archive.org/web/20260416044659/https://wiki.openstreetmap.org/wiki/Tag:moped%3Duse_sidepath>). It "applies only to roads with a classification that normally allows moped use"; `moped=no` is for an explicit sign on the main road.
- A 2024 routing thread reports BRouter ignoring `moped=designated` on a G12a path (<https://community.openstreetmap.org/t/routing-vraagje/74261?page=2>), i.e. no mainstream engine currently implements the Dutch semantics.

## 3. Snorfiets Rijbaan zones in OSM

### Legal shape of a Rijbaan zone

- National basis: Besluit van 6 juni 2018 (Stb. 2018, 184, "lokale scheiding fiets en snorfiets"), in force 1 July 2018 (<https://zoek.officielebekendmakingen.nl/stb-2018-184.html>). It adds BABW art. 8 lid 2 onder f: the G11 sign with an onderbord stating that snorfietsen may not use the fietspad; BABW art. 14a: the traffic decision must motivate it with crowding on the fietspad; RVV 1990 art. 5 lid 8: the Snorfiets rider must then use the Rijbaan; RVV art. 60: helmet obligation applies there. **The instrument exists only for G11.** A G12a (Bromfietspad) or G13 path is not covered, so inside a Rijbaan zone a G12a path legally still admits a Snorfiets unless the authority re-signs it.
- Amsterdam: Verkeersbesluit "Snorfiets naar de rijbaan met helmplicht", 11 Dec 2018, Stcrt. 2018, 71559, in force 8 April 2019 (<https://zoek.officielebekendmakingen.nl/stcrt-2018-71559.html>). "Het college voert de maatregel op alle verplichte fietspaden in de stad binnen de ring A10 in", but legally it is a street-by-street enumeration per wijk (Oost, Zuid, Centrum, West, Noord) referencing drawings SNOR-BO-*. Exceptions where the Snorfiets stays on the fietspad: "de belangrijkste doorstromingswegen voor auto en voor openbaar vervoer" (Prins Hendrikkade CS–IJtunnel, the north-south and east-west axes, the S100, Kattenburgerstraat) and, "voorlopig", Schellingwouderbrug and Amsterdamsebrug. The measure is permanent (not peak-only). The city map <https://maps.amsterdam.nl/snorfietsnaarrijbaan/> visualises the decision but states it will not be maintained and will eventually be removed; its legend references a GeoJSON download, but the legend file returned 404 on 2026-10-21, and the city's open geodata catalogue (<https://maps.amsterdam.nl/open_geodata/>) lists no "snorfiets naar de rijbaan" layer (only "Milieuzones uitstootvrij bromsnorfiets"), so **no maintained open dataset of the zone was found**.
- Utrecht: Gemeenteblad 2021, 212045 (decision 29 June 2021, in force 30 Sept 2021, <https://zoek.officielebekendmakingen.nl/gmb-2021-212045.html>): G11 with onderbord "Snorfietsen niet toegestaan" on 43 enumerated road sections (Amsterdamsestraatweg, Catharijnesingel, Vleutenseweg, Croeselaan, Biltstraat, ... ) plus 42 enumerated solitary cycle paths, i.e. the inner-city ring area, not a closed zone. The municipality page <https://www.utrecht.nl/snorfiets> describes the three signs used (blue-arrow "Snorfiets links de rijbaan op", red-arrow "Snorfiets rechts de rijbaan op", onderbord "Snorfietsen niet toegestaan") and says "If no sign applies, snorfietsers may use the fietspad". No map dataset is linked.

So a "Rijbaan zone" is, legally, not an area but a set of individually re-signed G11 paths. That is also how OSM represents it.

### How it appears in OSM

- **No relation, boundary or zone object exists.** An Overpass search of the NL area for relations or boundaries whose name/description/note mention "snorfiets" and for relations carrying `mofa`/`moped` keys could not be completed because the public server was at capacity (**unverified**), but neither the wiki nor the forum mentions any such object, and the global taginfo shows only 31 relations with `mofa=*` at all (<https://taginfo.openstreetmap.org/keys/mofa>).
- The representation is per way: `highway=cycleway` + `traffic_sign=NL:G11` (optionally with the onderbord appended) + **`mofa=no`**. Key:mofa (Wayback 2025-11-24, <https://web.archive.org/web/20251124231137/https://wiki.openstreetmap.org/wiki/Key:mofa>) states for the Netherlands: "mofas ('snorfietsen') follow largely the same access restrictions (including compulsory use of cycleways) as bicycles, except for cycleways with a G13 sign, [and] mofas are banned from the majority of cyclepaths witin the ringway A10 in Amsterdam". The forum confirms the practice: "in those Amsterdam sections, the tagging is mofa=no based on the onderborden" (Peter_Elderson) and the G11+onderbord is a "vaste G11-OB-combi" (JeroenHoek), Jan 2024, <https://community.openstreetmap.org/t/mofa-yes-of-designated-bij-g11-bord/109438?page=2>.
- `traffic_sign` values on Amsterdam cycleways (Overpass, 2026-10-21, all `highway=cycleway` with a `traffic_sign` inside the Amsterdam admin_level=8 area): `NL:G11` 1,822; `NL:G12a` 1,006; `NL:G11,OB1006` 82; `NL:G13` 39; `NL:G11,"snorfietsen niet toegestaan"` 27 (+10 with a typo "toegestaand", +1 capitalised); `NL:G11,OB505` 2; `NL:G07` 5. So the onderbord is recorded on only about 120 ways and in at least four spellings; the rest of the zone is only recoverable from `mofa=no`. The meaning of code "OB1006" was **not verified** (the wiki onderbord table lists OB1–OB59 style codes, not OB1006; it may be a municipal code).
- **Unverified (Overpass at capacity, queries not completed):** the per-municipality breakdown. Intended queries, all of the form `area["name"="Amsterdam"]["admin_level"="8"]["boundary"="administrative"]->.a; way(area.a)["highway"="cycleway"]->.c;` followed by `out count` for: `.c` (all), `.c["mofa"="no"]`, `.c["mofa"="yes"]`, `.c["mofa"="designated"]`, `.c[!"mofa"]`, `.c["traffic_sign"~"G11"]["mofa"="no"]`, `.c["traffic_sign"~"G11"]["mofa"="yes"]`, `.c["traffic_sign"~"G12a"]["mofa"="no"]`, `.c["traffic_sign"~"G12a"]["mofa"="designated"]`, plus `way(area.a)["highway"]["mofa"="use_sidepath"]` and `way(area.a)["highway"]["moped"="use_sidepath"]` (Rijbaan ways inside the zone that still send a Snorfiets to the sidepath; a `mofa=use_sidepath` road next to a `mofa=no` path is a contradiction the pipeline should detect). Same set for `area["name"="Utrecht"]`, plus `.c["traffic_sign"~"snorfiets",i]`. What is known: the national G11 + `mofa=no` count is 1,631 ways, and Amsterdam alone has 1,822 G11-signed cycleways, so the Amsterdam zone is clearly not fully expressed as G11 + `mofa=no`; part of it must be `mofa=no` on unsigned cycleways (4,628 nationally) or still missing. Re-run these counts in the routing-rules ticket (the Geofabrik taginfo cannot filter by municipality; a local Overpass or `osmium tags-filter` on the Noord-Holland/Utrecht extracts can).
- Nationally, `mofa=no` appears on 24,568 cycleways, of which 20,495 are signed G13 (the normal Onverplicht fietspad case) and 1,631 are signed G11; `mofa=no` without any `moped` tag occurs on 4,628 ways and `mofa=no` next to a non-`no` `moped` on 174. The G11 + `mofa=no` combination is the Rijbaan-zone signature to look for outside Amsterdam and Utrecht (other municipalities may have used the 2018 instrument; **not researched**).
- Tagging drift to note: 14,927 G11 cycleways (19.6%) carry `mofa=designated` instead of the wiki's `mofa=yes`, and 5,359 (7.1%) carry no `mofa` at all. A router must treat `yes` and `designated` alike and fall back to "Snorfiets allowed" on an unsigned-for-mofa G11. G12a tagging is far more consistent: only 315 G12a ways lack `moped`, 593 have a non-`designated` `moped` value and 24 have `mofa=no`.
- Forum caveat from OttoR (Amsterdam mapper): some Amsterdam cycleways carry `mofa=designated` with `moped=no`, others `moped=designated`, so no blanket rule per sign holds there (<https://community.openstreetmap.org/t/mofa-yes-of-designated-bij-g11-bord/109438>).

Conclusion: a router can honour the Amsterdam and Utrecht Rijbaan zones **only through per-way `mofa=no`**, and must then also treat the parallel Rijbaan as open to Snorfiets regardless of any `mofa=use_sidepath` left on it. Helmet obligation (RVV art. 60) applies on those Rijbaan stretches; this is a Turn-by-turn navigation warning concern, not a routing one, and there is no OSM tag for it.

## 4. NDW open data

Portal: <https://opendata.ndw.nu/> (file index, no descriptions). Docs: <https://docs.ndw.nu/>. Licence: ndw.nu/copyright states "Tenzij anders vermeld is op de inhoud van deze website de Creative Commons Zero (CC0) verklaring van toepassing" (<https://www.ndw.nu/copyright>); images are excluded, and derivative works may not suggest NDW endorsement. The portal and product pages do not restate a licence, so **CC0 applicability to the data files is inferred from the site-wide statement, not confirmed per dataset.** No API key is needed for the portal files (plain HTTPS GET worked).

Cadence: the files are full snapshots rewritten about every minute. Observed on 2026-10-21: `Last-Modified: 09:36:10Z` fetched at 09:36:28Z, with `<com:publicationTime>2026-10-21T09:36:00Z` inside the file. The incidents FAQ says the same ("wordt iedere minuut bijgewerkt", <https://docs.ndw.nu/faq/incidenten/>). None of the product pages state a frequency.

Format: DATEX II v3 XML (<https://docs.ndw.nu/producten/situatieberichten-v3/>), namespaces `http://datex2.eu/schema/3/situation` and `.../locationReferencing`. Location referencing in the closures file (inspected 2026-10-21): `loc:gmlLineString/loc:posList` on 884 records (WGS84 linestrings), ALERT-C/VILD table references on 611, `loc:pointByCoordinates` on 107. The GML geometry makes map-matching to OSM ways feasible; VILD-only records would need the VILD table (also on the portal, VILD6.14.A.zip).

| File on opendata.ndw.nu | Content (docs) | Size (2026-10-21) | Fit for ScooterPlanner |
|---|---|---:|---|
| `actueel_beeld.xml.gz` | All real-time situations, "ongevalideerd": accidents, incidents, road condition, weather, queues, plus the feeds below (<https://docs.ndw.nu/producten/situatieberichten-v3/actueel-beeld/>) | 411 KB | Warnings only |
| `tijdelijke_verkeersmaatregelen_afsluitingen.xml.gz` | Closed roads/carriageways/lanes and bridge closures entered by "wegbeheerders" (<https://docs.ndw.nu/producten/situatieberichten-v3/vm-afsluitingen/>). Sample: 810 situations, 958 records; 711 `carriagewayClosures`, 129 `laneClosures`, 9 `roadClosed`; sources are RWS districts, provinces (Overijssel, Limburg, Gelderland) and municipalities (Rotterdam 37, Utrecht 23, Den Haag 20, Eindhoven 19, Amsterdam 15, Groningen 11, ...). **Zero occurrences of "fietspad", "snorfiets" or "bromfiets"** in the whole file. | 239 KB | Candidate for temporary Rijbaan blocks in the Route plan (geometry + validity window present); useless for Bromfietspad/Verplicht fietspad closures |
| `planningsfeed_wegwerkzaamheden_en_evenementen.xml.gz` | Planned works and events incl. detours, speed restrictions, temporary lights (<https://docs.ndw.nu/producten/situatieberichten-v3/werkzaamheden-evenementen/>); data from a registration system (Melvin, <https://melvin.ndw.nu/public>, "landelijk in te zetten door alle wegbeheerders", scope "mainly the RVM network") | 16 MB | Warnings; possibly planned blocks |
| `tijdelijke_verkeersmaatregelen_maximum_snelheden.xml.gz` | Temporary speed limits (<https://docs.ndw.nu/producten/situatieberichten-v3/vm-max-snelheid/>) | 27 KB | Irrelevant (Bromfiets/Snorfiets limits are vehicle-bound, 45/25 km/h) |
| `planningsfeed_brugopeningen.xml.gz` | Planned and current bridge openings from bridge systems and sensors (<https://docs.ndw.nu/producten/situatieberichten-v3/planningsfeed-brugopeningen/>) | 121 KB | Warnings / ETA |
| `veiligheidsgerelateerde_berichten_srti.xml.gz` | Safety-related messages, **rijkswegen only** (<https://docs.ndw.nu/producten/situatieberichten-v3/srti/>) | 26 KB | Irrelevant (Autoweg/autosnelweg are always excluded) |
| `verkeersborden_actueel_beeld.{csv,geojson}.gz` (RD and WGS84) | Point data of traffic signs with `rvvCode`, `blackCode`, `zoneCode`, `status`, `textSigns[]` (onderbord type + text), `roadSectionId` (NWB), `placedOn`, `imageUrl`, `trafficOrderUrl` (<https://docs.ndw.nu/data-uitwisseling/interface-beschrijvingen/verkeersborden-api/>). Source: HR-groep survey cars bought by I&W for mid-2020 to Q1 2023, plus a mutation app for road authorities "rarely used in practice"; survey updated yearly, actueel beeld compiled monthly, API `current-state` daily (<https://docs.ndw.nu/producten/verkeersborden/>). | 235 MB | Offline QA: cross-check OSM `traffic_sign=NL:G11/G12a/G13` and onderbord texts ("snorfietsen niet toegestaan") against surveyed signs. Not real-time, no completeness claim. |
| `emissiezones.xml.gz` | Emission zones | 145 KB | Possibly relevant later: Amsterdam publishes a "Milieuzones uitstootvrij bromsnorfiets" layer in its open geodata catalogue and utrecht.nl mentions a ban on older Snorfietsen from 1 Jan 2028; whether this NDW file carries moped emission zones was **not checked** |
| `measurement*.xml.gz`, `trafficspeed.xml.gz`, `traveltime.xml.gz`, `reistijden_*`, `snelheden_en_intensiteiten_*` | Loop/travel-time measurements; v2.3 files being replaced by v3 between 2026-08-31 and 2027-02-09 | 1-9 MB | Irrelevant |
| `incidents.xml.gz` | Named in the FAQ as open data; **returned HTTP 404 on 2026-10-21** and is absent from the index | - | Unverified |

Not offered by NDW: ferries (none on the portal; OSM `route=ferry` is the only source), permanent speed limits (those are Rijkswaterstaat WKD Geopackage, monthly around the 5th, cycle paths recorded as "unknown", <https://docs.ndw.nu/handleidingen/wkd/wkd_maxsnelheden/>), and the cycle-path network as such (the NWB gained a fietsnetwerk in 2022, downloadable from nationaalwegenbestand.nl, <https://docs.ndw.nu/producten/fietsdata/>; licence not checked).

Verdict: NDW real-time feeds fit **warnings** (bridge openings, incidents, works) and perhaps **temporary Rijbaan blocks** for the Bromfiets profile, since closures carry WGS84 geometry and validity windows. They do not help the core Legal-route graph: no feed describes cycle-path access, the sampled closures contain no cycle-path events, and coverage is biased to RWS/provincial roads and large municipalities. The signs dataset is a one-off/monthly QA source, not a routing input.

## 5. Netherlands OSM extract

- Geofabrik: <https://download.geofabrik.de/europe/netherlands.html>. `netherlands-latest.osm.pbf` 1.3 GB, "contains all OSM data up to 2026-10-20T20:23:47Z"; `netherlands-updates/state.txt` sequenceNumber 4944, timestamp 2026-10-20T20:23:47Z. Licence ODbL 1.0; files on the public server have no `user`/`uid`/`changeset` metadata (<https://download.geofabrik.de/technical.html>).
- Cadence: Geofabrik re-reads the planet "once a day, around 21:00 CET" and publishes one `.osc.gz` diff per extract run; observed diffs 940-944 dated 2026-10-17 through 2026-10-21, 1.2-1.5 MB each (<https://download.geofabrik.de/europe/netherlands-updates/000/004/>). Diffs are kept 100 days; `pyosmium-up-to-date` applies them. Older dated `.osm.pbf` snapshots are kept "for a while".
- Province sub-extracts: Drenthe 60 MB, Flevoland 34 MB, Friesland 84 MB, Gelderland 186 MB, Groningen 54 MB, Limburg 96 MB, Noord-Brabant 197 MB, Noord-Holland 181 MB, Overijssel 108 MB, Utrecht 94 MB, Zeeland 42 MB, Zuid-Holland 202 MB. Useful for a prototype (Utrecht + Noord-Holland = 275 MB).
- Taginfo for the same extract: <https://taginfo.geofabrik.de/europe:netherlands/> (API `/api/4/...`), refreshed with the extract; this is the cheapest way to re-run the coverage table.
- Not researched: minutely planet replication (planet.openstreetmap.org) as an alternative to daily Geofabrik diffs; only needed if sub-day freshness matters, which the data quality in section 1 does not justify.

## Tag combinations the routing rules must decide on

Explicit list for the follow-up grilling ticket. "F/S/B" = Fiets / Snorfiets / Bromfiets.

1. `highway=cycleway` with none of `traffic_sign`, `moped`, `mofa` (83,897 ways, 32%). Proposed: treat as G11 (F yes, S yes, B no). Decide: yes/no, and whether to flag such ways in the UI as "unverified".
2. `highway=cycleway` + `traffic_sign=NL:G12a` (or `moped=designated`): F/S/B all yes. Decide: does `moped=designated` without the sign count, and does the sign without `moped` count (315 ways)?
3. `highway=cycleway` + `traffic_sign=NL:G11` + `mofa=yes` vs `mofa=designated`: both "allowed" for S. Decide: identical weight.
4. `highway=cycleway` + `traffic_sign=NL:G13` (+ `mofa=no`, `moped=no`): F yes, S no (engine on), B no. Decide: treat `electric_mofa=yes` (362 ways) as S-allowed, or ignore and route S around all G13 paths?
5. `highway=cycleway` + `mofa=no` while `moped` is not `no` or absent (Rijbaan-zone signature): S no regardless of sign. Decide: and force the parallel Rijbaan open for S even when it has `mofa=use_sidepath`.
6. `traffic_sign` strings containing an onderbord: `NL:G11,OB1006`, `NL:G11,"snorfietsen niet toegestaan"` (several spellings). Decide: parse the onderbord text as a secondary S-ban signal, or rely only on `mofa=no`.
7. `speed_pedelec=yes` on a G11 path (372 ways): the Speed pedelec vehicle type maps to Bromfiets but may use that path. Decide: give Speed pedelec its own access resolution (`speed_pedelec` > `moped`) or ignore until there is an explicit vehicle type.
8. `highway=path` untagged (82% have no `bicycle` tag): wiki fallback says F/S/B all yes; wiki caveat says mopeds are "seldom allowed". Decide: F yes / S ? / B no, or require `bicycle=*` before using any path.
9. `highway=footway` / `pedestrian` untagged: default `dismount` for all three. Decide: exclude, or allow at walking speed as a last-resort link (and only for F?).
10. `highway=footway` + `bicycle=yes` (G7 + onderbord "fietsen toegestaan"): F yes. Decide: does S follow F here (RVV art. 2b says the bicycle symbol covers Snorfiets "tenzij anders bepaald")?
11. Rijbaan ways (`primary`...`residential`) without any `moped`/`mofa` tag: B yes, S yes by the fallback table, but the legal default is `use_sidepath` when a parallel path exists. Decide: use `bicycle=use_sidepath` as a proxy for `moped=use_sidepath` (B must then check whether the sidepath is G12a: if it is G11, B stays on the Rijbaan), and/or a geometric parallel-path test.
12. `moped=use_sidepath` / `mofa=use_sidepath`: wiki says "negative access value, though may be weaker than no". Decide: hard exclusion vs heavy penalty.
13. `highway=trunk` + `motorroad=yes` (and `traffic_sign=NL:G3`), `highway=motorway*`: always excluded (Autoweg). `highway=trunk` with `motorroad=no` (1,567 ways, 18%) or without `motorroad` (53): open by default per wiki, and `trunk_link` with `motorroad=no` (1,069) is how Autoweg roundabouts and crossings are mapped. Decide: trust that, or exclude trunks entirely for S.
14. `highway=primary/secondary` with `bicycle=no` but no `moped` tag: C9/C15 would ban B too, C14 would not. Decide: whether `bicycle=no` implies `moped=no` when `traffic_sign` is absent (63% of primaries have `bicycle=no`, only 37% `moped=no`).
15. `traffic_sign=NL:C12` (`motor_vehicle=no` + `moped=yes` + `mofa=yes`): B and S allowed despite `motor_vehicle=no`. Decide: ensure the access hierarchy evaluates `moped`/`mofa` before `motor_vehicle`.
16. `traffic_sign=NL:C13` (`moped=no`, `mofa=no`, 170 ways) vs `NL:C14` (`bicycle=no` only, S with running engine allowed). Decide: confirm the hierarchy handles S-allowed-but-F-banned.
17. `highway=living_street`: all yes, `foot=designated`. Decide: speed/penalty only.
18. `highway=busway` / `psv` lanes: no for all. Decide: nothing, but confirm `lanes:psv` roads are not mistaken for closed roads.
19. `oneway` absent on cycleways (49%): OSM assumes two-way; Dutch law makes paths along roads one-way. Decide: trust OSM (risk: wrong-way instructions) or apply the NL assumption for sidepaths only.
20. `oneway:bicycle=no` without `oneway:moped`/`oneway:mofa` (38,748 vs 22,395 / 26,396): Decide: do B and S inherit the F exception.
21. `cycleway=lane` / `cycleway:both=lane` / `shared_lane` on a Rijbaan (fietsstrook): legally part of the Rijbaan; B rides the Rijbaan, S may use the lane. Decide: no access change, only guidance text.
22. Bromfietsdoorsteek (`highway=cycleway` + `moped=designated` short ramp, `traffic_sign=NL:D103/D104`): B must switch here. Decide: how Turn-by-turn navigation phrases it; no access decision.
23. `moped:conditional` / `mofa:conditional` (351 / 362 ways): Decide: evaluate time-dependent access or treat as `no`.
24. `route=ferry` (580 ways; access tagging coverage unverified, likely on relations): Decide: default for untagged ferries per Vehicle class, whether to require `bicycle=yes` as a proxy, and whether to read access from the route relation.
25. NDW `tijdelijke_verkeersmaatregelen_afsluitingen` `carriagewayClosures`: Decide: apply as temporary Rijbaan blocks for B (and for S inside Rijbaan zones) or warnings only.

## Not verified / open

- Live OSM wiki text (all wiki quotes are from Wayback captures dated 2025-11 to 2026-04).
- Whether any relation or boundary object models a Rijbaan zone (Overpass query failed for capacity), and the Amsterdam/Utrecht per-way counts listed in section 3.
- Cycle-path signs on `highway=path` (intended: `way(area.nl)["highway"="path"]["traffic_sign"~"NL:G1[123]"]`) and access tags on `route=ferry` ways/relations.
- Meaning of onderbord code "OB1006" in Amsterdam `traffic_sign` strings.
- Whether a maintained open dataset of the Amsterdam zone exists (maps.amsterdam.nl legend 404; data.amsterdam.nl not searched directly).
- Whether municipalities other than Amsterdam and Utrecht have used the 2018 instrument.
- CC0 applicability per NDW dataset; `incidents.xml.gz` existence; whether any road authority registers cycle-path closures in Melvin.
- Length-weighted (not count-weighted) share of untagged cycleways.
