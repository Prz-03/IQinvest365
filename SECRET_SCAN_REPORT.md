Secret scan report — quick summary

Date: 2025-12-17

1) Removed file from working tree
- `iqinvest365-firebase-database.json` contained a Firebase service-account private key and was removed from the working tree. You must *rotate the key immediately* in Google Cloud Console.

2) Other sensitive findings (locations)
- `app.py` references environment variables that MUST be stored as secrets (no values found in repo): `MASTER_TRON_PRIVATE_KEY`, `TRONGRID_API_KEY`, `ENCRYPTION_KEY`, `BINANCE_API_KEY/BINANCE_API_SECRET`, `EMAIL_PASSWORD`.
- `.gitignore` already ignores `iqinvest365-firebase-database.json` and `tradelink-key.json`.

3) Recommended immediate actions
- Revoke the compromised Firebase key in Google Cloud IAM -> Service Accounts -> Keys.
- Create a new service-account key with minimal permissions; store it securely and set `FIREBASE_KEY` in Fly secrets (base64 or raw JSON).
- Purge the leaked file from the git history using `git filter-repo` or `BFG` (runbook below).

4) Runbook: purge file from git history (BFG example)
```powershell
# Make a backup of the repo first
git clone --mirror . repo.git-mirror
cd repo.git-mirror
# Use BFG (download bfg.jar first)
java -jar ..\bfg.jar --delete-files iqinvest365-firebase-database.json
git reflog expire --expire=now --all
git gc --prune=now --aggressive
# Push cleaned history to origin (force push) - coordinate with team
git push --force
```

Or using git-filter-repo (preferred):
```powershell
pip install git-filter-repo
# From a fresh clone
git clone --mirror https://github.com/your/repo.git
cd repo.git
git filter-repo --invert-paths --paths iqinvest365-firebase-database.json
# Force-push cleaned history to remote
git push --force
```

5) How to set new FIREBASE_KEY in Fly (PowerShell example)
```powershell
$b64 = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes((Get-Content C:\path\to\new-key.json -Raw)))
flyctl secrets set FIREBASE_KEY=$b64 FLASK_SECRET_KEY="<your-flask-secret>" ENCRYPTION_KEY="<fernet-key>" FIREBASE_DB_URL="https://<your-db>.realtime-database.app"
```

6) Notes
- After history purge, collaborators must re-clone the repo.
- Rotating the service-account key is urgent; do that before redeploying.
