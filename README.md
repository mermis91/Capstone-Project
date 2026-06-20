# SpaceX Data Explorer

A command-line Python script that fetches and displays live SpaceX data — launches, rockets, crew, programs, launchpads, and more — directly from a public REST API. No API key required. No external dependencies.

---

## Table of Contents

- [Background](#background)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
  - [Interactive Mode](#interactive-mode)
  - [Direct Mode](#direct-mode)
  - [Run All Sections](#run-all-sections)
- [Sections](#sections)
  - [1 — Company Profile](#1--company-profile)
  - [2 — Latest Launches](#2--latest-launches)
  - [3 — Upcoming Launches](#3--upcoming-launches)
  - [4 — Rockets & Launch Stats](#4--rockets--launch-stats)
  - [5 — Programs](#5--programs)
  - [6 — Crew & Astronauts](#6--crew--astronauts)
  - [7 — Launchpads](#7--launchpads)
- [API Reference](#api-reference)
  - [Base URL](#base-url)
  - [Endpoints Used](#endpoints-used)
  - [Rate Limits & Fair Use](#rate-limits--fair-use)
- [Code Structure](#code-structure)
- [Error Handling](#error-handling)
- [A Note on the Original SpaceX API](#a-note-on-the-original-spacex-api)
- [Sample Output](#sample-output)

---

## Background

This script was built to explore SpaceX mission data from the command line. It was originally intended to use the community-maintained SpaceX REST API (`api.spacexdata.com`), but that project was **archived and shut down on June 6, 2026**. The script was therefore rewritten to use **The Space Devs API** (`lldev.thespacedevs.com`), which provides the same data (and more) and remains actively maintained.

All data is fetched live on every run — there is no local cache, so results always reflect the current state of the database.

---

## Requirements

- **Python 3.10 or later** (uses the `X | Y` union type syntax for type hints)
- An internet connection
- No third-party packages — only Python standard library modules are used:
  - `json`
  - `sys`
  - `urllib.request`
  - `urllib.error`
  - `urllib.parse`
  - `re` (imported lazily inside the crew section)

To check your Python version:

```bash
python3 --version
```

---

## Installation

No installation or package manager is needed. Simply download the script:

```bash
# Clone the repository (or just copy spacex.py)
git clone <your-repo-url>
cd spacex

# Make the script executable (optional, macOS/Linux)
chmod +x spacex.py
```

---

## Usage

### Interactive Mode

Run the script with no arguments to launch the interactive menu:

```bash
python3 spacex.py
```

You will see:

```
  SpaceX Data Explorer  (via The Space Devs API)
  ─────────────────────────────────────────────
  [1] Company Profile
  [2] Latest Launches
  [3] Upcoming Launches
  [4] Rockets & Launch Stats
  [5] Programs
  [6] Crew / Astronauts
  [7] Launchpads
  [0] All sections

  Select a section (default 0 = all):
```

Type a number and press Enter. Pressing Enter without typing anything defaults to `0` (all sections).

### Direct Mode

Pass the section number as a command-line argument to skip the menu entirely:

```bash
python3 spacex.py 1    # Company Profile
python3 spacex.py 2    # Latest Launches
python3 spacex.py 3    # Upcoming Launches
python3 spacex.py 4    # Rockets & Launch Stats
python3 spacex.py 5    # Programs
python3 spacex.py 6    # Crew / Astronauts
python3 spacex.py 7    # Launchpads
```

This is useful for scripting, piping output, or quickly jumping to a specific view.

### Run All Sections

```bash
python3 spacex.py 0
# or just press Enter at the prompt
```

Runs every section in order. Section 4 (Rockets) paginates through all SpaceX launches to compute vehicle stats, so it makes several API calls and may take a few seconds longer than other sections.

---

## Sections

### 1 — Company Profile

**Command:** `python3 spacex.py 1`

Fetches the SpaceX agency record from The Space Devs database and displays:

| Field | Description |
|---|---|
| Name | Official company name |
| Founded | Year SpaceX was incorporated |
| Admin | Current administrator/CEO |
| Country | Country code of headquarters |
| Launchers | Rocket families operated |
| Spacecraft | Crewed/cargo vehicles operated |
| Description | Full company bio (truncated to 200 characters) |
| Wikipedia | Link to the SpaceX Wikipedia article |

**API call:** `GET /agencies/121/`

---

### 2 — Latest Launches

**Command:** `python3 spacex.py 2`

Displays the 5 most recent SpaceX launches (including those with future NET dates that are sorted last by the database), ordered by NET (No Earlier Than) date descending. For each launch:

| Field | Description |
|---|---|
| Name | Full mission name |
| Date | Launch date, formatted to second or day precision based on available data |
| Rocket | Rocket variant used (e.g., Falcon 9, Falcon Heavy) |
| Orbit | Target orbit abbreviation (LEO, GTO, PO, etc.) |
| Status | Launch outcome abbreviation (Success, Failure, TBD, TBC, Go, etc.) |
| Description | Mission description (truncated to 110 characters) |

**API call:** `GET /launch/?search=spacex&limit=5&ordering=-net`

**Status abbreviations:**

| Abbrev | Meaning |
|---|---|
| Success | Launch and deployment succeeded |
| Failure | Launch failed |
| Go | Confirmed, countdown is live |
| TBC | To Be Confirmed — date is plausible but not official |
| TBD | To Be Determined — date is a rough estimate |
| Hold | Launch is on hold |

---

### 3 — Upcoming Launches

**Command:** `python3 spacex.py 3`

Shows the next 8 confirmed or estimated SpaceX launches, ordered by NET date ascending (soonest first). For each entry:

| Field | Description |
|---|---|
| Date | NET date (second or day precision) |
| Status | Confidence level of the date (Go / TBC / TBD) |
| Probability | Launch probability percentage, if published |
| Name | Mission name |
| Weather concern | Any flagged weather rule violation, if published |

The total count of all upcoming SpaceX launches currently in the database is shown at the top.

**API call:** `GET /launch/upcoming/?search=spacex&limit=8&ordering=net`

---

### 4 — Rockets & Launch Stats

**Command:** `python3 spacex.py 4`

This is the most data-intensive section. It consists of two parts:

**Part 1 — Rocket Configurations**

Lists every SpaceX rocket variant on record, with:

| Field | Description |
|---|---|
| Full name | Rocket variant name (e.g., Falcon 9 Block 5) |
| Reusability | Whether the vehicle is designed for reuse |
| Wikipedia | Link to the variant's Wikipedia page, if available |

**API call:** `GET /config/launcher/?manufacturer__name=SpaceX&limit=20`

**Part 2 — All-Time Launch Count Bar Chart**

Paginates through the entire history of SpaceX launches in the database (100 results per page) and tallies how many launches each rocket variant has flown. Results are displayed as a sorted bar chart using block characters:

```
  Launch counts by vehicle (all time):
    Falcon 9 Block 5                      223  ████████████████████████████████████████
    Falcon Heavy                            4  ████
    Starship V3                             2  ██
    Starship V2                             2  ██
```

The bar is capped at 40 characters wide regardless of actual count, so the chart remains readable even for high-volume vehicles.

**API calls:** `GET /launch/?search=spacex&limit=100&offset=N` (repeated until all pages are exhausted)

---

### 5 — Programs

**Command:** `python3 spacex.py 5`

Lists the space programs that SpaceX has participated in as a launch provider or direct participant. This includes both SpaceX-originated programs (e.g., Starlink, Polaris, Commercial Crew) and external programs for which SpaceX has provided launch services (e.g., GPS, Iridium, ISS resupply).

For each program:

| Field | Description |
|---|---|
| Name | Program name |
| Start date | Official or estimated program start |
| End date | Program end date, or "ongoing" if still active |
| Description | Program summary (truncated to 120 characters) |
| Wikipedia | Link to the program's Wikipedia article |

**API call:** `GET /program/?agencies__id=121&limit=20`

---

### 6 — Crew & Astronauts

**Command:** `python3 spacex.py 6`

This section has two sub-views:

**Sub-view 1 — People Currently in Space**

A real-time snapshot of every person currently off Earth, regardless of their agency or vehicle. For each person:

| Field | Description |
|---|---|
| Name | Full name |
| Agency | Space agency abbreviation (NASA, ESA, RFSA, CNSA, SpX, etc.) |
| Nationality | Country of citizenship |
| Time up | Cumulative time spent in space on the current mission, parsed from ISO 8601 duration format (e.g., `P325DT6H47M` → `325d 6h 47m`) |

> Note: The Space Devs database includes "Starman" — the mannequin in Elon Musk's Tesla Roadster launched on the Falcon Heavy demo flight — as an entry with agency "SpX" and nationality "Earthling". This is a deliberate, well-known easter egg in the database.

**API call:** `GET /astronaut/?in_space=true&limit=30`

**Sub-view 2 — Recent SpaceX Crewed Dragon Missions**

Lists up to 8 recent Crew Dragon flights (both operational Crew missions and commercial Axiom Space missions), ordered by NET date descending. For each mission:

| Field | Description |
|---|---|
| Name | Full mission name |
| Date | Launch date |
| Status | Mission outcome |
| Description | Mission description (truncated to 110 characters) |

**API call:** `GET /launch/?search=crew+dragon&limit=8&ordering=-net`

---

### 7 — Launchpads

**Command:** `python3 spacex.py 7`

Lists the launch sites that SpaceX owns and operates (currently all at Starbase, TX). For each pad:

| Field | Description |
|---|---|
| Name | Pad name |
| Status | Operational status |
| Location | Site name and country code |
| Total launches | Cumulative launches from this pad |
| Wikipedia | Link to the pad's Wikipedia page |

> This section shows only pads owned/named by SpaceX. Shared pads such as SLC-40 and LC-39A at Kennedy Space Center are operated by the U.S. Space Force and leased to SpaceX, so they do not appear here.

**API call:** `GET /pad/?search=spacex&limit=20`

---

## API Reference

### Base URL

```
https://lldev.thespacedevs.com/2.2.0
```

This is the **development/free tier** endpoint of The Space Devs API. It is rate-limited but does not require authentication for read-only requests.

A production endpoint (`api.thespacedevs.com`) is also available and requires a paid API key for higher rate limits. To switch, change the `BASE` constant at the top of `spacex.py`:

```python
BASE = "https://api.thespacedevs.com/2.2.0"
```

### Endpoints Used

| Endpoint | Description |
|---|---|
| `GET /agencies/121/` | SpaceX company profile |
| `GET /launch/` | Launch history (paginated) |
| `GET /launch/upcoming/` | Upcoming launches |
| `GET /config/launcher/` | Rocket vehicle configurations |
| `GET /program/` | Space programs |
| `GET /astronaut/` | Astronaut profiles |
| `GET /pad/` | Launch pad information |

All endpoints return JSON. Pagination is handled via `limit` and `offset` query parameters. The response envelope for list endpoints looks like:

```json
{
  "count": 231,
  "next": "https://lldev.thespacedevs.com/2.2.0/launch/?limit=5&offset=5",
  "previous": null,
  "results": [ ... ]
}
```

### Rate Limits & Fair Use

The development endpoint (`lldev.thespacedevs.com`) is provided free of charge. The script is designed to be considerate:

- It sets a `User-Agent` header (`spacex-explorer/1.0`) so requests are identifiable.
- It uses a 15-second timeout per request to avoid hanging indefinitely.
- Section 4 is the only section that makes multiple sequential requests (one per 100-result page). On a database of ~300 SpaceX launches this means approximately 3–4 requests. Avoid running section 4 in a tight loop.

If you need to make frequent automated requests, consider signing up for a free API key at [thespacedevs.com](https://thespacedevs.com) and switching to the production endpoint.

---

## Code Structure

```
spacex.py
│
├── Constants
│   ├── BASE          — API base URL
│   └── SPACEX_ID     — SpaceX's agency ID in The Space Devs DB (121)
│
├── HTTP helpers
│   └── get()         — Builds URL, sets User-Agent, handles HTTP/network errors
│
├── Formatting helpers
│   ├── header()      — Prints a section title with box-drawing borders
│   ├── fmt_date()    — Converts ISO 8601 timestamps to readable strings,
│   │                   respecting the precision level reported by the API
│   └── truncate()    — Caps a string at N characters and appends "…"
│
├── View functions (one per section)
│   ├── show_company()
│   ├── show_latest_launches()
│   ├── show_upcoming_launches()
│   ├── show_rockets()
│   ├── show_programs()
│   ├── show_astronauts()
│   └── show_launchpads()
│
├── SECTIONS dict     — Maps key strings ("1"–"7", "0") to (label, function) pairs
│
└── __main__ block    — Parses argv[1] or reads from stdin; dispatches to view functions
```

All view functions follow the same pattern:

1. Call `get()` with the appropriate path and query parameters.
2. Print a `header()`.
3. Iterate over `results` and print formatted rows.

---

## Error Handling

The script handles two categories of network error:

| Error | Cause | Behaviour |
|---|---|---|
| `urllib.error.HTTPError` | Server returned a 4xx or 5xx status | Prints the HTTP status code and URL, then exits with code 1 |
| `urllib.error.URLError` | DNS failure, connection refused, timeout | Prints the reason string, then exits with code 1 |

Invalid section numbers passed via `argv[1]` print "Invalid choice." and exit with code 1.

No retries are attempted. If a request fails, fix the underlying connectivity issue and re-run.

---

## A Note on the Original SpaceX API

This script was originally designed for `api.spacexdata.com`, a community-maintained open-source REST API that provided SpaceX-specific data including:

- `/launches` — launch history and upcoming missions
- `/rockets` — rocket specifications
- `/crew` — Dragon crew members
- `/capsules` — serialized Dragon capsules
- `/cores` — booster core history
- `/starlink` — Starlink satellite orbital data
- `/roadster` — Tesla Roadster real-time position
- `/ships` — SpaceX recovery fleet
- `/launchpads` and `/landpads`

That API was **archived on June 6, 2026** and is no longer reachable (all endpoints return HTTP 521). The GitHub repository (`r-spacex/SpaceX-API`) remains available for historical reference, and database exports are hosted at `backups.spacexdata.com`.

The Space Devs API covers all the same launch, rocket, pad, crew, and program data. It does not have equivalents for Starlink orbital telemetry, booster core reuse history, or the Roadster tracker.

---

## Sample Output

```
══════════════════════════════════════════════════════════════
  SpaceX — Company Profile
══════════════════════════════════════════════════════════════
  Name          : SpaceX
  Founded       : 2002
  Admin         : CEO: Elon Musk
  Country       : USA
  Launchers     : Falcon | Starship
  Spacecraft    : Dragon
  Description   : Space Exploration Technologies Corp., known as SpaceX, is an American
                  aerospace manufacturer and space transport services company…
  Wikipedia     : https://en.wikipedia.org/wiki/SpaceX

══════════════════════════════════════════════════════════════
  Next 8 Upcoming SpaceX Launches
══════════════════════════════════════════════════════════════
  Total upcoming SpaceX launches in database: 66

  • 2026-06-20 06:39 UTC  [TBC]
    Falcon 9 Block 5 | Globalstar 2-R Mission 1 (x 9)

  • 2026-06-20 14:00 UTC  [Go]
    Falcon 9 Block 5 | Starlink Group 17-28

══════════════════════════════════════════════════════════════
  People Currently in Space
══════════════════════════════════════════════════════════════
  Currently in space: 11 people

  • Jessica Meir                         NASA      American         Time up: 325d 6h 47m
  • Starman                              SpX       Earthling        Time up: 3049d 5h 0m
  • Sergey Kud-Sverchkov                 RFSA      Russian          Time up: 383d 15h 27m

══════════════════════════════════════════════════════════════
  SpaceX Rocket Configurations
══════════════════════════════════════════════════════════════
  Total rocket configs on record: 13

  • Falcon 1                             [Expendable]
  • Falcon 9                             [Reusable]
  • Falcon 9 Block 5                     [Reusable]
  • Falcon Heavy                         [Reusable]
  • Starship                             [Reusable]
  ...

  Launch counts by vehicle (all time):
    Falcon 9 Block 5                      223  ████████████████████████████████████████
    Falcon Heavy                            4  ████
    Starship V3                             2  ██
```
