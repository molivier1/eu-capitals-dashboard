# TD1 — EU capitals sustainable vacation dashboard

*Student manual. This is the same technical build as
`TD_1_DELIVERY_MANUAL_eu_capitals_dashboard.md`, reorganized around the stages of a
BI pipeline instead of raw click order, with the reasoning kept in and the
instructor-only verification/troubleshooting stripped out. If something breaks and
you can't figure out why after re-reading the relevant step, check that manual's
Troubleshooting section or ask your instructor — it's not repeated here.*

**Before you start:** create a file called `REFLECTIONS.md` at the root of your
repository. Every highlighted **Reflection** box below asks a short question (2-4
sentences is enough) — answer it in that file, under a heading matching the step. The
consolidated list is at the end of this document so you can check you didn't miss
one. This file is part of what you submit.

**Everything below happens in a web browser on github.com.** No local install of
git, Python, or anything else is required. Estimated time: 30-40 minutes for the
guided build, plus 10-20 minutes for the unguided extension at the end.

---

## The decision this dashboard supports

> Which EU capital offers the best vacation conditions next week while minimizing the
> CO₂ emissions of getting there from Paris?

This is CM1's argument made concrete: a dashboard is only worth building if it
supports a real, statable decision, for a real person, at the moment they need it.
"Show me some data about EU capitals" is not that. "Tell me where to go next week,
weighing two things I actually care about" is.

> **Reflection — before you build anything:** if you personally had to choose a
> destination, how would you weigh good weather against low CO₂? Would you ignore
> one of them entirely? Write 1-2 sentences now — you'll compare this to what the
> dashboard actually computes later in this TD.

---

## What you're building

```
eu-capitals-dashboard/
  .github/workflows/update-data.yml   # the scheduled Action
  data/
    capitals.csv                       # static reference input (not published)
  scripts/
    fetch_and_score.py                 # the pipeline the Action runs
  docs/                                 # GitHub Pages source
    index.html
    data/
      rankings.json                     # generated + committed by the Action
  README.md
  REFLECTIONS.md
```

A live, self-updating pipeline, not a one-off report: a GitHub Action fetches real
weather data on a schedule, scores every destination, and commits the result; a
static page reads that result and renders it. Nobody re-runs anything by hand — that
automation, not the first manual run, is the actual point (CM4).

## Repository setup (do this once)

1. **New repository** on github.com, name it `eu-capitals-dashboard`, set it
   **Public** (GitHub Pages on a free account requires this).
2. **Settings → Actions → General → Workflow permissions → "Read and write
   permissions" → Save.** The scheduled Action needs this to commit its own results
   back to the repo — skipping it is the single most common reason this kind of
   project fails on its first run.

**Checkpoint:** an empty repo with a `README.md`, and workflow permissions saved.

---

## Stage 1 — Acquisition: a static reference table and a live weather API

*Connects to CM3 — open/downloadable data and API acquisition patterns.*

Two different acquisition patterns, both real, both in this one project:

**A static reference table**, committed once and never re-fetched — the 27 EU
capitals' coordinates don't change day to day, so there's no reason to hit an API
for them every run.

Create `data/capitals.csv`:

```csv
country,capital,lat,lon
Austria,Vienna,48.2082,16.3738
Belgium,Brussels,50.8503,4.3517
Bulgaria,Sofia,42.6977,23.3219
Croatia,Zagreb,45.8150,15.9819
Cyprus,Nicosia,35.1856,33.3823
Czechia,Prague,50.0755,14.4378
Denmark,Copenhagen,55.6761,12.5683
Estonia,Tallinn,59.4370,24.7536
Finland,Helsinki,60.1699,24.9384
France,Paris,48.8566,2.3522
Germany,Berlin,52.5200,13.4050
Greece,Athens,37.9838,23.7275
Hungary,Budapest,47.4979,19.0402
Ireland,Dublin,53.3498,-6.2603
Italy,Rome,41.9028,12.4964
Latvia,Riga,56.9496,24.1052
Lithuania,Vilnius,54.6872,25.2797
Luxembourg,Luxembourg City,49.6116,6.1319
Malta,Valletta,35.8989,14.5146
Netherlands,Amsterdam,52.3676,4.9041
Poland,Warsaw,52.2297,21.0122
Portugal,Lisbon,38.7223,-9.1393
Romania,Bucharest,44.4268,26.1025
Slovakia,Bratislava,48.1486,17.1077
Slovenia,Ljubljana,46.0569,14.5058
Spain,Madrid,40.4168,-3.7038
Sweden,Stockholm,59.3293,18.0686
```

(Coordinates are city-centre approximations — adequate here. Note: the Netherlands'
seat of government is The Hague, but Amsterdam is the conventional choice for a list
like this.)

**A live API**, hit fresh on every scheduled run — because weather, unlike
coordinates, genuinely changes. [Open-Meteo](https://open-meteo.com) is free,
open-source, and needs no key or account, which matters for a script a scheduled
Action calls unattended: there's no secret to store or leak.

Create `scripts/fetch_and_score.py`:

```python
import csv, json, math, time
import requests

PARIS_LAT, PARIS_LON = 48.8566, 2.3522
EMISSIONS_FACTOR_KG_PER_KM = 0.15  # DEFRA-style short-haul factor -- verify/update

def haversine_km(lat1, lon1, lat2, lon2):
    r = 6371
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlambda = math.radians(lon2 - lon1)
    a = math.sin(dphi / 2) ** 2 + math.cos(phi1) * math.cos(phi2) * math.sin(dlambda / 2) ** 2
    return 2 * r * math.asin(math.sqrt(a))

def fetch_weather_batch(lats, lons, retries=4, backoff=5):
    """One request for every destination at once, instead of one connection per
    capital. This isn't just fewer lines -- it fixes a real failure mode: 26
    separate TLS handshakes from a shared GitHub Actions runner IP is 26 separate
    chances for a connection to hang or get throttled, and in testing it reliably
    failed partway through (a handful of capitals fetched fine, then the rest
    started timing out on every retry). One request means one connection, and it
    is also roughly 100x faster in practice (all 26 forecasts typically arrive in
    well under a second)."""
    url = "https://api.open-meteo.com/v1/forecast"
    params = {
        "latitude": ",".join(str(v) for v in lats),
        "longitude": ",".join(str(v) for v in lons),
        "daily": "temperature_2m_max,precipitation_sum,sunshine_duration",
        "timezone": "auto", "forecast_days": 7,
    }
    last_error = None
    for attempt in range(1, retries + 1):
        try:
            resp = requests.get(url, params=params, timeout=(15, 30))
            resp.raise_for_status()
            data = resp.json()
            return data if isinstance(data, list) else [data]
        except requests.exceptions.RequestException as e:
            last_error = e
            print(f"  batch weather fetch attempt {attempt}/{retries} failed: {e}")
            if attempt < retries:
                time.sleep(backoff * attempt)
    raise RuntimeError(f"Open-Meteo unreachable after {retries} attempts") from last_error

def normalize(values, invert=False):
    lo, hi = min(values), max(values)
    if hi == lo:
        return [0.5 for _ in values]
    scaled = [(v - lo) / (hi - lo) for v in values]
    return [1 - s for s in scaled] if invert else scaled

def main():
    with open("data/capitals.csv") as f:
        rows = list(csv.DictReader(f))

    dest_rows = [row for row in rows if row["capital"] != "Paris"]
    lats = [float(row["lat"]) for row in dest_rows]
    lons = [float(row["lon"]) for row in dest_rows]

    print(f"Fetching weather for {len(dest_rows)} capitals in a single batched request...")
    weather = fetch_weather_batch(lats, lons)
    if len(weather) != len(dest_rows):
        raise RuntimeError(
            f"Open-Meteo returned {len(weather)} results for {len(dest_rows)} "
            "requested locations -- refusing to guess the pairing."
        )
    print(f"Received {len(weather)} results")

    destinations = []
    for row, lat, lon, wx in zip(dest_rows, lats, lons, weather):
        distance_km = haversine_km(PARIS_LAT, PARIS_LON, lat, lon)
        daily = wx["daily"]
        avg_temp = sum(daily["temperature_2m_max"]) / len(daily["temperature_2m_max"])
        total_precip = sum(daily["precipitation_sum"])
        avg_sunshine = sum(daily["sunshine_duration"]) / len(daily["sunshine_duration"])
        destinations.append({
            "country": row["country"], "capital": row["capital"],
            "lat": lat, "lon": lon,
            "distance_km": round(distance_km, 1),
            "co2_kg_roundtrip": round(2 * distance_km * EMISSIONS_FACTOR_KG_PER_KM, 1),
            "avg_max_temp": round(avg_temp, 1),
            "total_precip_mm": round(total_precip, 1),
            "avg_sunshine_s": round(avg_sunshine, 0),
        })

    temp_scores = normalize([d["avg_max_temp"] for d in destinations])
    sun_scores = normalize([d["avg_sunshine_s"] for d in destinations])
    rain_scores = normalize([d["total_precip_mm"] for d in destinations], invert=True)
    co2_scores = normalize([d["co2_kg_roundtrip"] for d in destinations], invert=True)

    for d, t, s, rn, c in zip(destinations, temp_scores, sun_scores, rain_scores, co2_scores):
        d["weather_score"] = round((t + s + rn) / 3, 3)
        d["co2_score"] = round(c, 3)
        d["final_score"] = round((d["weather_score"] + d["co2_score"]) / 2, 3)

    destinations.sort(key=lambda d: d["final_score"], reverse=True)

    output = {
        "generated_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "emissions_factor_kg_per_km": EMISSIONS_FACTOR_KG_PER_KM,
        "destinations": destinations,
    }
    with open("docs/data/rankings.json", "w") as f:
        json.dump(output, f, indent=2)
    print(f"Wrote {len(destinations)} destinations to docs/data/rankings.json")

if __name__ == "__main__":
    main()
```

Commit both files (`data/capitals.csv`, then `scripts/fetch_and_score.py`).

> **Reflection — acquisition reliability:** before reading the docstring above a
> second time: why would making 26 separate API connections from the same script,
> in a loop, be *more* likely to fail than making 1 connection for all 26 capitals
> at once — not just slower, but more likely to actually fail? Write your guess,
> then check it against the comment in `fetch_weather_batch`.

> **Reflection — provenance (CM3):** the CO₂ figure (0.15 kg CO₂e/km) is a rounded
> value from a real, cited range (0.12-0.15, DEFRA-style short-haul factors) that
> the script's own comment says to "verify/update." If this dashboard were going to
> be used by real travelers next year instead of graded once, what would you
> actually do about this number before publishing it?

**Checkpoint:** both files exist in your repo's file list.

---

## Stage 2 — Cleaning & transformation: turning raw numbers into comparable scores

*Light transformation, setting up Stage 3's KPI design.*

Two things happen to the raw fetched numbers before they're usable: `haversine_km`
turns two pairs of coordinates into a real distance, and `normalize()` rescales
every raw metric (temperature, rain, sunshine, CO₂) onto the same 0-1 range so they
can be combined — a millimeter of rain and a kilogram of CO₂ have no shared unit
otherwise.

> **Reflection — relative vs. absolute:** `normalize()` finds the best and worst
> value *within this list of 26 capitals* and stretches everything between them to
> 0-1. If a 27th, unusually cold and rainy capital were added to the list, would a
> mediocre capital's real temperature change? Would its *score* change? What does
> that tell you about the difference between a measurement and a KPI built on top of
> it (CM2)?

---

## Stage 3 — KPI definition: combining weather and CO₂ into one number

*Connects to CM2 — a KPI is a construct you choose, not a neutral fact.*

Look again at the last few lines of `main()`:

```python
d["weather_score"] = round((t + s + rn) / 3, 3)
d["co2_score"] = round(c, 3)
d["final_score"] = round((d["weather_score"] + d["co2_score"]) / 2, 3)
```

`final_score` is a straight 50/50 average of weather and CO₂. That's not a law of
nature — it's one specific, editable choice, exactly the kind of choice CM2 argues
every KPI hides unless someone states it out loud.

> **Reflection — predict, then check:** based on the personal weighting you wrote
> down at the very start of this TD, which capital do you *predict* will come out
> on top once you run this script for real? Write the city name and one sentence of
> reasoning now, before you run anything.

You'll compare this prediction against the real result once the pipeline runs
(Stage 6).

---

## Stage 4 — Storage & versioning: committing results as data, not just displaying them

*Connects to CM3 — Git for data, reproducibility.*

Create a placeholder so the dashboard has something valid to read before the Action
has run for the first time: `docs/data/rankings.json`

```json
{
  "generated_at": "not yet generated",
  "emissions_factor_kg_per_km": 0.15,
  "destinations": []
}
```

Every real run overwrites this file and **commits it** — the result isn't just
displayed and discarded, it's versioned. Every past ranking is recoverable from Git
history, and `generated_at` is written into the file itself every time, which turns
out to matter a lot in Stage 6.

**Checkpoint:** the placeholder file exists, exactly as above.

---

## Stage 5 — Visualization for decision support

*Connects to CM1 — what decision does this support, for whom?*

Create `docs/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>EU capitals — sustainable vacation ranking</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
<style>
  body { font-family: system-ui, sans-serif; max-width: 900px; margin: 2rem auto; padding: 0 1rem; }
  #recommendation { padding: 1rem; border: 2px solid #0d7c9e; border-radius: 8px; }
  canvas { margin-top: 2rem; }
  #updated { color: #666; font-size: 0.85rem; }
</style>
</head>
<body>
  <h1>Best EU capital to visit next week, from Paris</h1>
  <p>Ranked by weather quality and CO&#8322; emissions of getting there — flight price is
     deliberately not a factor.</p>
  <div id="recommendation">Loading...</div>
  <canvas id="rankChart" height="320"></canvas>
  <canvas id="scatterChart" height="320"></canvas>
  <p id="updated"></p>

<script>
fetch('data/rankings.json')
  .then(r => r.json())
  .then(data => {
    const dest = data.destinations;

    if (!dest || dest.length === 0) {
      document.getElementById('recommendation').innerHTML =
        '<p>No ranking data yet. The automated workflow hasn\'t completed a run ' +
        'yet -- trigger it once from the repo\'s Actions tab ("Run workflow"), ' +
        'then reload this page.</p>';
      document.getElementById('updated').textContent = 'Status: ' + data.generated_at;
      return;
    }

    const top = dest[0];

    document.getElementById('recommendation').innerHTML =
      `<h2>Recommended: ${top.capital}, ${top.country}</h2>` +
      `<p>Weather score ${top.weather_score.toFixed(2)}, CO&#8322; score ` +
      `${top.co2_score.toFixed(2)} (${top.co2_kg_roundtrip} kg CO&#8322;e round trip, ` +
      `${top.distance_km} km each way).</p>`;

    document.getElementById('updated').textContent =
      `Data last updated ${data.generated_at} -- emissions factor ` +
      `${data.emissions_factor_kg_per_km} kg CO2e/passenger-km.`;

    // Green (best) to red (worst) colour spectrum, shared by both charts, based
    // on each destination's final_score relative to the others actually present.
    function scoreToColor(t) {
      const clamped = Math.max(0, Math.min(1, t));
      const hue = 120 * clamped; // 0 = red, 120 = green
      return `hsl(${hue.toFixed(0)}, 70%, 45%)`;
    }
    function normalizeForColor(values) {
      const lo = Math.min(...values);
      const hi = Math.max(...values);
      if (hi === lo) return values.map(() => 0.5);
      return values.map(v => (v - lo) / (hi - lo));
    }
    const scoreNorm = normalizeForColor(dest.map(d => d.final_score));
    const spectrumColors = scoreNorm.map(t => scoreToColor(t));

    new Chart(document.getElementById('rankChart'), {
      type: 'bar',
      data: {
        labels: dest.map(d => d.capital),
        datasets: [{
          label: 'Final score',
          data: dest.map(d => d.final_score),
          backgroundColor: spectrumColors
        }]
      },
      options: {
        plugins: {
          legend: { display: false },
          title: { display: true, text: 'Ranking, all 26 destinations (green = best, red = worst)' },
          tooltip: {
            callbacks: {
              title: (items) => {
                const d = dest[items[0].dataIndex];
                return `${d.capital}, ${d.country}`;
              },
              label: (item) => `Final score: ${item.parsed.y.toFixed(3)}`
            }
          }
        }
      }
    });

    new Chart(document.getElementById('scatterChart'), {
      type: 'scatter',
      data: {
        datasets: [{
          label: 'Weather vs CO2 (top-left is best)',
          data: dest.map(d => ({ x: 1 - d.co2_score, y: d.weather_score })),
          backgroundColor: spectrumColors,
          pointRadius: 9,
          pointHoverRadius: 13
        }]
      },
      options: {
        plugins: {
          legend: { display: false },
          title: { display: true, text: 'Trade-off: CO2 emissions vs weather quality (green = best overall, red = worst)' },
          tooltip: {
            callbacks: {
              label: (item) => {
                const d = dest[item.dataIndex];
                return `${d.capital}, ${d.country}`;
              }
            }
          }
        },
        scales: {
          x: { title: { display: true, text: 'CO2 emissions (normalized, lower = better)' } },
          y: { title: { display: true, text: 'Weather score (higher = better)' } }
        }
      }
    });
  })
  .catch(err => {
    document.getElementById('recommendation').innerHTML =
      '<p>Could not load ranking data (' + err + '). Check that ' +
      '<code>docs/data/rankings.json</code> exists in the repo and that GitHub ' +
      'Pages is serving from the <code>/docs</code> folder.</p>';
  });
</script>
</body>
</html>
```

Notice this page shows **two different charts of the same data**, not one: a ranked
bar chart, and a scatter plot with weather on one axis and CO₂ on the other. They
support two different questions — "just tell me the answer" versus "show me the
actual trade-off so I can judge it myself." The top-left corner of the scatter is
where both objectives agree; it is very often *not* the same point as the highest
bar in the ranking, once weights are anything other than 50/50.

**Checkpoint:** the file exists in your repo.

---

## Stage 6 — Automation & monitoring

*Connects to CM4 — orchestration, scheduling, and monitoring as a first-class stage.*

Create `.github/workflows/update-data.yml` (the path must be exact — the `.github`
folder with the leading dot, containing a `workflows` folder):

```yaml
name: Update capitals ranking

"on":
  schedule:
    - cron: "0 6 * * 1"   # every Monday 06:00 UTC
  workflow_dispatch: {}    # lets you trigger it manually from the Actions tab

permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - run: pip install requests
      - run: python scripts/fetch_and_score.py
      - name: Commit updated data
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/data/rankings.json
          git commit -m "Update rankings [automated]" || echo "Nothing to commit"
          git push
```

Now wire everything up and run it for real:

1. **Settings → Pages → Source: "Deploy from a branch"** (not "GitHub Actions" —
   this project has no build step, `docs/` is already the finished site) **→ Branch
   `main`, folder `/docs` → Save.** Note your Pages URL.
2. **Actions tab → "Update capitals ranking" → Run workflow → Run workflow.** Wait
   for the green checkmark (20-40 seconds).
3. Check `docs/data/rankings.json` in the **Code** tab — it should now hold 26 real
   destinations and a fresh `generated_at` timestamp, not the placeholder.
4. Visit your Pages URL. You should see a recommendation, both charts, and the
   "last updated" line.

> **Reflection — was your prediction right?** Compare the real top-ranked capital
> to what you predicted in Stage 3. If they match, what does that tell you about how
> well you understood the 50/50 formula? If they don't match, what surprised you —
> was it the weather data, the distance, or the formula itself?

> **Reflection — monitoring (CM4):** GitHub automatically disables a scheduled
> workflow after 60 days with no repository activity — a real, documented behavior,
> not a bug. If this dashboard's schedule silently stopped running (or started
> silently failing every week), what is the *one* thing on the live public page a
> visitor could look at to notice something was wrong, without checking the Actions
> tab? Is that signal good enough on its own, or would you add something?

**From this point on, the workflow re-runs automatically every Monday** with no
further action from you — that automatic re-run, not the first manual one, is the
actual point of the exercise.

---

## Stage 7 — On your own: change the flight origin

Every step so far assumed Paris as the departure city, because it's hardcoded into
`fetch_and_score.py`. Your task: make the whole pipeline work for a **different
origin city of your choice** — any real city with known coordinates, not necessarily
an EU capital. This is deliberately not walked through click-by-click — finding
exactly what needs to change, and confirming nothing broke, is the point. You have
everything you need from Stages 1-6.

**Hints, not answers** (find the exact lines yourself):

- One pair of constants near the top of `fetch_and_score.py` defines the origin's
  coordinates.
- The script excludes exactly one city, matched by name, from its own ranking
  (because you can't recommend someone fly to where they already are). If your
  origin is one of the 27 capitals, check this still works; if it isn't in the file
  at all, think about whether the exclusion is even still needed.
- `docs/index.html` has "Paris" written directly into the visible page text more
  than once. Decide how to handle this — edit the strings by hand, or have the
  script write the origin's name into `rankings.json` and read it from there.

**What "done" looks like:**

- [ ] The dashboard's recommendation, distances, and CO₂ figures visibly changed to
      reflect your new origin.
- [ ] No leftover "from Paris" text anywhere a viewer would see it.
- [ ] A fresh Action run completed successfully after your change.

> **Reflection — the unguided step:** which file(s) did you change, and why were
> those the right place(s) to change? And separately: is the "best" destination an
> absolute fact about these 26 capitals, or does it depend on where you start from?
> Compare your own top result to a classmate's (different origin city) if you can —
> did you get the same recommended capital?

---

## Reflection checklist (`REFLECTIONS.md` should answer all of these)

1. Before building anything: your personal weighting of weather vs. CO₂.
2. Why one batched API call is more reliable, not just faster, than 26 separate
   calls.
3. What you'd do about the CO₂ emissions-factor number before publishing this for
   real.
4. What normalization within a list of 26 means for a KPI vs. a raw measurement.
5. Your predicted winning capital, before running the script.
6. Whether your prediction matched the real result, and why/why not.
7. What signal on the live page would tell a visitor the pipeline had silently
   broken.
8. Which file(s) you changed for the origin-city extension, and whether the "best"
   destination is absolute or relative to your starting point.

## Final delivery checklist

- [ ] Repository public, workflow permissions read/write.
- [ ] All 5 pipeline files present and committed (see repo layout above).
- [ ] `REFLECTIONS.md` present with all 8 reflections answered.
- [ ] At least one green run in the Actions tab; `rankings.json` holds real data,
      not the placeholder.
- [ ] GitHub Pages live, dashboard shows real recommendations and charts.
- [ ] Stage 7 completed: dashboard now ranks from your own chosen origin city.

Submit: the repository URL, the live Pages URL, and `REFLECTIONS.md`.
