---
date: 2026-08-19
title: "The product layer between a roof model and a Czech bill"
description: "Google Solar API, a 6-decimal cache, GeoTIFF archives, a self-sufficiency heuristic, and NZÚ math that had to be deleted when the household program stopped."
tags:
  - Next.js
  - Google Solar API
  - Supabase
  - product
  - Czech
---

A satellite roof model is not a quote. Google’s Solar API (Building Insights plus Data Layers) returns panel layouts and DC production for a lat/lng. A Czech homeowner still needs Kč, heating, a subsidy regime that can vanish, and a reason to believe the percentage on the gauge. The layer between those two things is a product, not a model.

The site is [SolarKalkulačka.cz](https://solarkalkulacka.cz). Addresses come from Places (`includedRegionCodes: ["cz"]`). Analyses live in Supabase. I wrote the Czech domain math, the cache, the lead flow, and the hourly simulation. Google owns the roof geometry. Next.js is the frame.

This post focuses on that product layer: cache economics, conservative heuristics, and regulation as a breaking change. Conversion rates, installer marketplaces, and certified energy yields are out of scope because they are not in the repository. I will not invent them.

I explicitly mention “heuristic” for the self-sufficiency gauge because the number is the most tempting place to overclaim. Annual DC kWh from Google is a measurement of a sort. “Your house will be 73% independent” is a model I owe the user an explanation for.

## Two users, one billed API

A homeowner curious about rooftop PV is stuck between a salesman quote they cannot check and a generic “kWp × 1 000 kWh” calculator that does not know whether the roof faces a courtyard.

An installer wants a lead that already contains an address, a bill, heating type, and a first-pass system size.

Those are two products glued together. The first has to be conservative enough to read out loud to an electrician. The second has to take personal data, which means CSRF, a CAPTCHA, and Zod, not a mailto link.

Google Solar is not a free geocoder. Building Insights is one SKU. Data Layers — flux, mask, RGB, hourly shade GeoTIFFs — is the expensive one. If every revisit of the same house re-bills Google, the calculator ends when the credit card statement arrives.

## Design patterns

### Pattern 1: The cache key is a point on Earth

A first version that compared raw floats on `lat` / `lng` missed cache hits. Two Places picks of the same building are not bit-identical. Commit `0421e5a` (2026-01-16) stopped pretending they were.

```ts
const roundedLat = Math.round(lat * 1000000) / 1000000;
const roundedLng = Math.round(lng * 1000000) / 1000000;
const coordKey = `${roundedLat.toFixed(6)},${roundedLng.toFixed(6)}`;
```

Six decimals is about ten centimetres: finer than a roof, coarse enough that two geocodes of the same house collapse. A unique index on `coord_key` makes a second insert fail closed.

A JSON-only cache would still be wrong. Data Layers URLs expire, so a JSON “hit” would immediately re-fetch billed rasters. A miss therefore does two Google calls in parallel, then archives:

```ts
const [insightsRes, layersRes] = await Promise.all([
  fetch(`${SOLAR_API_URL}?location.latitude=${roundedLat}&...`),
  fetch(`${LAYERS_API_URL}?location.latitude=${roundedLat}&radius_meters=25&view=FULL_LAYERS&...`),
]);
```

`saveGeoTiffToStorage` pulls each Google URL with a 30-second abort and a 50 MB cap, uploads the TIFF, and asks `sharp` for a WebP at quality 80. The inserted row keeps `address` / `lat` / `lng` null and keys only on `coord_key`. The person is not the cache key. The point on Earth is.

I do not have a saved Google invoice in this repo, so I will not publish a dollar figure. The archive is the evidence: if Data Layers were cheap I would not have written it.

The analyze route also rate-limits (Upstash, 10 requests / minute), caps the body at 10 KB, refuses a missing CSRF pair with `timingSafeEqual`, and rejects anything outside a Czech bounding box. Coverage holes become “Pro tuto lokaci nemáme data,” not a guessed kWh. `requiredQuality=MEDIUM` so MEDIUM and HIGH both pass. `radius_meters=25` so Data Layers cover one house rather than the neighbour’s roof.

Consent first. `allowSolarCache` starts `false`. Cache writes stay off until that click.

{{< responsive-image src="images/solar-cache-flow.png" alt="Address to six-decimal cache key to hit or Google miss" maxWidth="720px" >}}

*The person is not the cache key. The point on Earth is.*


### Pattern 2: Let Google own the layout; pick the nearest config

Google’s useful gift is a list of `solarPanelConfigs`: panel counts and `yearlyEnergyDcKwh` for layouts that fit the roof. I do not invent a layout. I pick the nearest config to a consumption target.

Retail power is **6.5 Kč/kWh**, labelled conservative. Yearly kWh from the bill is `(monthlyBill × 12) / 6.5`. Heating is an adder: an electric boiler dumps in 2,500 kWh; a heat pump divides thermal demand by a COP of 3.4 / 3.0 / 2.8. Gas adds zero electric kWh. That last sentence is why the form asks.

System losses are 14%. Export is 30% of retail. Installed cost is a tier table (45 / 40 / 35 / 32 thousand Kč per kWp). Batteries are 15,000 Kč/kWh. ROI walks 25 years with 0.5% degradation, 3% inflation, 500 Kč/kWp O&M, and an inverter replacement in year 12. Planning constants. Not a quote.

```ts
const targetProductionAcKwh = targetConsumptionKwh * 0.9;
const targetProductionDcKwh = targetProductionAcKwh / (1 - CZ_CONFIG.SYSTEM_LOSS_RATE);
// pick config minimizing |yearlyEnergyDcKwh − targetProductionDcKwh|
```

Target 90% of consumption, lift it back to DC so it matches Google, take the closest array. Not the biggest roof-filling array. A 20 kWp palace on a 4,000 kWh house would make production look heroic and self-consumption look like fiction.

In the browser, `geoTiff.ts` turns GeoTIFF keys into a proj4 definition so the bounding box projects to WGS84. `FluxOverlay` paints annual flux and drops it on the satellite map. Panel polygons come from Google’s `solarPanels` list.

{{< responsive-image src="images/solar-homepage.png" alt="SolarKalkulačka homepage with partner-search email" maxWidth="800px" >}}

*Live homepage, August 2026. The honest CTA is an email, not a carousel of Partner A. I did not capture a flux overlay without walking an analysis in the browser.*


The homepage sample is a real cached row: `50.036017,14.326416` (Nad Mušlovkou 918, Praha-Řeporyje). If that row disappears, the sample 404s. That is the correct failure.

### Pattern 3: Cap the number you most want to inflate

**Energetická soběstačnost** is a heuristic. It is not hourly load matching. The comments in the function are the spec:

```ts
if (!hasBattery || batterySizeKwh <= 0) {
  if (loadRatio <= 1) return 0.30 * loadRatio;
  return Math.min(0.30 + 0.1 * Math.log10(loadRatio), 0.40);
}
// with battery: 60% of daily kWh as evening, then cap at 90%
return Math.min(baseSelfSufficiency + batteryBonus, 0.90);
```

No battery: the 30–40% band, because daytime-only self-use is not independence. With a battery: evening/night is assumed to be 60% of daily kWh, then clipped, then capped at 90% because January in Czechia exists. The gauge does not read Google’s hourly shade or a load profile. Annual savings use the same ratio.

There is a separate hourly cartoon at `/simulace`: a 24-hour irradiance curve, seasonal sunrise, three consumption patterns. It is a teaching tool. It does not feed the gauge.

Monthly bars spread annual production across twelve PVGIS-shaped multipliers (December 0.30, June 1.40). I did not pull Google’s `monthlyFluxUrl`. The shape is Czech-latitude honest. It is not twelve independent measurements.

{{< responsive-image src="images/solar-gauge.png" alt="Self-sufficiency gauge capped at 90 percent as a heuristic" maxWidth="720px" >}}

*A closed-form band, not hourly load matching. /simulace does not feed this number.*


## Case study: the law as a feature

I implemented Nová zelená úsporám as if it were a function: base + per kWp + battery bonus + wallbox + a 10% region bump, then a 50% cost cap. NZÚ Light, Oprav dům po babičce, and RES+ for firms sat next to it.

On 12 April 2026, in commit `518a6fa`, I removed the default household path.

```ts
return {
  amount: 0,
  programId: "nzu_cancelled_2026",
  programName: "NZÚ 2026 pozastaveno",
  notes: "Dotace NZÚ jsou v roce 2026 pozastaveny. Sledujte aktuality na novazelenausporam.cz.",
};
```

The badge is amber, not green. The PDF lost the dotace line. The landing page stopped promising a household grant I could no longer defend.

I did not delete every formula. `calculateNzuStandardSubsidy` is dead. NZÚ Light and RES+ still have branches if someone checks leftover boxes. That is incomplete cleanup, not a secret second subsidy. The product-facing default for a household is zero. When a regulation is a feature, shipping the deletion is the feature.

## What exists, and what does not

A calculator that cannot accept a name is a demo. `POST /api/leads` checks CSRF, a 5/minute Redis limit, Cloudflare Turnstile (fail-closed in production), then a Zod schema. Drafts get a session token so a later PATCH cannot wander into someone else’s lead.

I also built the UI: a three-step form, masked Partner A/B/C cards, `/dekujeme`. Installer matching reads `data/installers.json`. Those entries are placeholders — Firma Solaris at +420 123 456 789. They are not a partner network.

On 27 January 2026 I unmounted the form and the installer cards from `/analyza` and put a simulation link there instead. The APIs are still in the tree. The homepage now says we are looking for installation partners. That is the true sentence.

PDF is the same kind of leftover. `pdfExport.ts` exists (`0524c6b`, 17 January 2026). The download button is `disabled={true}` and labelled Premium. An old `PROJECT.md` still has an empty checkbox for PDF export. The checkbox is wrong. The lock is real.

| Exists | Does not |
|---|---|
| Live Czech address → Google Solar roof model | Certified yield or hourly load-match |
| 6-decimal cache + GeoTIFF/WebP archive | Conversion rate, lead volume, revenue |
| Heuristic self-sufficiency gauge, capped at 90% | A real installer network |
| `/simulace` not wired into the gauge | Proof leftover NZÚ Light should still be on the form |
| Lead API with CSRF / Turnstile / Zod; forms off `/analyza` | |
| Household subsidy default 0 Kč after NZÚ 2026 | |

## Implications

Satellite roof data is a gift if you keep it in DC kWh and let Google own the layout. The moment you turn it into independence, you have left the API and entered a model.

The cache is the product economics. Rounding, a string key, a unique index, and archived rasters are less glamorous than a heatmap, and they are why I can leave the heatmap online.

Regulation is a breaking change with no deprecation window. Implementing NZÚ carefully was correct. Deleting the default path when the household program stopped was also correct. Leaving the helper functions in the file is a reminder that “removed from the UI” and “gone from the codebase” are different commits.

I still want the lead form back — after there is someone real to receive the row. Until then the honest CTA is an email address, not a carousel of Partner A.

The useful test is not “does the gauge animate.” It is “can I read `czSolarMath.ts` out loud to a sceptical electrician and not flinch.” The 90% cap and the cancelled-subsidy banner are there for the days the answer is no.
