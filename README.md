# Yahoo Fantasy league export (personal, read-only)

A small personal script that reads my own Yahoo Fantasy Football league once or twice a week
and saves it to a JSON file on my home server, so I can review my roster, matchup and free agents.

- Personal, non-commercial use for a single private league that I play in.
- Read-only: uses the Fantasy Sports Read permission. It never changes lineups, makes claims or trades.
- Runs a few times a week. No data is shared, sold or published.
- Keys and tokens stay on my own machine and are not in this repository.

## Files
- `yahoo_export.py`: exports league settings, standings, my roster, this week's opponent, waivers and free agents.
- `yahoo_auth.py`: one-time OAuth sign-in.

## Setup
1. Create a Yahoo app with Fantasy Sports Read and redirect URI `https://localhost:8000`.
2. `python3 -m venv venv && venv/bin/pip install yahoo_oauth yahoo_fantasy_api`
3. Put the client ID and secret in `oauth2.json` (not committed).
4. Run `venv/bin/python yahoo_auth.py`, then `venv/bin/python yahoo_export.py`.
