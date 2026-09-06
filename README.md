# 🎧 My Spotify Story Unwrapped

A personal, interactive **Spotify Wrapped** — rebuilt as a Tableau dashboard from raw Spotify Extended Streaming History data. Pick a year and every KPI, chart, and ranking on the page recalculates: total streams and hours listened vs. the year before, a dynamically ranked Top 5 Artists list, an hour-by-day listening heatmap, and breakdowns by platform, time of day, and shuffle vs. intentional listening.

![Tableau](https://img.shields.io/badge/Built%20with-Tableau-E97627?style=flat&logo=tableau&logoColor=white)
![Data](https://img.shields.io/badge/Data-Spotify%20Extended%20Streaming%20History-1DB954?style=flat&logo=spotify&logoColor=white)
![Status](https://img.shields.io/badge/Status-Complete-2E8B57?style=flat)


🔗 **Live dashboard:** https://public.tableau.com/views/spotifydashboard_17741827423100/lightmode?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link

## Preview

![Dashboard preview](assets/dashboard-preview.png)


## Table of Contents

- [Overview](#overview)
- [Dashboard Features](#dashboard-features)
- [Data Source](#data-source)
- [Data Preparation](#data-preparation)
- [How It Works](#how-it-works)
- [Parameters and Dashboard Actions](#parameters-and-dashboard-actions)
- [Calculated Fields Reference](#calculated-fields-reference)
- [Tech Stack](#tech-stack)
- [Data Privacy Note](#data-privacy-note)

## Overview

Spotify's own "Wrapped" only looks back once a year, at whatever slice of data Spotify decides to show. This project uses my full **Extended Streaming History** export (every play event, every year I've had an account) to build a version I control — one where I can flip between any year from 2018–2026, see it compared to the year before, and drill into *when*, *how*, and *on what* I actually listen.

The whole dashboard is driven by a single **Select Year** parameter. Nothing is hardcoded — change the parameter and every number, chart, and ranking on the page updates.

## Dashboard Features
 
| Tile | What it shows |
|---|---|
| **Total Streams** | Current-year stream count, % change vs. the previous year, and a mini trend sparkline |
| **Hours Listened** | Current-year total hours, % change vs. the previous year, and a mini trend sparkline |
| **Top 5 Artists** | The 5 most-played artists for the selected year, ranked by minutes played |
| **Listening Trend** | Month-by-month minutes listened across the selected year |
| **Listening Activity by Hour & Day** | Minutes listened, mapped as a Sun–Sat × 0–23h heatmap of when I listen most |
| **Year on Year** | Total minutes listened for every year in the data (2018–2026) |
| **Listening by Time of Day** | Minutes listened, split by Morning / Afternoon / Evening / Night |
| **Shuffle vs. Intentional Listening** | Minutes listened, shuffled vs. deliberately chosen, against an average reference line |
| **Listening by Platform** | Minutes listened, split by iPhone / Android / Windows / Other |

## Data Source

The dashboard runs on Spotify's **Extended Streaming History** export, which you can request yourself from *Spotify → Account → Account Privacy → Request data*. It arrives by email as a set of JSON files, one row per playback event — Spotify splits a long history into sequentially numbered chunks. In this project, that's two files:
 
| File | Coverage |
|---|---|
| `Streaming_History_Audio_2018-2022_0.json` | Dec 7, 2018 – Jan 15, 2022 |
| `Streaming_History_Audio_2022-2026_1.json` | Jan 15, 2022 – Mar 18, 2026 |

The fields the dashboard actually uses:

| Field | Description |
|---|---|
| `ts` | UTC timestamp for when the track stopped playing, ordered year-month-day followed by a 24-hour time |
| `platform` | Device/OS the track was streamed on |
| `ms_played` | Length of the stream, in milliseconds |
| `master_metadata_track_name` | Track name |
| `master_metadata_album_artist_name` | Artist name |
| `shuffle` | `True` or `False`, depending on whether shuffle mode was on during playback |

## Data Preparation

Every date/time field in the workbook is built on top of one base calculation:

```
Timestamp IST
DATEADD('minute', 330, [Timestamp])
```

Spotify logs `ts` in UTC. Since I'm in India, I shift every timestamp forward by 5 hours 30 minutes so that "Year," "Hour of Day," "Day of Week," and everything else downstream reflects when I actually pressed play — not the UTC log time.

## How It Works
 
This section walks through the techniques behind the trickier visuals. One term comes up right away: **`Select Year`** is the integer parameter behind the Year dropdown at the top of the dashboard (allowable values: a list, 2018–2026). Every formula below that references `[Select Year]` is reading whatever year is currently selected there — full details on how it's wired into filters and dashboard actions are in [Parameters and Dashboard Actions](#parameters-and-dashboard-actions).
 
### The year-over-year KPI cards (dual-axis line charts)
 
The **Total Streams** and **Hours Listened** tiles each pack three things into a single worksheet: a big current-year number, a ▲/▼ percentage vs. last year, and a 12-month trend sparkline — all in one sheet, no separate "big number" object.
 
**1. Split the measure by year, not by filtering rows.**
Instead of filtering the data to one year (which would make year-over-year comparison impossible), two calculated fields flag every row with its value *only if* it belongs to the relevant year, and 0 otherwise:
 
```
Current Year Hours
IF YEAR([Timestamp IST]) = [Select Year] THEN [Hours Played] ELSE 0 END
 
Previous Year Hours
IF YEAR([Timestamp IST]) = [Select Year] - 1 THEN [Hours Played] ELSE 0 END
```
 
The same pattern is used for stream counts, just flagging `1` instead of `[Hours Played]` — so a plain `SUM()` doubles as a `COUNT()`:
 
```
Current Year Streams
IF YEAR([Timestamp IST]) = [Select Year] THEN 1 ELSE 0 END
 
Previous Year Streams
IF YEAR([Timestamp IST]) = [Select Year] - 1 THEN 1 ELSE 0 END
```
 
Because both fields exist for every row regardless of year, both can sit on the same **Month Number** axis and be summed independently — which is what makes the side-by-side comparison possible in the first place.
 
**2. Overlay both years as a dual-axis line chart.**
`SUM(Current Year Hours)` and `SUM(Previous Year Hours)` both go on **Rows**, combined into a **dual axis** and synchronized so they share one scale. Each line is formatted separately — both bold, current year in black and previous year in gray — which is what produces the sparkline under each KPI number: two bold lines tracing the same 12 months side by side, this year against last.

<p align="center">
 <img src="assets/hours-listened-1.png" width="500" alt="Hours Listened">
</p>
 
**3. Turn the same sheet's title into the "big number."**
Rather than building a separate text object, the title text itself contains inserted calculated fields:
 
```
Hours Listened
SUM({SUM([Current Year Hours])})
{[Arrow - Hours]}  SUM({[% Change - Hours]}) vs. PY
```
 
The curly-brace aggregation totals Current Year Hours across the entire view (all 12 months) rather than letting it compute separately for each month on the line. A title needs one static value, and without the braces there isn't one to give — the calculation produces 12 different monthly numbers instead, so Tableau falls back to showing the range across all of them, rather than the intended full-year figure. `Arrow - Hours` and `% Change - Hours` are two more calculated fields feeding the same title:
 
```
% Change - Hours
(SUM([Current Year Hours]) - SUM([Previous Year Hours])) / SUM([Previous Year Hours])
 
Arrow - Hours
IF SUM([Current Year Hours]) > SUM([Previous Year Hours]) THEN "▲" ELSE "▼" END
```
 
The same fields power the tooltip too, just at monthly grain instead of full-year. Hovering any point on the sparkline breaks the total back down to that specific month: hours that month this year, hours that same month last year, and the % change between them, all pulled from the identical `Current Year Hours` / `Previous Year Hours` / `% Change - Hours` fields, just evaluated per-month rather than totaled across the whole view.
 
A small companion sheet, **KPI Legend**, renders the "2025 vs 2024" caption above the cards — built from two more passthrough fields, `Current Year` (`[Select Year]`) and `Previous Year` (`[Select Year]-1`), so it always names the correct pair of years.
 
The **Total Streams** card follows the exact same pattern, swapping Hours for Streams throughout.

<p align="center">
 <img src="assets/total-streams-1.png" width="500" alt="Total Streams">
</p>

 
### Cleaning up platform data
 
Spotify logs the `platform` field inconsistently — in this export, the same iPhone shows up as `ios` in some rows and as version-specific strings like `iOS 15.5 (iPhone12,8)` in others, while Android and Windows devices log full build strings such as `Android OS 7.1.2 API 25 (Xiaomi, Redmi 4)` or `Windows 10 (10.0.19042; x64; AppX)`. `Platform Clean` normalizes all of it with nested string matching:
 
```
Platform Clean
IF CONTAINS(LOWER([Platform]), "ios") OR CONTAINS(LOWER([Platform]), "iphone") THEN "iPhone"
ELSEIF CONTAINS(LOWER([Platform]), "android") THEN "Android"
ELSEIF CONTAINS(LOWER([Platform]), "windows") THEN "Windows"
ELSEIF CONTAINS(LOWER([Platform]), "mac") THEN "Mac"
ELSEIF CONTAINS(LOWER([Platform]), "web") THEN "Web"
ELSE "Other"
END
```
`LOWER()` makes the match case-insensitive; `CONTAINS()` catches any variant that mentions the platform name anywhere in the raw string, rather than needing an exact match.
 
### Bucketing time of day
 
`Time of Day` turns the raw hour into four labeled, human-readable buckets used by the donut chart and available as a cross-filter dimension elsewhere:
 
```
Time of Day
IF DATEPART('hour', [Timestamp IST]) >= 5 AND DATEPART('hour', [Timestamp IST]) < 12 THEN "Morning"
ELSEIF DATEPART('hour', [Timestamp IST]) >= 12 AND DATEPART('hour', [Timestamp IST]) < 17 THEN "Afternoon"
ELSEIF DATEPART('hour', [Timestamp IST]) >= 17 AND DATEPART('hour', [Timestamp IST]) < 21 THEN "Evening"
ELSE "Night"
END
```

### Ranking the Top 5 Artists with INDEX()
 
The Top 5 Artists bar chart needs to show *only* the top 5 — but "top 5" depends on which year is selected, so it can't be a static filter on artist name.
 
The fix is Tableau's `INDEX()` table calculation:
 
```
Top 5 artist Filter
INDEX()
```
 
With Artist sorted descending by `SUM(Minutes Played)` and the calculation set to compute **along Table (across)**, `INDEX()` returns each artist's rank — 1 for the most-played, 2 for the next, and so on. Dropping this field onto the Filters shelf and restricting it to a **range of 1 to 5** keeps only the top 5 ranked bars, out of however many distinct artists exist for that year. Because the ranking is a table calculation rather than a hardcoded value, it recalculates automatically every time the year parameter changes.
 
### Labeling Shuffle vs. Intentional
 
The raw `shuffle` field is just a boolean — `True` or `False`, nothing more descriptive than that. Rather than a calculated field, the **Shuffle vs. Intentional Listening** chart relabels those two values directly via Tableau's field aliases (`False` → "Intentional," `True` → "Shuffle") so the bar chart reads naturally without needing a lookup formula.

## Parameters and Dashboard Actions

**`Select Year`** is an integer parameter (allowable values: a fixed list, 2018–2026) that drives the whole dashboard. It's separate from — and works alongside — a **`Year`** filter:

```
Year filter (condition by formula)
[Year] = [Select Year]
```

The two aren't redundant. Sheets that only need to show *one* year's data (Top 5 Artists, the Time of Day donut, Platform breakdown, the Hour & Day heatmap, Shuffle vs. Intentional) use the `Year` filter to restrict rows outright. The two KPI cards deliberately **don't** use that filter — they need both years' rows present at once — and instead rely on the `Current Year …` / `Previous Year …` calculated fields to split the data without removing any of it.

The **Top 5 Artists** sheet also carries the `Top Filter`, restricting the `INDEX()`-based ranking field to 1–5, applied on top of the `Year` filter.

**Cross-filtering:** clicking a mark in one chart (an artist, a platform, an hour/day cell, a time-of-day slice, a shuffle bar) filters the rest of the dashboard through a set of dashboard filter actions. Each action is deliberately **excluded from its own source sheet** — so clicking an artist in the Top 5 Artists chart filters everything *else*, without that chart also filtering (and hiding) its own bars.

## Calculated Fields Reference
 
Full list of every calculated field in the workbook, grouped by purpose.
 
### Date & time
 
**Timestamp IST**
```
DATEADD('minute', 330, [Timestamp])
```
 
**Year**
```
YEAR([Timestamp IST])
```
 
**Year-month (YYYY-MM)**
```
STR(YEAR([Timestamp IST])) + "-" + RIGHT("0" + STR(MONTH([Timestamp IST])), 2)
```
 
**Month Name**
```
DATENAME('month', [Timestamp IST])
```
 
**Month Number**
```
MONTH([Timestamp IST])
```
 
**Day of Week**
```
DATENAME('weekday', [Timestamp IST])
```
 
**Hour of Day**
```
DATEPART('hour', [Timestamp IST])
```
 
**Time of Day**
```
IF DATEPART('hour', [Timestamp IST]) >= 5 AND DATEPART('hour', [Timestamp IST]) < 12 THEN "Morning"
ELSEIF DATEPART('hour', [Timestamp IST]) >= 12 AND DATEPART('hour', [Timestamp IST]) < 17 THEN "Afternoon"
ELSEIF DATEPART('hour', [Timestamp IST]) >= 17 AND DATEPART('hour', [Timestamp IST]) < 21 THEN "Evening"
ELSE "Night"
END
```
 
### Core metrics
 
**Hours Played**
```
[Ms Played] / 3600000
```
 
**Minutes Played**
```
[Ms Played] / 60000
```
 
### Platform cleaning
 
**Platform Clean**
```
IF CONTAINS(LOWER([Platform]), "ios") OR CONTAINS(LOWER([Platform]), "iphone") THEN "iPhone"
ELSEIF CONTAINS(LOWER([Platform]), "android") THEN "Android"
ELSEIF CONTAINS(LOWER([Platform]), "windows") THEN "Windows"
ELSEIF CONTAINS(LOWER([Platform]), "mac") THEN "Mac"
ELSEIF CONTAINS(LOWER([Platform]), "web") THEN "Web"
ELSE "Other"
END
```
 
### Year-over-year comparison engine
 
**Current Year**
```
[Select Year]
```
 
**Previous Year**
```
[Select Year]-1
```
 
**Current Year Hours**
```
IF YEAR([Timestamp IST]) = [Select Year] THEN [Hours Played] ELSE 0 END
```
 
**Previous Year Hours**
```
IF YEAR([Timestamp IST]) = [Select Year] - 1 THEN [Hours Played] ELSE 0 END
```
 
**Current Year Streams**
```
IF YEAR([Timestamp IST]) = [Select Year] THEN 1 ELSE 0 END
```
 
**Previous Year Streams**
```
IF YEAR([Timestamp IST]) = [Select Year] - 1 THEN 1 ELSE 0 END
```
 
**% Change - Hours**
```
(SUM([Current Year Hours]) - SUM([Previous Year Hours])) / SUM([Previous Year Hours])
```
 
**% Change — Streams**
```
(SUM([Current Year Streams]) - SUM([Previous Year Streams])) / SUM([Previous Year Streams])
```
 
**Arrow - Hours**
```
IF SUM([Current Year Hours]) > SUM([Previous Year Hours]) THEN "▲" ELSE "▼" END
```
 
**Arrow - Streams**
```
IF SUM([Current Year Streams]) > SUM([Previous Year Streams]) THEN "▲" ELSE "▼" END
```
 
### Ranking
 
**Top 5 artist Filter**
```
INDEX()
```
*Computed along Table (across)*

## Tech Stack
 
- **Tableau Desktop** (Tableau Public–compatible) for all modeling, calculations, and dashboard design
- **Spotify Extended Streaming History** (JSON) as the sole data source
- Tableau features used throughout: calculated fields, table calculations (`INDEX()`, table-scoped `SUM({...})` aggregation), an integer **parameter**, **dual-axis** charts, **dashboard filter actions**, field aliases, and formula-driven titles/tooltips

## Data Privacy Note
 
Spotify's extended streaming history contains personal data, including IP addresses. Neither the raw export nor the packaged Tableau workbook is included in this repository, since a `.twbx` export would embed that same underlying data. Only the dashboard's logic — calculated fields, parameters, structure — is documented here.
