#!/usr/bin/env python3
"""Read-only export of a Yahoo Fantasy Football league to JSON.

Runs on your own box. Your Yahoo keys and token stay in oauth2.json next to
this script and never leave the machine. Output: league_YYYYMMDD_HHMM.json
(plus latest.json) in OUT_DIR.

Setup:  pip install yahoo_oauth yahoo_fantasy_api
First run is interactive (opens a Yahoo authorize URL, you paste the code);
after that the refresh token renews itself.
"""
import json, os, sys, datetime as dt
from yahoo_oauth import OAuth2
import yahoo_fantasy_api as yfa

HERE = os.path.dirname(os.path.abspath(__file__))
LEAGUE_NAME = os.environ.get("LEAGUE_NAME", "WEAREFAMILY")
OUT_DIR = os.environ.get("OUT_DIR", os.path.join(HERE, "out"))
POSITIONS = ["QB", "RB", "WR", "TE", "K", "DEF"]
FA_PER_POS = 25

def main():
    sc = OAuth2(None, None, from_file=os.path.join(HERE, "oauth2.json"), browser_callback=False)
    if not sc.token_is_valid():
        sc.refresh_access_token()
    gm = yfa.Game(sc, "nfl")
    year = int(os.environ.get("SEASON", dt.date.today().year))

    league = None
    for lid in gm.league_ids(year=year):
        lg = gm.to_league(lid)
        if lg.settings().get("name", "").strip().upper() == LEAGUE_NAME.upper():
            league = lg
            break
    if league is None:
        sys.exit(f"League {LEAGUE_NAME!r} not found for {year}")

    week = league.current_week()
    my_key = league.team_key()
    teams = league.teams()

    def roster(team_key, wk):
        return league.to_team(team_key).roster(week=wk)

    # opponent this week, from the raw matchups payload
    opp_key = None
    try:
        raw = league.matchups(week=week)
        blob = json.dumps(raw)
        sb = raw["fantasy_content"]["league"][1]["scoreboard"]["0"]["matchups"]
        for k, m in sb.items():
            if k == "count":
                continue
            t = m["matchup"]["0"]["teams"]
            keys = [t["0"]["team"][0][0]["team_key"], t["1"]["team"][0][0]["team_key"]]
            if my_key in keys:
                opp_key = keys[1] if keys[0] == my_key else keys[0]
    except Exception as e:
        raw, opp_key = {"error": str(e)}, None

    free_agents = {}
    for pos in POSITIONS:
        try:
            free_agents[pos] = league.free_agents(pos)[:FA_PER_POS]
        except Exception as e:
            free_agents[pos] = {"error": str(e)}

    out = {
        "exported_at": dt.datetime.now(dt.timezone.utc).isoformat(),
        "league": league.settings(),
        "stat_categories": league.stat_categories(),
        "week": week,
        "my_team_key": my_key,
        "teams": {k: v.get("name") for k, v in teams.items()},
        "standings": league.standings(),
        "my_roster": roster(my_key, week),
        "opponent_key": opp_key,
        "opponent_roster": roster(opp_key, week) if opp_key else None,
        "matchups_raw": raw,
        "waivers": league.waivers(),
        "free_agents": free_agents,
    }

    os.makedirs(OUT_DIR, exist_ok=True)
    stamp = dt.datetime.now().strftime("%Y%m%d_%H%M")
    path = os.path.join(OUT_DIR, f"league_{stamp}.json")
    for p in (path, os.path.join(OUT_DIR, "latest.json")):
        with open(p, "w") as f:
            json.dump(out, f, indent=1, default=str)
    print(path)

if __name__ == "__main__":
    main()
