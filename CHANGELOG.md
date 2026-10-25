# Changelog

## 0.6.1 (beta)

The first beta. It rolls up everything since 0.5.0 (the 0.6.0 section below was an internal integration build) and adds the exit
summary fix and the slider findings. Summary of what is new since 0.5.0: player names on rolls for both teams, the mod list per run,
the Modder's map with the sliders measured in real games, share packs / summaries / the comparison card, the merged label table
(535 draw sites: 211 PROVEN, 274 LIKELY, 50 GUESS), a StateLog reader thread so busy PCs lose no data, and the exit summary fix.

Versions: tracer (rolltrace.dll) 0.5.1, Mod Manager plugin 0.6.0, app 0.6.1.

Sliders measured in real games.
- Ten sweep games (sliders at 0 or 100, one group at a time) were read against stock games. New in the labels and the Modder's map:
  **Fumbles is ball security, higher means fewer fumbles** (0 gives a fumble chance of exactly 1.0, 100 exactly 0; PROVEN). Holding,
  Clipping and the face-mask / pass-interference pair scale their own foul kind of the contact-foul roll (PROVEN). Defensive holding
  has no effect of its own (PROVEN). Pass defense reaction time multiplies the break-on-ball chance by x0.5 at 0 and x1.5 at 100
  (PROVEN; that roll's label went from LIKELY to PROVEN). WR catching, QB accuracy, tackling, offside, roughing the passer and
  blocking are LIKELY. Interceptions and pass coverage show nothing yet.
- The map's slider rows say how each was measured, and the map's check accepts measured sliders with a "how we know" note.

Exit summary (tracer).
- The log now ends with its exit summary when the game closes itself. The game never exits through the normal Windows exit path:
  on a window close (and on Quit) it calls TerminateProcess on itself, which skips the DLL's exit code, so 0.4.0 to 0.5.0 never wrote
  the `"why":"exit"` line in a real game. The DLL now wraps the game's own TerminateProcess import: when the game ends itself, the
  writer is parked between passes, its buffer is flushed and a fresh exit summary is written, then the game's call goes on unchanged.
  The line adds `"via":"TerminateProcess"` (or `"DllMain"` for a normal exit), `"exit_code"`, `"caller_rva"` and `"writer_wait_ms"`.
  `exit_hook = 0` in rolltrace.ini turns it off. A kill from outside the game (Task Manager) still leaves no exit line.
- `tools/rolltrace-report.js` prints how the exit summary was written.

Player names (checker).
- The check that judges names against each snap was fixed: the 73 "impossible" names in the 0.6.0 test game were all on punts and field goals (special-teams units mix linebackers and backs into the kicking team and receivers into the return team), and the DLL had named every one of them right.

## 0.6.0 (internal build, folded into 0.6.1)

Versions: tracer (rolltrace.dll) 0.5.0, Mod Manager plugin 0.6.0, app 0.6.0. Merges the player-name fix (tracer 0.4.2), the mod
list (tracer 0.4.3 + plugin 0.5.1), the StateLog reader thread, sharing with its review fixes, the modder's map and the merged
label table.

Player names (tracer).
- Draws name the right player on both teams. The player objects a draw carries are the 22 on-field slots, and the game refills a
  slot with whoever is on the field each play. 0.4.1 named a slot once and kept that name all game, and it read the in-game ratings,
  which carry a skill boost for the CPU team, so it named only your team. In the 0.4.1 smoke game, 626 of its 1,269 named entries on
  scrimmage plays put an offensive player on a defensive snap or the reverse.
- The DLL now reads each slot's roster ratings again on every writer pass and matches all 15 exactly (`"pgid_by":"roster"`). A player
  attribution has not read (a lineman, a sub) is looked up in the game's player table between plays and named from then on. Until then
  the draw carries `"fp"` (his 15 ratings), and the app names him from the player lines (`"pgid_by":"roster_app"`).
- The app drops the old 0.4.1 `"pgid_by":"ratings"` names and adds a `pgid` column to the decision rolls.
- A draw is named from the read of its slot made around the draw itself, not from a later read: a draw made just before a slot was
  refilled is written `"pgid_by":"roster_late"` with no name, never with the next player's name.
- Two players with the same 15 ratings stay unnamed even when only one of them was read. The DLL counts every fingerprint once in
  the game's player table and writes the answer as an `"fpq"` line, which the app also uses.
- The tracer's database statements run at raised priority, so it never holds the game's database lock while it waits for a core.
  The stats line shows every statement's time: `stmt_us_max`, `stmt_us_p95` and `stmt_over_4ms`.
- The `pgid` column holds the draw's subject only. A new `pgids` column lists every named player by role.

Mod list (plugin + tracer + app).
- Each log records which mods the game was launched with. At launch the plugin reads the Mod Manager pack being applied and writes
  `modlist-<date>-<time>-<pid>.json` beside the logs: each enabled mod's file name, title, version, size and hashes, plus the
  plugins in the Plugins folder. The DLL copies it into the session line as `run.mods` (matched on the game's PID and process start
  time; it waits at most 2 s for the file); without one it says `"state":"unknown"` and why. File names only, never folder paths.
  The plugin hashes only mod headers at launch (whole-file hashes come from a cache and a background pass), and the set id is stable.
- The Runs page has a Mods column, the run details list the mods, Compare names each side's mods, and share packs, summaries and the
  comparison card carry the list (a stock launch reads "no mods (stock)"; an old log reads "not recorded"). `modlist = 0` in
  plugin.cfg switches it off.

StateLog reader (tracer).
- A small normal-priority reader thread now reads every StateLog block as the game publishes it (polled every `statelog_poll_ms`,
  20 ms; no per-pass cap). On a busy PC the old tail inside the writer pass (at most 400 blocks a pass) was lapped by the game's ring:
  1.69 M of 12.3 M blocks were lost in the 10-24 data-2 game. The reader only copies and counts and queues what the writer needs;
  full queues are counted (`sl_qdrop`, `slw_qdrop`), never waited on. `statelog_reader = 0` in rolltrace.ini keeps the old tail.
  The stats and exit summary lines carry `sl_lost`, `sl_qdrop`, `slw_qdrop` and `sl_reader`.

Labels (tracer + app).
- The shipped label table is the merged table (passes A-E): 535 rows, every draw site labelled (211 PROVEN, 274 LIKELY, 50 GUESS in 0.6.1),
  84 key rows (about 1.7 more roll lines per play than before). `fumble.roll` and `tackle.preroll` are PROVEN. Shipped as
  `rolltrace-labels.tsv` beside the DLL and compiled in (sha256 `b0b57d17...`).

Tests.
- The browser tests end their own headless Edge (its process tree plus every process on the run's own profile) and stay off ports
  8790-8799.

Sharing results.
- **Share packs (.rlab):** `ratings-lab pack <run>` or **Share pack** on the Runs page writes one run (its roll log plus a manifest with
  pack format, tracer and app version, game build, style, difficulty, slider values and hash, mods, date, play count and the sha256
  of every file) as a compressed file. Windows user names, absolute paths (including OneDrive) and the PC name are scrubbed from every
  text file first, the replacements are reported, and saves are never included. `ratings-lab import`, **Import pack** and
  drag-and-drop take one in: the checksum and every sha256 are verified, damaged or edited packs and packs from a newer format are
  refused with a plain message, and the run appears as a normal run marked "imported" that you can compare with your own.
- **Summary shares (.rlsum.json):** `ratings-lab summary <run|tag>` or **Summary** on a run: per roll site the count, threshold mean
  and spread, hit rate and the manifest fields, scrubbed, a few KB. Imported summaries are summary-only runs that the Compare page
  can diff against your runs or other summaries (aggregates only: no draw sites, curves or knob values).
- **Mod comparison card:** **Save comparison card** on the Compare page (or `ratings-lab card <A> <B>`) writes one self-contained,
  1200 px wide HTML page listing which rolls moved above a noise floor, with both sides' manifest lines. HTML only, no PNG.
- New commands: `pack`, `import`, `summary`, `card`, `peek`. The Runs and Compare pages, and `/api/runs`, know about imported runs and summaries.

Sharing fixes (review of 2026-10-24, before any release).
- **Unpacking is capped (zip bombs).** A pack holds at most 4 files: its own roll log plus result.json, rolltrace.log and plugin.log.
  Each file stops unpacking at its cap (512 MB for the log, 8 MB for the others, 200 times its packed size), whatever size the
  manifest claims, and a pack file is at most 200 MB. Importing streams each file to disk instead of unpacking it in memory, and a
  refused import leaves no folder behind. `peek` no longer unpacks anything.
- **An imported pack can never replace or stand in for one of your runs.** It is imported under its own id (the sender's id plus
  `-imp-` and 8 letters of the pack checksum). A pack with a second roll log, or a log not named after its run id, is refused. The run
  list reads your folders apart from the imported folder, so a bigger copy in there can never win over your own log, and stray files or
  half-finished imports in that folder are ignored. Manifest field types are checked, run ids with dots, odd names or device names are
  refused, and an imported result.json is cut to its test id and verdict (a path-like id is dropped).
- **The comparison card no longer credits game-to-game variation to the mod.** Logs, summaries and comparisons now carry per-game
  numbers. With 2 or more games per side the card tests game against game (Welch t-test over games); with 1 game on one side it uses the
  other side's spread; with 1 game per side it says "1 game per side: differences may be noise between games", calls itself "what
  differed", and uses a floor set above what 4 real stock games differed by (threshold x3, rolls per play x5, hit rate 15 points).
  The hit-rate test now needs 10 rolls a side and |z| 3; a roll seen on one side only counts only when the other side's play count
  makes missing it near impossible, and never when the sides used different tracer versions; the card prints how many flags chance
  alone would give. See the user guide, section 10.
- **Scrubbing.** JSON (logs, result.json, manifests, summaries) is scrubbed value by value: keys, numbers and structural values (line
  types, labels, field names) are never changed, so a user name like "max" or "player" can no longer corrupt a log or a summary. User
  folders with spaces or apostrophes, `\/`-escaped paths and paths at the end of a value are removed whole. New option **Strip player
  names** (Runs page) / `--strip-players`: player names become `<player>`, ids stay. result.json is packed as `{id, verdict}` only;
  summary names and download file names (a tag holding your user name) are scrubbed; leftover warnings no longer print letters of the
  name. The Runs page and the CLI say what is not removed (team names, mod titles, notes, player names unless stripped).
- Summaries refuse missing or negative counts, hits above rolls, a non-list run list and malformed per-game rows. API routes refuse
  requests that another web site starts (Sec-Fetch-Site), so a foreign page cannot trigger builds or packs.
- Tests: `app/test/test-share-safety.js` (zip bombs, run takeover, leftovers, ids, scrubbing cases, the card on synthetic multi-game
  logs: two stock sides that differ only between games flag nothing, a real change is found); the share, app and browser tests updated.

Modder's map.
- **Modder's map page** (top bar): "what moves fumbles / INTs / broken tackles", one card per outcome (15 outcomes plus home-field
  modifiers): the deciding rolls with their PROVEN / LIKELY badge, each linked to that roll's page in your runs; the inputs (ratings,
  tuning knobs, curves, sliders) with an arrow for what the outcome does when the input goes up and whether the link is proven,
  likely or predicted (every slider stays predicted until the slider sweep); and the in-game check that would prove it. Search box,
  one-outcome links (`#/map/<outcome>`), **Map CSV**, and the "what needs a live run" and ranked differential-run lists.
- Each roll's page has a **Used in** line with the outcomes it belongs to (and says so when the map's confidence differs from the
  database's label).
- The map ships with the tool: `tools/make-data-pack.js` copies `modders-map.json` (schema `ratings-lab-modders-map/1`, checked
  first) into `data\` next to the field index; the app serves it at `/api/modders-map` (also found next to the field index, or
  `ROLLTRACER_MODDERS_MAP`), `/api/status` reports it, and `serve.js` serves it too. The viewer carries a bundled copy
  (`viewer/js/modders-map.bundle.js`, made by `node viewer/tools/build-modders-map.js`; `--check` reports a stale bundle), so the
  page works offline and from disk.
- Tests: the app, viewer and browser checks cover the route, a bad or broken map file, the data pack, search, the roll links, the
  "used in" line, the offline fallback and phone width. Test servers now stay off ports 8790-8799.

## 0.5.0 (pre-release)

Roll Tracer is now **Ratings Lab**. It traces more than rolls (random draws, rating-to-chance curves, thresholds, and which ratings
and sliders feed each outcome), so the name changed. Nothing about how the game is read changed.
- App, viewer, command line and README renamed: `ratings-lab.cmd` / `ratings-lab` (the old `roll-tracer.cmd` still starts it).
- The app keeps its settings, tags and databases in `%APPDATA%\RatingsLab`. If `%APPDATA%\RollTracer` exists and the new folder does
  not, it is copied over once; the old folder is left alone.
- The Mod Manager lists the plugin as "Ratings Lab". The game-side DLL is the 0.4.1 build, unchanged; plugin and log file names keep
  the old names (`RollTracePlugin.dll`, `Plugins\RollTrace`, `rolltrace-*.jsonl`), so 0.4.x installs, logs and lab-run folders are read as before.

## 0.4.1 (pre-release)

Found by the first full game with 0.4.0 in the real game, which passed (no crash, 0 dropped draws).
- Players are named by a ratings fingerprint. The player-id learner of 0.4.0 never settled on a place in the player object, so draws
  carried no player; 0.4.1 matches each player object's ratings against the roster and logs the id with `"pgid_by":"ratings"`.
- Difficulty is logged as your setting plus each team's effective value (for example setting 3 / 3, effective 3 / 1).
- Logs are much smaller: about 22 to 54 MB for a full game instead of 128 MB.
- The app reads the style, difficulty and slider hash from the first game line (0.4.x logs carry them there, not in the session line),
  so the Runs page and Compare show them instead of "unknown". Difficulty shows the per-team effective value when the log has it.
- The fumble roll is proven in the game by an A/B run (fumble chance x10: threshold mean 0.104 to 0.533, hit rate 5.3% to 52.6%).
- Screenshots of real logs in the README.

## 0.4.0 (pre-release, first packaged build)

First build of Roll Tracer as a standalone tool for gameplay modders. The plugin has been tested offline only; its first in-game
smoke test is still to come, so treat this as pre-release.

Plugin (log only, every roll is passed back to the game unchanged):
- Logs each labelled gameplay draw: the drawn value, the threshold it was compared with when that can be read, hit or miss, and the
  player involved. 276 labelled sites, 43 of them key decisions (catches, drops, interceptions, fumbles, broken tackles, kicks,
  penalties, pass-rush and block engagements and more).
- New labels: the fumble roll, the tackle pre-roll, block engagement rolls, and several smaller sites.
- Logs every tuning-curve evaluation (which curve, which rating fed it, which game-style copy), and who did what on each play.
- Records the game style, difficulty and slider settings of each game, and stamps every log with the plugin build, the labels table and
  the game version.
- Defaults are set for leaving it on while you play: one log per launch, a size cap, a free-disk floor, old logs trimmed, no stage expiry.

App and viewer:
- One command or double-click: finds the plugin's logs, builds the analysis database, opens the viewer on 127.0.0.1.
- Decisions, fields, curves and sliders pages with evidence levels; search; what-if calculator; target-rate solver.
- Runs page with tags; Compare (tag A vs tag B) with rates, 95% intervals, knob values and curve changes; warnings when sides differ.
- CSV and HTML report export. Command-line: where, refresh, runs, tag, untag, compare, export.
- Bundled field index (data/), so nothing outside the package folder is needed.
