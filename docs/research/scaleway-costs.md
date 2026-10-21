# Scaleway sizing and pricing under €50/month

Resolves [issue #5](https://github.com/AarjanM/scooterplanner/issues/5) (part of the wayfinder map, #1).

All prices are **EUR, excluding VAT**, as **observed on 2026-10-21**. Monthly figures use Scaleway's own
convention of 730 h/month (the Instance catalog API returns `monthly_price = hourly_price × 730`, and the
pricing pages mark monthly numbers "approx."). Prices change; every number below carries its source URL.
Items marked **[unverified]** could not be confirmed from a primary source and need checking before they
drive a decision.

## TL;DR

- Under €50/month, Scaleway in `nl-ams` buys an **8 GB instance, not 16 GB**. The cheapest 16 GB instance
  (ARM `BASIC2-A4C-16G`, €50.30) already exceeds the ceiling on its own; 32 GB starts at €100.59.
- **Recommended**: `BASIC2-A2C-8G` (ARM, 2 shared vCPU, 8 GB, `nl-ams-1`) at **≈ €36/month all-in**,
  ≈ €14 headroom. x86 fallback: `DEV1-L` (4 shared vCPU, 8 GB) at **≈ €42/month**, ≈ €8 headroom.
- This only works if the routing engine's Netherlands graph stays at roughly **≤ 5 GB resident**. The
  routing-engine ticket must confirm that; if the engine needs the upper end of the 4–12 GB assumption,
  Scaleway cannot do it under €50 and either the budget, the provider or the single-instance shape must move.
- **Self-hosted Postgres wins on cost**: the cheapest Managed Database node is €11.39/month before storage,
  i.e. 25–30 % of the whole budget; Serverless SQL Database is Paris-only and €100/month if kept warm.
- **Instance egress is included in the instance price**; Object Storage egress costs €0.01/GB after 75 GB
  free per month. Serving tiles from the instance is therefore the cheap path at low traffic; the bucket is
  the cheap path only once bucket egress is cheaper than a bigger instance.
- `COPARM1` **no longer exists** in the catalog. Scaleway's ARM range is now `BASIC2-A*` (Ampere), and in
  the Netherlands it is only in `nl-ams-1`.
- Terraform/OpenTofu provider is mature: `scaleway/scaleway` v2.87.0 (2026-10-16), partner-premier tier,
  162 resource types covering every product in this bill of materials, mirrored in the OpenTofu registry.
- Hetzner is 2–3× cheaper for the same RAM (CX43 8 vCPU/16 GB at €16.49) but has **no Netherlands
  location** and every CX/CAX/CPX plan was shown as "currently unavailable" on the observation date.

## 1. Instances: nl-ams and fr-par, 8/16/32 GB, ARM vs x86

### Catalog snapshot

Source: Scaleway Instance catalog API, `GET https://api.scaleway.com/instance/v1/zones/{zone}/products/servers`
(public, no auth), plus `.../products/servers/availability`, queried 2026-10-21 09:20 UTC. Cross-checked
against <https://www.scaleway.com/en/pricing/virtual-instances/> (same hourly figures for every type listed there).
Architecture, shared/dedicated vCPU and bandwidth: Instances datasheet
<https://www.scaleway.com/en/docs/instances/reference-content/instances-datasheet/>.

Prices in `nl-ams-1`, `nl-ams-2`, `nl-ams-3`, `fr-par-1` and `fr-par-2` are **identical**. `fr-par-3` is a premium
zone at ≈ 1.5× (e.g. `POP2-2C-8G` €80.48 vs €53.66) and carries none of the budget ranges.

| Type | Arch | vCPU | RAM | vCPU type | Public bw | €/h | ≈ €/month | nl-ams zones | Availability nl-ams-1 |
|---|---|---|---|---|---|---|---|---|---|
| **8 GB class** | | | | | | | | | |
| `BASIC2-A2C-8G` | arm64 (Ampere M128-30) | 2 | 8 GB | shared | 200 Mb/s | 0.0345 | **25.19** | 1 | available |
| `DEV1-L` | x86 (EPYC 7281) | 4 | 8 GB | shared | 400 Mb/s | 0.04284 | **31.27** | 1, 2 | available |
| `BASIC2-A4C-8G` | arm64 | 4 | 8 GB | shared | 400 Mb/s | 0.0517 | 37.74 | 1 | available |
| `PLAY2-MICRO` | x86 (EPYC 7543) | 4 | 8 GB | shared | 400 Mb/s | 0.05508 | 40.21 | 1, 2, 3 | available |
| `PRO2-XXS` | x86 (EPYC 7543) | 2 | 8 GB | shared | 350 Mb/s | 0.0561 | 40.95 | 1, 2, 3 | scarce |
| `DEV1-XL` | x86 | 4 | 12 GB | shared | 500 Mb/s | 0.065076 | 47.51 | 1, 2 | available |
| `POP2-2C-8G` | x86 | 2 | 8 GB | dedicated | 400 Mb/s | 0.0735 | 53.66 | 1, 2, 3 | available |
| **16 GB class** | | | | | | | | | |
| `BASIC2-A4C-16G` | arm64 | 4 | 16 GB | shared | 400 Mb/s | 0.0689 | **50.30** | 1 | available |
| `GP1-XS` (older gen) | x86 | 4 | 16 GB | shared | 500 Mb/s | 0.09282 | 67.76 | 1, 2 | n/a in API |
| `POP2-HM-2C-16G` | x86 | 2 | 16 GB | dedicated | 400 Mb/s | 0.103 | 75.19 | 1, 2, 3 | available |
| `PRO2-XS` | x86 | 4 | 16 GB | shared | 700 Mb/s | 0.1122 | 81.91 | 1, 2, 3 | shortage |
| `POP2-4C-16G` | x86 | 4 | 16 GB | dedicated | 800 Mb/s | 0.147 | 107.31 | 1, 2, 3 | available |
| **32 GB class** | | | | | | | | | |
| `BASIC2-A8C-32G` | arm64 | 8 | 32 GB | shared | 800 Mb/s | 0.1378 | **100.59** | 1 | available |
| `GP1-S` (older gen) | x86 | 8 | 32 GB | shared | 800 Mb/s | 0.19074 | 139.24 | 1, 2 | n/a in API |
| `POP2-HM-4C-32G` | x86 | 4 | 32 GB | dedicated | 800 Mb/s | 0.206 | 150.38 | 1, 2, 3 | available |
| `PRO2-S` | x86 | 8 | 32 GB | shared | 1.5 Gb/s | 0.22338 | 163.07 | 1, 2, 3 | shortage |
| `POP2-8C-32G` | x86 | 8 | 32 GB | dedicated | 1.6 Gb/s | 0.29 | 211.70 | 1, 2, 3 | scarce |

Notes:

- **`COPARM1` is gone.** It appears in no zone's catalog and is absent from the product availability page
  (<https://www.scaleway.com/en/docs/account/reference-content/products-availability/>). The ARM range is
  `BASIC2-A*`, available in `fr-par-1`, `fr-par-2` and `nl-ams-1` only. `BASIC2-A*` is not on the public pricing
  page at all; its prices come from the catalog API.
- `DEV1` is only in `nl-ams-1/2` and `fr-par-1/2`; `PLAY2` in `nl-ams-1/2/3`; `PRO2`/`POP2` in all three AMS zones.
- `GP1-*` is still priced in the API but not on the pricing page; treat it as a legacy range **[unverified
  whether new GP1 instances can still be created]**.
- The "availability" field is a live stock signal (`available` / `scarce` / `shortage`). On the observation date
  **all `PRO2` sizes were in `shortage` in `nl-ams-1`** and `scarce`/`shortage` in `fr-par-1`, which argues against
  planning on `PRO2-XXS`.
- Shared vCPU ranges (`DEV1`, `PLAY2`, `BASIC2-A`, `PRO2`) are oversubscribed; a routing-graph import is CPU-heavy.
  Doing the import off-box (CI, laptop) and shipping the built graph sidesteps this.
- Bandwidth figures are "up to" caps, not metered quotas (bandwidth overview:
  <https://www.scaleway.com/en/docs/instances/reference-content/instances-bandwidth-overview/>).

### Block Storage

Source: <https://www.scaleway.com/en/pricing/storage/> (page shows Paris; the page offers no other region selector
and Instance prices are identical across `nl-ams`/`fr-par-1/2`, so parity is assumed **[unverified for nl-ams]**).

| Item | €/GB/h | ≈ €/GB/month (730 h) |
|---|---|---|
| Block Storage 5K IOPS | 0.000130 | 0.0949 |
| Block Storage 15K IOPS | 0.000177 | 0.1292 |
| Block / Unified snapshot | 0.000049 | 0.0358 |

All budget-range instances are **block-storage only** (`DEV1` also allows up to 80 GB local SSD; local-SSD pricing is
not published on the pricing page **[unverified]**). Block Storage is available in all three AMS zones.

### Public IP

- "Each Instance's flexible IPv4 costs €0.004/hour, excl. VAT" → **€2.92/month**
  (<https://www.scaleway.com/en/docs/instances/faq/>). IPv6 is included in list prices
  (<https://www.scaleway.com/en/pricing/virtual-instances/>: "List prices include egress and IPv6 addresses.
  Storage (local, block) and attached public IPv4 addresses are excluded.").
- The Network pricing page lists "Additional IP (IPv4)" at €0.005/h (≈ €3.65/month)
  (<https://www.scaleway.com/en/pricing/network/>). The two pages disagree by €0.73/month; the BoM uses the
  Instances FAQ figure and the difference is inside the headroom.

### Savings Plans

Not applicable at this scale: Compute-only, **minimum commitment €50/month**, 12 or 36 months, discount in the
10 % order of magnitude (<https://www.scaleway.com/en/docs/billing/additional-content/understanding-savings-plans/>).

## 2. Object Storage: tiles and backups

Source: <https://www.scaleway.com/en/pricing/storage/> (Paris; same **[unverified for nl-ams]** caveat as above),
<https://www.scaleway.com/en/docs/object-storage/faq/>.

| Item | Price |
|---|---|
| Standard Multi-AZ | €0.000022/GB/h ≈ **€0.01606/GB/month** |
| Standard One Zone | €0.000011/GB/h ≈ **€0.00803/GB/month** |
| Glacier (Paris/Amsterdam buckets only) | €0.0000035/GB/h ≈ €0.00254/GB/month; restore €0.009/GB |
| Requests (GET/PUT/LIST) | included |
| Ingress | included |
| Egress to internet | **75 GB free every month, then €0.01/GB** |
| Intra-region transfer (AMS↔AMS) | free |
| Inter-region transfer (AMS↔PAR) | billed as egress |
| Free trial | 750 GB Standard storage for 90 days (new users only), no egress trial |

Billing granularity is "a billable unit started is due": 1 KB for 1 s bills as 1 GB-hour. Irrelevant at our volumes.

### Estimate: ~5 GB tiles + backups

| Line | Size | Class | €/month |
|---|---|---|---|
| Netherlands vector tiles (PMTiles) | 5 GB | Standard Multi-AZ | 0.08 |
| Nightly Postgres dumps, 30 retained | ≈ 6 GB | Standard One Zone | 0.05 |
| Route-handoff objects + weekly graph archive | ≈ 10 GB | Standard One Zone | 0.08 |
| **Storage subtotal** | ≈ 21 GB | | **≈ 0.21** |

Egress scenarios (tiles fetched straight from the bucket by the app; a vector-tile session for a Dutch Route plan
is assumed at ≈ 30 MB/rider/month, which is a guess **[unverified]**):

| Scenario | Riders/month | Egress | Billable | €/month |
|---|---|---|---|---|
| Launch | 1,000 | 30 GB | 0 (under 75 GB free) | **0.00** |
| Growth | 10,000 | 300 GB | 225 GB | **2.25** |
| Busy | 33,000 | 1 TB | 925 GB | **9.25** |

Backups written from the instance to a bucket in the same region are intra-region, so free; nightly dumps don't
touch the egress allowance.

### Is serving PMTiles from the bucket viable?

Yes, technically:

- PMTiles needs HTTP `Range` GETs. Scaleway's object-operations reference shows `Accept-Ranges: bytes` on GET/HEAD
  responses (<https://www.scaleway.com/en/docs/object-storage/api-cli/object-operations/>) and the performance guide
  says "Use the `Range` header in `GET` requests (both for multipart and non-multipart objects)"
  (<https://www.scaleway.com/en/docs/object-storage/reference-content/optimize-object-storage-performance/>).
- CORS configuration is supported (same object-operations reference), required for a browser/WebView client.
- Bucket website hosting exists and a CNAME can point a custom hostname at the bucket
  (<https://www.scaleway.com/en/docs/object-storage/how-to/use-bucket-website/>). **[unverified]** whether a CNAME
  works over HTTPS; the bucket's certificate will not cover a custom name, so custom-domain TLS almost certainly
  requires Edge Services (section 4) or a reverse proxy on the instance.

Cheap? Cheap in storage (pennies) and free up to 75 GB/month egress; after that €0.01/GB. The comparison is against
the instance, whose egress is already paid for: at 300 GB/month a tile server on the instance costs €0 extra as long as
the instance has the CPU and the 200–400 Mb/s cap is not saturated (300 GB/month ≈ 1 Mb/s average, so it is not).
Recommendation: **start by serving the PMTiles from the instance** (e.g. `pmtiles serve` or Caddy with range support,
fronted by the same reverse proxy as the API), keep the authoritative copy in Object Storage, and move serving to the
bucket (+ Edge Services) only if tile traffic starts competing with the routing engine for CPU.

## 3. Managed Postgres and Serverless SQL vs self-hosted

Source: <https://www.scaleway.com/en/pricing/managed-databases/> (Paris; Managed Database for PostgreSQL is
region-wide in nl-ams per the availability page; Serverless SQL Database is **Paris only**).

| Option | Compute | Storage | Backups | ≈ €/month (10 GB DB) |
|---|---|---|---|---|
| Managed `DB-DEV-S` (2 vCPU, 2 GB) | €0.0156/h = €11.39 | 5K: €0.0993/GB/month → €0.99 | €0.03/GB/month → €0.30 | **≈ 12.70** |
| Managed `DB-PLAY2-PICO` (1 vCPU, 2 GB) | €0.0233/h = €17.01 | €0.99 | €0.30 | ≈ 18.30 |
| Managed `DB-DEV-M` (3 vCPU, 4 GB) | €0.0382/h = €27.89 | €0.99 | €0.30 | ≈ 29.20 |
| Serverless SQL, always warm (min 1 vCPU) | €0.13752/vCPU/h = €100.39 | €0.000272/GB/h = €0.199/GB → €1.99 | 7 days free | ≈ 102 |
| Serverless SQL, scale-to-zero (min 0 vCPU) | pay per active second, 5-min minimum per wake-up | €1.99 | free | **unpredictable**; ≈ €10 at 10 % duty, cold starts of "a few seconds" |
| **Self-hosted in Docker on the instance** | 0 (shares the instance) | 0 (shares the block volume) | dumps to One Zone ≈ €0.05 | **≈ 0.05** |

Confirmed: **self-hosted Postgres is the right call under €50**. Managed `DB-DEV-S` would eat 25–30 % of the budget
and only fits alongside the ARM 8 GB instance (€25.19 + €12.70 + storage + IP ≈ €47). Serverless SQL is ruled out
twice over: wrong region for an `nl-ams` instance (cross-region latency on every query, and the availability page lists
it as "Not available" in Amsterdam) and the warm price is 2× the whole budget. Revisit managed Postgres only if
the budget rises or ops time becomes the scarcer resource; the Terraform resource (`scaleway_rdb_instance`) exists
either way, so the switch is a config change.

## 4. Egress, CDN and Edge Services

- **Instance egress is included** in the instance price ("straightforward pricing without egress fees"; "List prices
  include egress", <https://www.scaleway.com/en/pricing/virtual-instances/>). The only limit is the per-type bandwidth
  cap (200 Mb/s on `BASIC2-A2C-8G`, 400 Mb/s on `DEV1-L`).
- **Object Storage egress**: 75 GB/month free, then €0.01/GB (section 2).
- **Edge Services** (<https://www.scaleway.com/en/pricing/network/#edge-services>,
  <https://www.scaleway.com/en/docs/edge-services/reference-content/understanding-pricing/>) is Scaleway's CDN/WAF
  layer. It fronts an Object Storage bucket or a Load Balancer (Serverless Containers/Functions via CLI/Terraform
  only); it does **not** front a bare Instance, so putting the API behind it would also need a Load Balancer
  (`LB-S` €0.023/h ≈ €16.79/month, out of budget).

| Plan | €/month | Pipelines | Cache egress included | WAF | Extra cache egress |
|---|---|---|---|---|---|
| Starter | 0.99 | 1 | 100 GB | +€4/month add-on | €0.0135/GB |
| Professional | 12.99 | 10 | 1 TB | included (5M req) | €0.0135/GB |
| Advanced | 109.99 | 100 | 10 TB | included (50M req) | €0.0135/GB |

  Extra pipeline €4/month pro rata. Only cache→client bytes count; origin→cache fills do not. Free Let's Encrypt
  TLS on a custom **subdomain** (root domains are not supported). Edge Services is listed as a Paris-region product,
  but the pipeline wizard lets you "Select the Region of the bucket" and the docs show an `nl-ams` bucket endpoint, so
  it can front an Amsterdam bucket. **[unverified]** whether the origin fill from an `nl-ams` bucket to the Paris-hosted
  cache is billed as inter-region Object Storage egress (€0.01/GB after the free 75 GB); at a 5 GB tile set with a
  reasonable TTL this would be a few cents either way.
- Verdict: Edge Services is worth its €0.99 **for custom-domain TLS and caching on a public tile bucket**, not for
  saving egress money: its overage (€0.0135/GB) is dearer than raw bucket egress (€0.01/GB). It becomes cheaper than
  the bucket only inside the included 100 GB. For v1, a reverse proxy on the instance covers TLS for both API and
  tiles at €0.
- **Domains & DNS**: €0.007/h per zone ≈ €5.11/month (<https://www.scaleway.com/en/pricing/network/>). Hosting the zone
  at the registrar or another free DNS avoids this; the BoM lists it as optional.

## 5. Terraform / OpenTofu provider maturity

Sources: Terraform Registry API `https://registry.terraform.io/v1/providers/scaleway/scaleway`, GitHub
`scaleway/terraform-provider-scaleway` via `gh api`, OpenTofu registry API `https://api.opentofu.org/registry/docs/providers/scaleway/scaleway/index.json`, all 2026-10-21.

- Latest **v2.87.0, published 2026-10-16**; tier **partner-premier**; 16.6 M downloads; repo created 2017-06, last push
  2026-10-20, 263 stars, 199 open issues. Release cadence over the last five weeks: 2.83.1 (09-16), 2.84.0 (10-01),
  2.85.0 (10-06), 2.86.0 (10-07), 2.87.0 (10-16), i.e. roughly weekly.
- Built on `terraform-plugin-sdk/v2` v2.40.1 **and** `terraform-plugin-framework` v1.19.0 (mid-migration, normal for
  a provider this age). `required_version = ">= 0.13"`.
- **OpenTofu**: the OpenTofu registry mirrors the provider with v2.87.0 as current; no OpenTofu-specific caveats found.
- 162 resource docs (`docs/resources/`). Every product in this BoM has a resource: `scaleway_instance_server`,
  `scaleway_instance_ip`, `scaleway_instance_ip_reverse_dns`, `scaleway_block_volume`, `scaleway_block_snapshot`,
  `scaleway_object_bucket` (+ `_policy`, `_acl`, `_website_configuration`, `_lock_configuration`, SSE),
  `scaleway_rdb_instance`, `scaleway_sdb_sql_database`, `scaleway_edge_services_plan` / `_pipeline` / `_backend_stage` /
  `_cache_stage` / `_tls_stage` / `_dns_stage` / `_waf_stage` / `_route_stage`, `scaleway_vpc`, `scaleway_vpc_private_network`,
  `scaleway_domain_record`, `scaleway_secret`, `scaleway_iam_*`.
- Verdict: **mature enough to be the only provisioning path**; no need for click-ops fallbacks.

## 6. EU-owned alternatives (comparison only)

Observed 2026-10-21, EUR excl. VAT.

**Hetzner Cloud** (<https://www.hetzner.com/cloud/cost-optimized/>, <https://www.hetzner.com/cloud/regular-performance/>,
<https://www.hetzner.com/cloud/general-purpose/>, <https://www.hetzner.com/storage/object-storage/>, <https://docs.hetzner.com/cloud/servers/overview/>).
Locations: Falkenstein, Nuremberg, Helsinki (plus US/SG); **no Netherlands**.

| Type | vCPU | RAM | NVMe | Traffic incl. | €/month | Status on page |
|---|---|---|---|---|---|---|
| CX33 (x86 shared) | 4 | 8 GB | 80 GB | 20 TB | 8.99 | "currently unavailable" |
| CAX21 (ARM shared) | 4 | 8 GB | 80 GB | 20 TB | 10.99 | "currently unavailable" |
| CX43 | 8 | 16 GB | 160 GB | 20 TB | 16.49 | "currently unavailable" |
| CAX31 (ARM) | 8 | 16 GB | 160 GB | 20 TB | 21.49 | "currently unavailable" |
| CX53 | 16 | 32 GB | 320 GB | 20 TB | 29.99 | "currently unavailable" |
| CPX32 (x86 shared, newer) | 4 | 8 GB | 160 GB | 20 TB | 35.99 | "currently unavailable" |
| CCX13 (dedicated) | 2 | 8 GB | 80 GB | 20 TB | 43.49 | available |
| CCX23 (dedicated) | 4 | 16 GB | 160 GB | 20 TB | 86.49 | available |

Primary IPv4 €0.50/month; traffic overage €1.00/TB; Object Storage €6.49/month flat including 1 TB storage and 1 TB
egress. Volumes price not found on a primary page **[unverified]**.

**OVHcloud** (<https://www.ovhcloud.com/nl/vps/>, <https://www.ovhcloud.com/en-ie/public-cloud/prices/>). VPS "from"
prices carry `pricing=upfront12` in the order links, i.e. 12-month upfront; no-commitment monthly price not shown
**[unverified]**: VPS-2 4 vCPU/8 GB/75 GB €7.21; VPS-3 6 vCPU/12 GB €10.40; VPS-4 8 vCPU/24 GB €19.96, unlimited
traffic. Public Cloud (hourly, no commitment): `d2-8` 8 GB €20.60/month; `b3-8` 2 vCPU/8 GB €88.04, `b3-16` €176.00,
`b3-32` €352.01 (B3 figures came from a page JSON entry labelled `"os":"windows"` **[unverified for Linux]**).
Regions GRA/SBG/RBX (FR), DE1, WAW1, UK1; **no Netherlands**.

**Exoscale**: pricing page renders prices client-side; no figures could be captured **[unverified]**. Zones CH/AT/DE/BG,
no Netherlands.

Comparison paragraph: on raw price per GB of RAM Hetzner is 2–3× cheaper than Scaleway (16 GB for €16.49 vs €50.30,
32 GB for €29.99 vs €100.59) and bundles 20 TB of traffic and local NVMe, which would turn the "8 GB only" constraint
into a comfortable 16–32 GB box with €20+ headroom. OVH's committed VPS tier is similar or cheaper still. Both,
however, lack an Amsterdam location (nearest are Falkenstein/Nuremberg at ~400 km and Gravelines/Roubaix at ~250 km;
latency to Dutch riders would be ~10 ms worse, which matters little for a Route plan request and not at all for
Turn-by-turn navigation that runs on-device), and on the observation date Hetzner's whole shared-vCPU catalogue
was marked unavailable. Scaleway buys `nl-ams` residency, a public catalog API, and a first-party CDN/WAF; Hetzner buys
roughly double the RAM for the same money. If the routing-engine ticket lands above ~5 GB resident, Hetzner CX43/CAX31
is the obvious escape hatch and should be re-checked for stock at that point.

## 7. Costed bills of materials

Common assumptions: `nl-ams-1`; one Block Storage 5K volume; one retained snapshot of the full volume; one flexible
IPv4; Object Storage per section 2 at the launch egress scenario (€0 egress). Optional lines not in the totals: Scaleway
DNS zone €5.11, Edge Services Starter €0.99, Managed `DB-DEV-S` €12.70.

### 8 GB (fits)

| Line | ARM: `BASIC2-A2C-8G` | x86: `DEV1-L` |
|---|---|---|
| Instance | 25.19 | 31.27 |
| Block Storage 5K, 60 GB (OS + Docker + ~5 GB graph + Postgres + 5 GB tiles) | 5.69 | 5.69 |
| Snapshot, 60 GB retained | 2.15 | 2.15 |
| Flexible IPv4 | 2.92 | 2.92 |
| Object Storage ≈ 21 GB, egress inside free tier | 0.21 | 0.21 |
| **Total** | **36.16** | **42.24** |
| **Headroom to €50** | **13.84** | **7.76** |
| Headroom at 10,000-rider bucket egress (+2.25) | 11.59 | 5.51 |

### 16 GB (does not fit)

| Line | ARM: `BASIC2-A4C-16G` | x86: `GP1-XS` | x86: `PRO2-XS` |
|---|---|---|---|
| Instance | 50.30 | 67.76 | 81.91 |
| Block Storage 5K, 80 GB | 7.59 | 7.59 | 7.59 |
| Snapshot, 80 GB | 2.86 | 2.86 | 2.86 |
| Flexible IPv4 | 2.92 | 2.92 | 2.92 |
| Object Storage | 0.21 | 0.21 | 0.21 |
| **Total** | **63.88** | **81.34** | **95.49** |
| Over €50 by | 13.88 | 31.34 | 45.49 |

### 32 GB (does not fit)

| Line | ARM: `BASIC2-A8C-32G` | x86: `GP1-S` | x86: `PRO2-S` |
|---|---|---|---|
| Instance | 100.59 | 139.24 | 163.07 |
| Block Storage 5K, 120 GB | 11.39 | 11.39 | 11.39 |
| Snapshot, 120 GB | 4.30 | 4.30 | 4.30 |
| Flexible IPv4 | 2.92 | 2.92 | 2.92 |
| Object Storage | 0.21 | 0.21 | 0.21 |
| **Total** | **119.41** | **158.06** | **181.89** |

## 8. Recommendation

**`BASIC2-A2C-8G` in `nl-ams-1`, ≈ €36/month, ≈ €14 headroom**, with `DEV1-L` (≈ €42, ≈ €8 headroom) as the x86
fallback if any v1 component turns out not to ship arm64 images.

Conditions and risks, in order of importance:

1. **RAM budget.** 8 GB total means roughly: routing engine ≤ 5 GB resident, Postgres ≈ 0.5 GB, API + tile server
   ≈ 0.5 GB, OS/Docker ≈ 1 GB, ~1 GB slack. The routing-engine ticket must report the actual resident size for a
   Netherlands graph with Bromfiets/Snorfiets/Fiets Vehicle class profiles. GraphHopper with CH/LM for three
   profiles, or Valhalla with NL tiles, is plausibly inside this; a 12 GB figure is not, and no Scaleway shape under
   €50 fixes that.
2. **CPU for graph builds.** Two shared Ampere vCPUs are fine for serving but slow for importing. Build the graph in CI
   or on a laptop and upload the artefact to Object Storage; the instance only downloads it (intra-region, free).
3. **ARM compatibility.** GraphHopper is JVM and runs on arm64; Valhalla's official Docker images and the TypeScript API
   are multi-arch in general but this has **not** been verified for the specific images we will pin **[unverified]**.
4. **Single-zone ARM.** `BASIC2-A` exists only in `nl-ams-1`; there is no second AMS zone to move to. Acceptable for v1
   (one instance anyway), but the `DEV1-L` fallback has `nl-ams-2` too.
5. **Headroom use.** The €14 covers: Edge Services Starter (€0.99) when the tile bucket goes public, growth-scenario
   egress (€2.25), DNS at Scaleway (€5.11) if wanted, and a second snapshot generation. It does **not** cover managed
   Postgres on top of `DEV1-L`.

## Open questions surfaced

- What is the measured resident memory of the Netherlands graph per engine (GraphHopper vs Valhalla) with three Vehicle
  class profiles? This decides whether 8 GB is viable at all.
- Do the chosen routing-engine and tile-server images publish arm64 builds?
- Tile traffic per rider per month: the 30 MB assumption needs a measurement from a prototype.
- Is Edge Services origin fill from an `nl-ams` bucket billed as inter-region Object Storage egress?
- Does Hetzner's shared-vCPU shortage persist? If a 16 GB floor is needed, re-check CX43/CAX31 stock and decide
  whether `nl-ams` residency is a hard requirement.

## Source index

| Topic | URL | Observed |
|---|---|---|
| Instance catalog + prices per zone | `https://api.scaleway.com/instance/v1/zones/{zone}/products/servers` | 2026-10-21 09:20 UTC |
| Instance stock | `https://api.scaleway.com/instance/v1/zones/{zone}/products/servers/availability` | 2026-10-21 |
| Instance pricing page | <https://www.scaleway.com/en/pricing/virtual-instances/> | 2026-10-21 |
| Instance datasheet (arch, shared/dedicated, bw) | <https://www.scaleway.com/en/docs/instances/reference-content/instances-datasheet/> | 2026-10-21 |
| Instance bandwidth caps | <https://www.scaleway.com/en/docs/instances/reference-content/instances-bandwidth-overview/> | 2026-10-21 |
| Instances FAQ (billing, IPv4 €0.004/h) | <https://www.scaleway.com/en/docs/instances/faq/> | 2026-10-21 |
| Product availability by region | <https://www.scaleway.com/en/docs/account/reference-content/products-availability/> | 2026-10-21 |
| Block + Object Storage pricing | <https://www.scaleway.com/en/pricing/storage/> | 2026-10-21 |
| Object Storage FAQ | <https://www.scaleway.com/en/docs/object-storage/faq/> | 2026-10-21 |
| Object operations (Range, CORS) | <https://www.scaleway.com/en/docs/object-storage/api-cli/object-operations/> | 2026-10-21 |
| Object Storage performance (Range) | <https://www.scaleway.com/en/docs/object-storage/reference-content/optimize-object-storage-performance/> | 2026-10-21 |
| Bucket website | <https://www.scaleway.com/en/docs/object-storage/how-to/use-bucket-website/> | 2026-10-21 |
| Managed Databases + Serverless SQL pricing | <https://www.scaleway.com/en/pricing/managed-databases/> | 2026-10-21 |
| Serverless SQL FAQ | <https://www.scaleway.com/en/docs/serverless-sql-databases/faq/> | 2026-10-21 |
| Network pricing (IP, LB, Edge Services, DNS) | <https://www.scaleway.com/en/pricing/network/> | 2026-10-21 |
| Edge Services pricing rules | <https://www.scaleway.com/en/docs/edge-services/reference-content/understanding-pricing/> | 2026-10-21 |
| Edge Services quickstart / pipeline | <https://www.scaleway.com/en/docs/edge-services/quickstart/>, <https://www.scaleway.com/en/docs/edge-services/how-to/create-pipeline/> | 2026-10-21 |
| Savings plans | <https://www.scaleway.com/en/docs/billing/additional-content/understanding-savings-plans/> | 2026-10-21 |
| Terraform provider | <https://registry.terraform.io/providers/scaleway/scaleway/latest>, <https://github.com/scaleway/terraform-provider-scaleway> | 2026-10-21 |
| OpenTofu registry | <https://search.opentofu.org/provider/scaleway/scaleway> | 2026-10-21 |
| Hetzner Cloud | <https://www.hetzner.com/cloud/cost-optimized/>, <https://www.hetzner.com/cloud/regular-performance/>, <https://www.hetzner.com/cloud/general-purpose/>, <https://www.hetzner.com/storage/object-storage/>, <https://docs.hetzner.com/cloud/servers/overview/> | 2026-10-21 |
| OVHcloud | <https://www.ovhcloud.com/nl/vps/>, <https://www.ovhcloud.com/en-ie/public-cloud/prices/> | 2026-10-21 |
| Exoscale | <https://www.exoscale.com/pricing/> | 2026-10-21 (prices not rendered) |
