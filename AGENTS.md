# AGENTS.md

pybaseball pulls baseball statistics into pandas DataFrames: it scrapes Baseball Savant (Statcast),
Baseball Reference, and FanGraphs, and downloads the Lahman and Retrosheet databases and the Chadwick
player register from GitHub. There is no server or CLI; the product is the functions re-exported in
`pybaseball/__init__.py`. Most bugs are "the page changed", so check the live HTML before changing a parser.

## Layout

- One module per data source at the top of `pybaseball/`. Baseball Reference: `standings.py`,
  `team_results.py`, `team_game_logs.py`, `league_*_stats.py`, `amateur_draft*.py`, and the `*_bref`
  functions in `team_*.py`. FanGraphs: `batting_leaders.py`, `pitching_leaders.py`, the rest of `team_*.py`.
  Savant: `statcast*.py`. GitHub: `lahman.py`, `retrosheet.py`, `playerid_lookup.py`.
- `pybaseball/datasources/` fetches and parses per site. `pybaseball/datahelpers/` is shared, network-free
  helpers. `pybaseball/cache/` is the opt-in DataFrame cache (`@cache.df_cache()` on about 40 public
  functions; files in `~/.pybaseball/cache`).
- `tests/pybaseball/` mirrors the package and runs offline. `tests/integration/` hits live sites.

## Commands

CI runs Python 3.8 to 3.11 (`.github/workflows/pytest.yml`). 3.12 to 3.14 install and run but hit the known
failures below. If no supported interpreter is active, create one here instead of searching the machine:

```bash
python3.11 -m venv .venv && source .venv/bin/activate   # or: uv venv --python 3.11 --seed .venv (--seed adds pip)
make install                                  # pip install -e .[test]; uses whatever `pip` is on PATH, so activate first
pip install "pandas<3"                        # setup.py has no upper bound; pandas 3 fails 13 unit tests
make test TEST_FLAGS="-n 5"                   # unit tests only (tests/pybaseball), same as CI, about 10 s
pytest tests/pybaseball/test_batting_leaders.py -q   # one file; add -k <name> for one test
make mypy ONLY_MODIFIED=0                     # whole package; about 170 pre-existing errors, CI ignores them
python -c "from pybaseball import standings; print(standings(2024)[0])"   # smoke run: one live Baseball Reference call, returns a list of DataFrames
```

Known failures on a clean checkout, not caused by your change: on Python 3.12 to 3.14, 8 tests in
`tests/pybaseball/cache/` fail with `'called_once_with' is not a valid assertion` (misspelled mock
assertions in the tests). `make validate-cache` fails with a 403 from FanGraphs. `make mypy` without
`ONLY_MODIFIED=0` exits 2 with "Missing target" on a clean tree. There is no lint step; lines are 120 chars.

## Conventions

- A unit test loads a saved page from `tests/pybaseball/data/<name>.html`, feeds it through the
  `response_get_monkeypatch` fixture (`bref_get_monkeypatch` for Baseball Reference), and compares to
  `tests/pybaseball/data/<name>.csv` with `pd.testing.assert_frame_equal(..., check_dtype=False)`. Copy
  `tests/pybaseball/test_batting_leaders.py`. Autouse fixtures in `tests/pybaseball/conftest.py` make every
  `requests.*` call raise; `BRefSession` is not blocked, so always mock it. A test that needs the network
  belongs in `tests/integration/`.
- Baseball Reference blocks clients that exceed about 10 requests per minute for an hour (docstring in
  `pybaseball/datasources/bref.py`), so every request to it goes through the `BRefSession` singleton:
  curl_cffi with Chrome impersonation and a 10-per-minute throttle. Never call that site with `requests.get`.
  Every other site uses `requests.get` directly.
- A new public function is three edits: the function itself, type-annotated (`mypy.ini` sets
  `disallow_untyped_defs`), its re-export in `pybaseball/__init__.py`, and a `docs/<function>.md` page.

## Don't

- Don't run bare `pytest` from the root. `pytest.ini` points it at all of `tests/`, which adds 706
  integration tests that hit live sites, sleep 6 seconds between tests in several files, and parametrize
  over every season since 1876.
- Don't edit `pybaseball/data/fangraphs_teams.csv`; `make generate-team-ids` regenerates it.
- Don't hand-edit `tests/pybaseball/data/*.html` or `*.csv`; they are captured pages and their expected
  output. If a site changed, re-capture the page.
- `scripts/` is in `.gitignore` but tracked; a new file there needs `git add -f` or it never shows up.
