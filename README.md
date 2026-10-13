# 80/20 Workout Library Lookup

A lightweight single-page app for quickly looking up workouts from the [80/20 Endurance](https://www.8020endurance.com/8020-workout-library/) workout library.

## Usage

Open `index.html` in any browser. Type a workout code (e.g. `RRe9`, `CFo3`, `SCI1`) and the matching workout details appear instantly.

## Features

- **479 workouts** across Run (190), Bike (218), and Swim (71)
- Instant filter-as-you-type search across codes, categories, sport, and descriptions
- Tap a workout (or press Enter on a code) to open a full workout view with a shareable link
- Workout view breaks each workout into numbered steps (with repeat blocks), a zone breakdown, and a tappable timeline with time/distance markers
- Optional personal zone ranges (heart rate, pace or power) per sport, shown next to every step — copy them from the 80/20 Zones app or Zone Calculator
- **Recently viewed** — the last workout you opened is shown at the top when you reopen the app, so you don't have to retype it (stored locally in your browser)
- Case-insensitive matching
- Color-coded zone indicators (Zone 1 through Zone 5)
- Sport badges (Run / Bike / Swim)
- Dark theme with 80/20 branding
- Zero dependencies — single HTML file, no build step

## Workout Categories

| Sport | Categories |
|-------|-----------|
| Run | Recovery (RRe), Foundation (RF), Aerobic Intervals (RAe), Long (RL), Cruise Intervals (RCI), Tempo (RT), Fast Finish (RFF), Long Fast Finish (RLFF), Anaerobic Intervals (RAn), Short Intervals (RSI), Hill Repeats (RHR), Mixed Intervals (RMI), Speed Play (RSP), Long Intervals (RLI), Long Speed Play (RLSP), Long Marathon Simulator (RLMS), Taper (RTa) |
| Bike | Recovery (CRe), Foundation (CF), Aerobic Intervals (CAe), Cruise Intervals (CCI), Tempo (CT), Fast Finish (CFF), Force (CFo), Mixed Intervals (CMI), Anaerobic Intervals (CAn), Speed Play (CSP) |
| Swim | Recovery (SRe), Foundation (SF), Aerobic Intervals (SAe), Cruise Intervals (SCI), Tempo (ST), Mixed Intervals (SMI), Short Intervals (SSI), Speed Play (SSP), Taper (STa), Time Trial (STT) |

## Data Source

All workout data is sourced from the [80/20 Endurance Workout Library](https://www.8020endurance.com/8020-workout-library/). Workouts are copyright Matt Fitzgerald and David Warden and are provided for personal use only.
