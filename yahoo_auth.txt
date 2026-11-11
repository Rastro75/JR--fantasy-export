#!/usr/bin/env python3
"""One-time Yahoo sign-in that shows Yahoo's own error if it fails.
Reads oauth2.json, prints the approve link, takes the code, saves the token
into oauth2.json. Never prints your secret or token."""
import json, os, sys, time, base64, urllib.parse, urllib.request, urllib.error
HERE = os.path.dirname(os.path.abspath(__file__))
P = os.path.join(HERE, "oauth2.json")
cfg = json.load(open(P))
cid, sec = cfg["consumer_key"].strip(), cfg["consumer_secret"].strip()
cb = cfg.get("callback_uri", "https://localhost:8000").strip()
print("Client ID length:", len(cid), " Secret length:", len(sec), " Redirect:", cb)
url = "https://api.login.yahoo.com/oauth2/request_auth?" + urllib.parse.urlencode(
    {"client_id": cid, "redirect_uri": cb, "response_type": "code", "scope": "fspt-r"})
print("\nOpen this on your PC and approve:\n" + url + "\n")
raw = input("Paste the code (or the whole localhost address): ").strip()
if "code=" in raw:
    raw = urllib.parse.parse_qs(urllib.parse.urlparse(raw).query).get("code", [raw])[0]
body = urllib.parse.urlencode({"grant_type": "authorization_code", "redirect_uri": cb, "code": raw}).encode()
auth = base64.b64encode(f"{cid}:{sec}".encode()).decode()
req = urllib.request.Request("https://api.login.yahoo.com/oauth2/get_token", data=body,
    headers={"Authorization": "Basic " + auth, "Content-Type": "application/x-www-form-urlencoded"})
try:
    tok = json.load(urllib.request.urlopen(req))
except urllib.error.HTTPError as e:
    txt = e.read().decode(errors="replace")
    try:
        j = json.loads(txt); print("Yahoo said:", e.code, j.get("error"), "-", j.get("error_description"))
    except Exception:
        print("Yahoo said:", e.code, txt[:300])
    sys.exit(1)
cfg.update({"access_token": tok["access_token"], "refresh_token": tok["refresh_token"],
            "token_type": tok.get("token_type", "bearer"), "token_time": time.time(),
            "guid": tok.get("xoauth_yahoo_guid")})
json.dump(cfg, open(P, "w"), indent=1); os.chmod(P, 0o600)
print("Signed in. Token saved to oauth2.json. Now run: venv/bin/python yahoo_export.py")
