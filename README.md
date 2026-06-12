# Helldivers 2 — API Data Explorer

A single-file starter (`index.html`) that downloads everything from the
Helldivers 2 community APIs every 60 seconds and dumps it as plain text.
All data lands in one object — `apiData` — and a set of render functions
write it onto the page. Build your own site by replacing the render
functions with your own HTML.

> **Note on line numbers:** they point into `index.html` as of this commit.
> If you add or remove lines above a function, the numbers below shift.

---

## ⭐ RENDER FUNCTIONS — the ones that write information on the page

These all read from `apiData` and write plain text into a `<pre>` element.
**Replace these with your own HTML to build your site.**

| Line | Function | Writes to | What it shows |
|-----:|----------|-----------|---------------|
| 580 | `putTextInElement(elementId, text)` | any element | Small helper — sets `textContent` of an element by id |
| 586 | `renderEverything()` | — | Calls all the render functions below, in order |
| 599 | `renderWarStatistics()` | `#output-war-statistics` | Player count, impact multiplier, missions won/lost, kills, deaths |
| 615 | `renderAssignments()` | `#output-assignments` | Major Order + minor orders: title, deadline, briefing, each task with progress / target / planet / faction |
| 671 | `renderCampaigns()` | `#output-campaigns` | Every planet you can fight on: liberation or defense %, enemy regen, player count, plus each city/region's capture progress |
| 710 | `renderDefenseEvents()` | `#output-defense-events` | Planets under enemy attack: defense %, deadline, defenders, attacker |
| 729 | `renderPlanets()` | `#output-planets` | Enemy-held and battle planets: health, liberation %, regen, hazards, supply lines, active attack info |
| 775 | `renderNewsDispatches()` | `#output-dispatches` | Latest 10 in-game news messages |
| 790 | `renderSpaceStation()` | `#output-space-station` | DSS location, next-move vote timer, the 3 tactical actions (charge %, time until ready / ends / off cooldown) |
| 857 | `renderPlanetEffects()` | `#output-planet-effects` | **Per-planet modifiers & effects**: enemy fleets (Jet Brigade, Rupture Strain…), buffs (Dark Fluid Jetpack Augmentation…), debuffs (longer Eagle rearm…), anomalies — each with a description of what it does |
| 894 | `renderStatusBar()` | `#status-bar` | Data source badge (PRIMARY / FALLBACK / OFFLINE), last error, last updated time |

---

## Data fetching functions

| Line | Function | What it does |
|-----:|----------|--------------|
| 270 | `downloadJson(url, fetchOptions, retryCount)` | Fetches any URL as JSON with retries; a 429 (rate limit) waits a full rate window before retrying. Returns the parsed JSON or throws |
| 290 | `downloadFromPrimaryApi(path)` | Shorthand: primary API base URL + path, with the required headers attached |
| 306 | `downloadEverythingFromPrimaryApi()` | Downloads everything from api.helldivers2.dev in two rate-limit-safe waves (fast feeds every cycle; planets/dispatches/DSS every 5th cycle, cached in between). Fills `apiData` |
| 352 | `downloadEverythingFromBackupApi()` | Downloads everything from helldiverstrainingmanual.com and reshapes it to match the primary API format, so your code never cares which source ran. Fills `apiData` |
| 457 | `downloadSupplementalData()` | Runs after either API: grabs the planet effects list + war clock (backup-only data) and the effect names database |
| 498 | `overlayFreshBattleDataOntoPlanetList()` | The `/planets` feed can be minutes stale; overlays the fresh campaign/defense planet objects onto it so new attacks never go missing. Rebuilds `planetsByIndex` and the active-battle set |
| 545 | `downloadAllData()` | The orchestrator: picks the API per the server dropdown, downloads, post-processes. Returns `true` on success, `false` if all sources failed |
| 238 | `downloadEffectNamesDatabase()` | Downloads the community effect ID → name/description file once per session and merges it into `EFFECT_NAME_BY_ID` / `EFFECT_DESCRIPTION_BY_ID` |

## Helper functions (math & formatting)

| Line | Function | Returns |
|-----:|----------|---------|
| 143 | `getLiberationPercent(planet)` | 0–100 — how liberated an enemy planet is: `(1 - health/maxHealth) * 100` |
| 150 | `getDefenseProgressPercent(defenseEvent)` | 0–100 — how close a defense is to winning |
| 157 | `getEnemyRegenPercentPerHour(planet)` | % per hour the enemy regens; out-damage this or liberation falls |
| 163 | `planetIsUnderAttack(planet)` | `true` when `planet.event` is set |
| 168 | `getEnemyFactionOnPlanet(planet)` | `'automaton'` \| `'terminids'` \| `'illuminate'` \| `null` |
| 177 | `normalizeFactionName(factionName)` | any faction spelling → lowercase key, `null` for Humans |
| 186 | `formatDuration(totalSeconds)` | `"2d 6h"` / `"3h 14m"` / `"45m"` |
| 198 | `getSecondsUntil(isoDateString)` | seconds from now until the date (0 if passed, `null` if invalid) |
| 206 | `formatBigNumber(number)` | `"1.23M"` / `"45.6K"` |
| 215 | `getTaskValue(task, valueTypeId)` | one typed number out of an assignment task (use `TASK_VALUE_TYPE`) |
| 222 | `getEffectName(effectId)` | readable name, e.g. `1310` → `"Rupture Strain"` |
| 228 | `getEffectDescription(effectId)` | what the modifier does (e.g. Eagle rearm penalty), `''` if unknown |

## UI / refresh loop functions

| Line | Function | What it does |
|-----:|----------|--------------|
| 261 | `changeServerPreference(newPreference)` | Called by the Server dropdown — saves to localStorage, refreshes immediately |
| 917 | `refreshEverythingNow()` | Full download + render cycle right now; also wired to the Refresh button |
| 934 | `startCountdownTimer()` | 1-second ticker that refreshes everything when the countdown hits zero |

---

## Configuration constants (lines 73–85)

| Constant | Meaning |
|----------|---------|
| `PRIMARY_API_URL` | Main community API (v1) — rich data, rate-limited ~5 req/10s, needs headers |
| `PRIMARY_API_V2_URL` | Main API v2 — only used for the DSS (`/space-stations`) |
| `PRIMARY_API_REQUIRED_HEADERS` | `X-Super-Client` / `X-Super-Contact` — required by the primary API |
| `BACKUP_API_URL` | Fallback API — no rate limit, no headers, but no cities/supply lines |
| `EFFECT_NAMES_DATABASE_URL` | Community JSON mapping effect IDs → names + descriptions |
| `REFRESH_INTERVAL_SECONDS` | How often the page re-downloads (60) |
| `RATE_LIMIT_WINDOW_MILLISECONDS` | Wait between request waves so the primary API never 429s |
| `HEAVY_FEEDS_REFRESH_EVERY_N_CYCLES` | Big feeds (planets ~1MB) only re-download every Nth cycle |

## Lookup tables (lines 89–119)

| Table | Meaning |
|-------|---------|
| `FACTION_NAME_BY_ID` | `1`=Humans, `2`=Terminids, `3`=Automaton, `4`=Illuminate |
| `TASK_VALUE_TYPE` | What each number in `task.values` means: `FACTION_ID`=1, `TARGET_AMOUNT`=3, `PLANET_INDEX`=12 |
| `DSS_ACTION_STATUS_NAME` | `1`=CHARGING, `2`=ACTIVE, `3`=COOLDOWN |
| `EFFECT_NAME_BY_ID` | Effect ID → name snapshot; live database merged on top at startup |
| `EFFECT_DESCRIPTION_BY_ID` | Effect ID → what the modifier does; filled from the live database |

---

## The `apiData` object (line 124) — where all the data lives

Read everything from here, never from raw fetch results.

### `apiData.warStatistics`
- `.statistics.playerCount` — Helldivers online right now
- `.statistics.missionsWon` / `.missionsLost` / `.deaths`
- `.statistics.terminidKills` / `.automatonKills` / `.illuminateKills`
- `.impactMultiplier` — how much each mission counts. Goes **down** when more players are online (the game balances liberation speed against player count)

### `apiData.assignments` — Major Order + minor orders
Each assignment:
- `.id`, `.title`, `.expiration` (ISO date or null)
- `.briefing` — may contain `<i=N>…</i>` markup, strip before showing
- `.progress` — array of numbers, one per task (current value)
- `.tasks[]` — each task has `.type`, `.values[]`, `.valueTypes[]` → read with `getTaskValue()`
- Known task types: `3`=eradicate, `9`=complete operations, `11`=liberate planet, `13`=hold planet

### `apiData.activeCampaigns` — where you can fight right now
Each: `.id`, `.type` (`0`=liberation, `4`=defense), `.planet` (full planet object)

### `apiData.defenseEvents`
Full planet objects whose `.event` is filled in (planets under attack).

### `apiData.planets` — all ~269 planets
Each planet:
- `.index` — the ID everything else references planets by
- `.name`, `.sector`, `.biome.name`, `.hazards[]` (`{name, description}`)
- `.health` / `.maxHealth` — **lower health = more liberated**
- `.regenPerSecond` — enemy healing rate; see `getEnemyRegenPercentPerHour()`
- `.currentOwner` / `.initialOwner` — `"Humans"` / `"Terminids"` / `"Automaton"` / `"Illuminate"`
- `.waypoints[]` — planet indexes this planet connects to (supply lines)
- `.attacking[]` — planet indexes this planet is attacking
- `.regions[]` — cities: `.name`, `.size`, `.health`/`.maxHealth`, `.isAvailable`, `.players`.
  Capturing a region deals `region.maxHealth` damage to the planet, so the boost is `region.maxHealth / planet.maxHealth * 100` %
- `.event` — `null` normally; when under attack: `.faction`, `.health`/`.maxHealth` (**lower = defense doing better**), `.startTime`, `.endTime` (deadline)
- `.statistics.playerCount` — Helldivers on this planet

### `apiData.planetsByIndex`
Same planets keyed by index: `apiData.planetsByIndex[259].name`

### `apiData.indexesOfPlanetsWithActiveBattles`
`Set` of planet indexes with a live campaign or defense.

### `apiData.newsDispatches`
Each: `.id`, `.message` (may contain markup), `.published` (ISO date or null on backup)

### `apiData.spaceStations` — the DSS
From the **primary** API: `.planet` (full object), `.electionEnd` (ISO date),
`.tacticalActions[]` with `.name`, `.status` (1/2/3), `.statusExpire`,
`.costs[0]` (`.currentValue`, `.targetValue`, `.deltaPerSecond` — 0 means
charging is halted while another boost is active).
From the **backup** API: `.cameFromBackupApi: true`, `.planetIndex`,
`.electionEndWarTime` (compare to `apiData.currentWarTimeSeconds`), `.activeEffectIds[]`.

### `apiData.planetActiveEffects` — modifiers, fleets, buffs, debuffs
Flat list of `{index, galacticEffectId}` — one entry per effect per planet.
This is where enemy fleets (Jet Brigade, Rupture Strain…), **buffs**
(Dark Fluid Jetpack Augmentation…), **debuffs** (increased Eagle rearm
time…) and anomalies (Black Hole, Gloom…) live.
Resolve with `getEffectName(id)` and `getEffectDescription(id)`.
⚠️ This feed lags behind liberations — check `planet.currentOwner` before
showing enemy fleets on a planet.

### Bookkeeping
- `apiData.currentWarTimeSeconds` — game clock, for backup DSS countdowns
- `apiData.currentDataSource` — `'PRIMARY'` | `'FALLBACK'` | `null` (offline)
- `apiData.lastSuccessfulFetchTimestamp` — `Date.now()` of the last good refresh

---

## Internal tracking variables

| Line | Variable | Meaning |
|-----:|----------|---------|
| 234 | `effectNamesDatabaseWasDownloaded` | guards the once-per-session effect DB download |
| 255 | `SERVER_PREFERENCE_STORAGE_KEY` | localStorage key for the server choice |
| 257 | `serverPreference` | `'auto'` \| `'live'` \| `'backup'` |
| 296 | `refreshCycleCounter` | decides when the heavy primary feeds are due |
| 298 | `heavyFeedCache` | cached planets / dispatches / DSS between heavy refreshes |
| 452 | `supplementalCycleCounter` | same idea for the supplemental backup status |
| 453 | `cachedBackupWarStatus` | last `/war/status` response (effects + war clock) |
| 541 | `lastErrorMessage` | last download error, shown in the status bar |
| 913 | `secondsUntilNextRefresh` | the on-screen countdown value |
| 914 | `countdownTimer` | the `setInterval` handle |

---

## APIs used

| API | Used for | Notes |
|-----|----------|-------|
| `api.helldivers2.dev/api/v1` | everything | ~5 req/10s limit, requires `X-Super-Client` + `X-Super-Contact` headers |
| `api.helldivers2.dev/api/v2` | `/space-stations` (DSS) | same limit/headers |
| `helldiverstrainingmanual.com/api/v1` | fallback for everything, plus `/war/status` for planet effects + war clock even in primary mode | no limit, no headers, but no cities/supply lines |
| `raw.githubusercontent.com/helldivers-2/json` | effect ID → name + description | downloaded once per session |
