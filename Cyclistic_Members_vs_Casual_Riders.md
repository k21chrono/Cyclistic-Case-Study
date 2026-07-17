# Cyclistic: Members vs. Casual Riders

This report examines how annual members and casual riders differ in their use of Cyclistic's bike-share service, to inform a marketing campaign aimed at converting casual riders into members.

---

## Data Summary

| | |
|---|---|
| **Period** | May 2026 |
| **Source** | [divvy-tripdata.s3.amazonaws.com/202605-divvy-tripdata.zip](https://divvy-tripdata.s3.amazonaws.com/202605-divvy-tripdata.zip) |
| **Rows** | 653,704 |
| **Columns** | 13 |
| **Tooling** | DuckDB (SQL exploration) → Metabase (visualization) |

**Schema:**

| Column | Type / Values |
|---|---|
| `ride_id` | string, hash — primary key |
| `rideable_type` | `electric_bike` / `classic_bike` |
| `started_at`, `ended_at` | datetime |
| `start_station_name`, `start_station_id` | string (title case / 3 letters + 5 numbers) |
| `end_station_name`, `end_station_id` | string (title case / 3 letters + 5 numbers) |
| `start_lat`, `start_lng`, `end_lat`, `end_lng` | float |
| `member_casual` | `member` / `casual` |

### User Base Overview

```sql
SELECT
  COUNT(member_casual) AS total_trips,
  SUM(CASE WHEN member_casual = 'member' THEN 1 ELSE 0 END) AS member_trips,
  SUM(CASE WHEN member_casual = 'casual' THEN 1 ELSE 0 END) AS casual_trips,
  ROUND(100.0 * COUNT(*) FILTER (WHERE member_casual = 'casual') / COUNT(*), 1) AS casual_pct
FROM trips
```

| Total trips | Member trips | Casual trips | Casual % |
|---|---|---|---|
| 653,704 | 408,599 | 245,105 | 37.5% |

Roughly **3 in 8 trips** are taken by casual riders — worth keeping as a baseline rate, since several findings below compare a sub-group's casual share against this 37.5% average.

---

## ⚠️ Data Constraints & Limitations

Read this before the findings — it affects how much weight each one should carry.

| Constraint                        | Detail                                                                                                             | Status                                                                                                                                |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Single month**                  | Only May 2026 is covered.                                                                                          | Open — no seasonal comparison. May's weather may skew leisure (casual) ridership higher than a winter month would.                    |
| **Duration hard cap at 1500 min** | Ride duration tops out at exactly 1500 minutes for both user types.                                                | Confirmed as a data-capture ceiling, not a natural distribution tail.                                                                 |
| **Small long-duration segments**  | Segment sizes shrink sharply as trip duration increases (table below).                                             | Open — treat Day Tripper and Extreme Tripper findings as directional, not conclusive.                                                 |
| **No significance testing**       | All comparisons are descriptive (means, percentages, counts); no confidence intervals or hypothesis tests applied. | Open — findings on the full 653k-row dataset are robust regardless; findings on segments under ~1,000 rows should be read cautiously. |

**Segment sizes at a glance** — the key caveat for the duration-bin analysis further down:

| Segment | Duration | n (approx.) | Share of dataset |
|---|---|---|---|
| Quick Trippers | 0–120 min | ~647,000 | >99% |
| Part Time Trippers | 121–480 min | 2,602 | 0.40% |
| Day Trippers | 481–720 min | 837 | 0.13% |
| Extreme Trippers | 721–1500 min | 104 | 0.016% |

The three long-duration segments combined make up **under 0.6% of all rides**. Their charts are useful for generating hypotheses, not for making firm claims — a single unusual rider can visibly move a pie chart at n = 104.

---

## Ride Duration

### Mean Duration

```sql
SELECT member_casual, TRUNC(MEAN(DATEDIFF('minute', started_at, ended_at)), 2) AS duration
FROM trips
GROUP BY member_casual
```

| User type | Mean duration (min) |
|---|---|
| Casual | 21.88 |
| Member | 12.51 |

**Finding:** Casual riders' average trip is **~9 minutes (75%) longer** than members'. Given the dataset size (600k+ rides), this difference is robust.

### Duration Cap — Resolved

The initial max-duration query returned exactly **1500 minutes** for both user types — a suspiciously round number suggesting a hard cap.

```sql
SELECT
    COUNT(*) FILTER (WHERE DATEDIFF('minute', started_at, ended_at) = 1499) AS trips_1499,
    COUNT(*) FILTER (WHERE DATEDIFF('minute', started_at, ended_at) = 1500) AS trips_1500,
    COUNT(*) FILTER (WHERE DATEDIFF('minute', started_at, ended_at) = 1501) AS trips_1501
FROM trips
```

| trips at 1499 min | trips at 1500 min | trips at 1501 min |
|---|---|---|
| 113 | **444** | 0 |

**Finding — confirmed:** A roughly 4x spike exactly at 1500 minutes, followed by an immediate hard drop to zero at 1501, is the signature of a recording ceiling, not a natural distribution tail. **Any ride reported at 1500 minutes should be treated as "≥1500 minutes, exact duration unknown," not as a real 1500-minute ride.** This affects a small number of rows (444 of 653,704, or 0.07%) but is worth flagging in any duration-based modeling.

```sql
SELECT member_casual, COUNT(ride_id)
FROM trips
WHERE DATEDIFF('minute', started_at, ended_at) BETWEEN 1401 AND 1499
GROUP BY member_casual
```

| User type | Count (1401–1499 min) |
|---|---|
| Casual | 111 |
| Member | 25 |

**Finding:** Consistent with the broader duration story — casual riders account for **~4x more rides** near the upper duration limit than members.

---

## Day-of-Week Patterns

```sql
PIVOT trips
ON STRFTIME(started_at, '%a') IN ('Sun','Mon','Tue','Wed','Thu','Fri','Sat')
USING COUNT(*)
GROUP BY member_casual;
```
![[count by dow and user type.png]]

| User type | Sun | Mon | Tue | Wed | Thu | Fri | Sat |
|---|---|---|---|---|---|---|---|
| Casual | 45,570 | 32,443 | 23,939 | 24,360 | 23,941 | 36,663 | 58,189 |
| Member | 52,004 | 51,959 | 57,036 | 60,884 | 58,494 | 66,088 | 62,134 |

**Finding:** Casual riders peak sharply on Saturday and bottom out midweek (Tuesday). Members are comparatively flat across the week, with a mild lift toward Friday/Saturday.

### Weekend vs. Weekday Share

```sql
SELECT member_casual,
  ROUND(100.0 * COUNT(CASE WHEN STRFTIME(started_at, '%w') IN ('0','6') THEN 1 END) / COUNT(*), 2) AS weekend_pct,
  ROUND(100.0 * COUNT(CASE WHEN STRFTIME(started_at, '%w') NOT IN ('0','6') THEN 1 END) / COUNT(*), 2) AS weekday_pct
FROM trips
GROUP BY member_casual;
```

| User type | Weekend % | Weekday % |
|---|---|---|
| Casual | 42.33 | 57.67 |
| Member | 27.93 | 72.07 |

**Finding:** Casual riders are proportionally **~1.5x more likely** to ride on a weekend than members (42% vs. 28% of their respective trip totals).

---

## Bicycle Type Preference

```sql
WITH bike_percentages AS (
  SELECT member_casual, rideable_type,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY member_casual), 2) AS pct
  FROM trips
  GROUP BY member_casual, rideable_type
)
PIVOT bike_percentages ON rideable_type USING FIRST(pct)
GROUP BY member_casual;
```

| User type | Classic bike | Electric bike |
|---|---|---|
| Casual | 27.09% | 72.91% |
| Member | 31.71% | 68.29% |

**Finding:** No meaningful difference — all values fall within ~5 points of each other. **No further analysis needed here**; bike type is not a useful targeting lever.

---

## Geographic Analysis

Earlier drafts of this report binned location at 0.1° (~11 km), which was too coarse to detect real differences and left this question flagged as inconclusive. This version replaces that with a proper station-level rollup.

```sql
WITH station_coords AS (
    SELECT
        start_station_id,
        start_station_name,
        TRUNC(MAX(start_lat), 2) AS start_lat,
        TRUNC(MAX(start_lng), 2) AS start_lng
    FROM trips
    WHERE start_station_id IS NOT NULL
    GROUP BY 1, 2
)
SELECT
    station_coords.start_station_name,
    station_coords.start_lat,
    station_coords.start_lng,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE trips.member_casual = 'casual') AS casual_count,
    COUNT(*) FILTER (WHERE trips.member_casual = 'member') AS member_count,
    ROUND(100.0 * COUNT(*) FILTER (WHERE trips.member_casual = 'casual') / COUNT(*), 1) AS casual_pct
FROM trips
JOIN station_coords ON trips.start_station_id = station_coords.start_station_id
GROUP BY 1, 2, 3
HAVING total > 100 AND casual_pct > 50
ORDER BY total DESC
LIMIT 10
```

*(`HAVING total > 100` deliberately excludes low-traffic stations, so single-digit ride counts can't produce a misleading 100% casual reading — the same small-sample discipline applied to the duration segments above.)*

**Top 10 casual-majority stations, by volume:**

| Station | Lat | Lng | Total | Casual % |
|---|---|---|---|---|
| Navy Pier | 41.89 | -87.61 | 7,792 | 78.7% |
| DuSable Lake Shore Dr & Monroe St | 41.88 | -87.61 | 5,372 | 81.6% |
| DuSable Lake Shore Dr & North Blvd | 41.91 | -87.62 | 3,988 | 56.6% |
| Michigan Ave & Oak St | 41.90 | -87.62 | 3,900 | 62.8% |
| Theater on the Lake | 41.92 | -87.63 | 3,181 | 58.2% |
| Millennium Park | 41.88 | -87.62 | 2,870 | 70.4% |
| Dusable Harbor | 41.91 | -87.61 | 2,686 | 72.5% |
| Shedd Aquarium | 41.86 | -87.61 | 2,541 | 83.8% |
| Montrose Harbor | 41.96 | -87.63 | 2,244 | 57.7% |
| Adler Planetarium | 41.86 | -87.60 | 1,768 | 75.7% |
![[geographic casual over 50.png]]
**Finding :** every station on this list sits directly on Chicago's lakefront/museum-campus corridor. This isn't a broad "coastal vs. inland" gradient — it's a **specific, named cluster of tourist and recreational destinations** (Navy Pier, Shedd Aquarium, Field Museum's neighbor Adler Planetarium, Millennium Park, the lakefront harbors). Navy Pier alone accounts for 7,792 trips at 78.7% casual, nearly double the dataset-wide casual rate of 37.5%.

**Scale check:** of the 636 unique stations with more than 100 rides, only **65 (≈10%)** are casual-majority (>50% casual). Casual dominance is concentrated in a small, identifiable minority of high-traffic, lakefront/tourist stations — not spread evenly across the service area.

**Bottom 10 casual-majority stations:**

| Station | Lat | Lng | Total | Casual % |
|---|---|---|---|---|
| St Louis Ave & Foster Ave | 41.97 | -87.71 | 178 | 7.9% |
| St. Louis Ave & Balmoral Ave | 41.98 | -87.71 | 236 | 10.2% |
| Wood St & Taylor St (Temp) | 41.87 | -87.67 | 430 | 13.3% |
| Greenwood Ave & 47th St | 41.80 | -87.59 | 197 | 13.7% |
| Morgan St & Polk St | 41.87 | -87.65 | 1,177 | 13.8% |
| Ogden Ave & Roosevelt Rd | 41.86 | -87.68 | 179 | 14.0% |
| Wolcott Ave & Polk St | 41.87 | -87.67 | 773 | 14.4% |
| Loomis St & Lexington St | 41.87 | -87.66 | 1,400 | 16.0% |
![[geographic casual under 37.png]]
**Finding :** These sit in residential neighborhoods well inland from the lakefront, consistent with commuter/residential use rather than leisure. Including both ends of the spectrum makes the geographic story a contrast — lakefront/tourist vs. inland/residential — rather than a one-sided list, which is a stronger basis for an Out-of-Home targeting map.

As stated above, casual share reaches as high as 83.8% at individual stations, well above the 37.5% dataset-wide average — concentrated specifically along the lakefront corridor. Useful for Out-of-Home placement targeting.

---

## Duration Distribution

### Share of Trips by Duration (0–200 min)

![[times by user up to 200.png]]

**Finding:** Casual riders' share of trips climbs steadily with duration — by the 50-minute mark, casual riders make up the majority of trips still in progress.

### Quick Trippers (0–120 min · ~99% of dataset)

![[quick trippers.png]]

**Finding:** This bin holds the overwhelming majority of all rides and is dominated by members. Because this segment is so large, its breakdown is the most statistically reliable finding in the report.

#### Quick Tripper Breakdown: Under vs. Over 60 Minutes

![[quick trippers breakdown.png]]

**Finding:** Casual riders are proportionally far more likely to run past the 60-minute mark than members — the strongest quantified evidence in the report for a "casual riders take longer trips" story, and consistent with the mean-duration and duration-cap findings above.

### Part Time Trippers (121–480 min · n = 2,602 · 0.40% of dataset)

![[part time trippers.png]]

**Interpretation:** Likely represents workers parking at a job site for a shift, gig workers, or long service-area hauls.
⚠️ **Small sample (n = 2,602).** Directionally useful, not statistically definitive — treat the exact percentage split as approximate.

### Day Trippers (481–720 min · n = 837 · 0.13% of dataset)

![[day trippers.png]]

**Interpretation:** Likely full-time workers parking bikes for a full shift, full-time gig workers, or extended out-of-area hauls.
⚠️ **Small sample (n = 837).** Treat as directional only — at this size, a handful of unusual trips can shift the split by several points.

### Extreme Trippers (721–1500 min · n = 104 · 0.016% of dataset)

![[extreme trippers.png]]

⚠️ **Very small sample (n = 104), and now further complicated by the duration cap finding above** — trips reported at exactly 1500 minutes in this bin are capped values, not necessarily true 1500-minute rides, so both the segment's upper boundary and its user-type split should be treated as approximate. The apparent shift in member share versus the Day Tripper segment is well within what could occur by chance at this sample size — **not a reliable trend**, and not a basis for strategy without real market research into this segment specifically (see Further Analysis).

---

## Chronology

**Trips by day of week:** see Day-of-Week Patterns above — casual usage clearly favors weekends; member usage is comparatively steady across the week.

### AM/PM Split

![[count am_pm.png]]

| User type | AM | PM |
|---|---|---|
| Casual | 24.2% | 75.8% |
| Member | 30.9% | 69.1% |

**Finding:** Both groups skew PM, but casual riders skew slightly more so. The gap is real but modest — useful as a minor input to campaign timing, not a headline finding.

### Hour of Day

![[trips by hour of day and user type.png]]

**Finding:** The clearest behavioral signal in the dataset. Members show a **bimodal commuter pattern** — a trough through the 9-to-5 workday with peaks around typical commute hours. Casual riders show a **steady, monotonic rise** through the day into an afternoon/early-evening peak, with no midday dip. This strongly suggests member usage skews commute-driven while casual usage skews leisure-driven — and now pairs well with the geographic finding: leisure-pattern usage concentrates at lakefront/tourist stations.

---

## Key Findings

Ranked by strength of evidence (sample size, consistency across cuts, and resolution status):

1. **Casual riders take meaningfully longer trips.** Mean duration is 75% higher (21.9 vs. 12.5 min), corroborated by the duration-cap and quick-tripper breakdowns. *(Strong — full 653k-row dataset, multiple independent cuts agree.)*
2. **Casual usage doesn't follow a 9-to-5 commute pattern; member usage does.** Textbook commuter double-peak for members vs. a steady leisure-style climb for casual riders. *(Strong — full dataset.)*
3. **Casual usage concentrates geographically at a specific lakefront/tourist cluster**, not spread evenly across the service area — only ~10% of qualifying stations are casual-majority, but they include the highest-volume stations in the system.
4. **Casual riders skew toward weekends and, more mildly, toward PM.** *(Strong for weekend effect, moderate for AM/PM.)*
5. **The 1500-minute duration cap is a data artifact, not a real ride length.** Confirmed by the spike-then-cliff pattern at 1499/1500/1501. *(Resolved — small in volume (0.07% of rides) but should inform how duration is modeled elsewhere.)*
6. **Bike type shows no meaningful difference.** *(Strong null result — confidently ruled out as a targeting lever.)*
7. **Long-duration segment patterns (Part Time / Day / Extreme Trippers) are suggestive only.** Based on samples from 2,602 down to just 104 rides. *(Weak — hypothesis-generating, not confirmatory.)*

---

## My Suggestions

1. **Emphasize the benefits of membership for trips longer than 30–50 minutes**, where casual riders disproportionately concentrate and where per-ride pricing likely feels most painful.
2. **Target marketing spend toward weekends, with a mild lean toward afternoons/evenings**, matching the strongest and best-supported casual usage pattern.
3. **Deploy Out-of-Home marketing at the specific lakefront/tourist stations identified above** (Navy Pier, Shedd Aquarium, Millennium Park, DuSable Lake Shore Dr corridor, Adler Planetarium)

---

## Areas for Further Analysis

- **Add a multi-month or seasonal comparison** — May-only data may overstate casual/leisure ridership relative to colder months.
- **Treat Part Time / Day / Extreme Tripper findings as hypotheses to test**, not conclusions — ideally with a larger date range to build up sample size in these segments before acting on them, and with real qualitative/market research into who these riders actually are.
- **Expand data collection for the Quick Tripper segment** (the largest, most reliable slice of the dataset) to further tailor messaging.
- **Model the 1500-minute-cap rides as censored data** (≥1500 min, not =1500 min) in any future duration-based analysis, to avoid understating the true tail of casual ride lengths.

---

*Prepared by Kendrick H. — K21Confirm@gmail.com*
