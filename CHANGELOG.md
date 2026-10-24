# Changelog

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
