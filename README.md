# MLBC Sim League — 2001 Season Dashboard

An interactive single-file web dashboard for the [MLBC Sim League](https://mlbcsimleague.com), built by reverse-engineering High Heat Baseball 2003 simulation export files. All statistics are parsed directly from the game's `BoxScores.txt` play-by-play log — no manual entry.

---

## Live Dashboard

Open `mlbc_dashboard.html` in any modern browser. No server, build step, or dependencies required.

---

## Features

### Standings
Full 30-team standings split into American and National Circuits, with W, L, PCT, GB, Runs, Runs Allowed, and Run Differential.

### Stat Leaders
Fourteen leaderboard categories organized into three sections:

**Batting**
| Stat | Description |
|------|-------------|
| HR | Home runs (season total via cumulative event log) |
| SB | Stolen bases (season total via cumulative event log) |
| RC | Runs Created (Technical RC formula) |
| bWAR | Batting Wins Above Replacement (wOBA-based) |
| OPS | On-base plus slugging |
| OPS+ | OPS adjusted for league average (100 = league avg) |
| wOBA | Weighted On-Base Average (2001-era linear weights) |

**Pitching**
| Stat | Description |
|------|-------------|
| pWAR | Pitching Wins Above Replacement (FIP-based) |
| ERA | Earned Run Average |
| xFIP | Expected FIP (normalizes HR allowed to league HR/FB rate) |
| QS | Quality Starts (6+ IP, 3 or fewer ER) |
| WINS | Pitcher wins (season total) |
| SAVES | Saves (season total) |

**Team**
- Run Differential leaders (positive and negative)

### Box Scores
Browse all 1,880 regular season games. Filter by team name. Click any game to see the full line score.

### Hall of Fame
Career stats for players inducted before the 2001 season: Randy Johnson, Roger Clemens, Ellis Burks.

---

## Data Sources & Parsing

All data is sourced from High Heat Baseball 2003 simulation export files:

| File | Contents |
|------|----------|
| `BoxScores.txt` | 21MB play-by-play log, 2,234 total games |
| `HallOfFamers.txt` | Career stats for retired Hall of Famers |
| `Season.hh` | Binary file: season metadata, team names, schedule path |
| `Players.hh` | Binary file: 3,213 player records (768 bytes each) |

### Parsing Approach

**Standings** are derived by counting wins and losses from `FINAL` score lines across all 1,880 regular season games.

**Counting stats** (HR, SB) use the cumulative season total embedded in each event log entry. High Heat encodes the running season total directly in every HR and SB event — e.g. `HR: Montgomery (47, 5th inning off...)` — so the maximum value seen across all games equals the season total. Simple occurrence counting significantly undercounts these stats.

**Batting lines** are parsed from fixed-width box score sections:
```
Mayday Montgomery cf    5  2  3  1  1  0  2  .340
                       AB  R  H BI BB SO LO  AVG
```

**Pitching lines** use full names (not last-name collapse) to avoid collisions between players sharing a last name. 21 last-name collisions exist across 373 pitchers tracked.

**Extra events** (2B, 3B, HBP, SF, CS) are parsed from `BATTING` narrative sections following each team's box score.

---

## Stat Methodology

### Runs Created (RC)
Uses the Bill James Technical RC formula:

```
A = H + BB + HBP - CS
B = TB + 0.26×(BB + HBP) + 0.52×(SF + SB)
C = AB + BB + HBP + SF

RC = (A × B) / C
```

### wOBA
2001-era linear weights:
```
wOBA = (0.69×BB + 0.72×HBP + 0.89×1B + 1.27×2B + 1.62×3B + 2.10×HR) / PA
```

### OPS+
```
OPS+ = 100 × (OBP/lgOBP + SLG/lgSLG − 1)
```
League averages: OBP .348, SLG .388 (324 qualifying batters, 200+ PA)

### bWAR (Batting WAR)
```
wRAA     = (wOBA − lgwOBA) / wOBA_scale × PA        # wOBA_scale = 1.25
BsR      = SB×0.2 − CS×0.4                          # baserunning
PosAdj   = positional_value × (PA / 600)             # scaled to playing time
RepLevel = 20 runs / 600 PA × PA                     # replacement baseline

bWAR = (wRAA + BsR + PosAdj + RepLevel) / 10
```

Positional adjustment values (runs per 162 games):

| Pos | Adj |
|-----|-----|
| C | +12.5 |
| SS | +7.5 |
| 2B | +2.5 |
| 3B | +2.5 |
| CF | +2.5 |
| LF/RF | −7.5 |
| 1B | −12.5 |
| DH | −17.5 |

### pWAR (Pitching WAR)
FIP-based, with replacement level set at lgERA × 1.37:

```
FIP      = (13×HR + 3×BB − 2×SO) / IP + FIP_constant
FIP_c    = lgERA − (13×lgHR/9 + 3×lgBB/9 − 2×lgK/9) / 9

RepERA   = lgERA × 1.37
pWAR     = (RepERA − FIP) / 9 × IP / RPW            # RPW = 9.0
```

League-wide 2001: ERA 4.83, FIP constant 3.28, HR/9 1.05, BB/9 3.54, K/9 6.12

### xFIP
Replaces actual HR allowed with expected HR based on league HR/9 rate:
```
xFIP = (13×(IP/9×lgHR9) + 3×BB − 2×SO) / IP + FIP_constant
```

---

## 2001 Season Highlights

**Batting**
- **Mayday Montgomery** (Lake Elsinore) led the league with 60 HR, 184.9 RC, and 9.7 bWAR
- **Zan Allen** (Omaha) posted the highest OPS (1.078), OPS+ (+192), and wOBA (.465) despite Omaha finishing 52-75
- **Chad Darensbourg** (Edmonton) stole 76 bases, leading a dominant Edmonton team that went 78-51

**Pitching**
- **Don Nimmons** (Corpus Christi) led all pitchers with 10.7 pWAR, a 2.74 ERA over 259.2 IP, and 28 Quality Starts (76% QS rate)
- **Forrest Fenn** (Hillsboro) posted the best xFIP (3.49) and a 3.03 FIP despite a 4.20 ERA — significant positive regression candidate
- **Oscar Craigen** (Dayton) led the league with 37 saves

**Teams**
- **Edmonton Trappers** finished 78-51 with a +194 run differential, the best in the league
- **Sugar Land Space Cowboys** finished 47-84 (−158 run differential), the worst record and run differential in the league

---

## Binary File Reverse Engineering

`Players.hh` and `Season.hh` were decoded from scratch:

**Players.hh** — 3,213 player records, 768 bytes each
- Offset +8: Last name (null-terminated, 16 chars max)
- Offset +24: First name (null-terminated, 16 chars max)  
- Offset +93–110: Player ratings (1–10 scale)

**Season.hh** — Contains version string (`4.0.09`), season identifier (`Season2002`), schedule path, injury data, and all 30 team names.

---

## League Structure

30 teams across two circuits:

**American Circuit** — Edmonton Trappers, Corpus Christi Hooks, Portland Pickles, Oklahoma City RedHawks, Dayton Dragons, Frisco RoughRiders, Peoria Chiefs, Peninsula Pilots, Lake Elsinore Storm, Midland RockHounds, Mahoning Valley Scrappers, Wisconsin Timber Rattlers, Idaho Falls Chukars, Bangor Blue Ox, Auckland Tuatara

**National Circuit** — Chesapeake Baysox, San Francisco Seals, Richmond Flying Squirrels, Louisville Bats, Casper Ghosts, Hillsboro Hops, Lynchburg Hillcats, Norwich Sea Unicorns, Rocket City Trash Pandas, Altoona Curve, El Paso Chihuahuas, Scranton Wilkes-Barre Yankees, Omaha Storm Chasers, Maine Guides, Sugar Land Space Cowboys

---

## Dashboard Design

- Retro CRT terminal aesthetic — green phosphor on near-black
- Fonts: [Bebas Neue](https://fonts.google.com/specimen/Bebas+Neue) (headers) + [IBM Plex Mono](https://fonts.google.com/specimen/IBM+Plex+Mono) (data)
- Scanline overlay via CSS `repeating-linear-gradient`
- Zero runtime dependencies — pure HTML/CSS/JS, single file

---

## Repository Structure

```
mlbc_dashboard/
├── mlbc_dashboard.html    # Complete dashboard (single file, open in browser)
└── README.md
```

The raw simulation export files (`BoxScores.txt`, `Players.hh`, `Season.hh`, etc.) are not included in this repository as they are game data files from the MLBC Sim League.
