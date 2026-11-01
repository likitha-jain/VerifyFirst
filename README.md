# ScamMap — Community Scam Map for India

ScamMap is a FastAPI-based community reporting and verification prototype. It supports reports about suspicious phone numbers, UPI IDs, URLs, sender IDs and scam SMS messages. The frontend includes reporting, lookup, verification status, alerts and a dark interactive map.

This guide is written for Windows 10/11 + VS Code + PowerShell. Follow the steps in order.

---

## 1. What you need

Install these first:

1. **Python 3.11 or 3.12** (64-bit). During installation, tick **Add Python to PATH**.
2. **Visual Studio Code**.
3. **Git** (optional, useful for version control).
4. Internet access for installing Python packages and loading the Leaflet libraries/map tiles in the browser.

Check Python in a new VS Code terminal:

```powershell
py --version
python --version
```

If `py` is not recognised, reinstall Python and enable the PATH option.

## 2. Extract and open the project

1. Extract the ZIP file.
2. Open VS Code.
3. Select **File → Open Folder**.
4. Choose the extracted **ScamMap** folder — the folder that contains `app`, `scripts`, `tests` and `requirements.txt`.
5. In VS Code select **Terminal → New Terminal**.

All commands below must be run from this `ScamMap` project folder. Confirm with:

```powershell
Get-ChildItem
```

You should see `app`, `scripts`, `tests`, `requirements.txt` and `.env.example`.

## 3. Create a Python virtual environment

Run:

```powershell
py -3 -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation, run this command for the current terminal only, then activate again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

You should now see `(.venv)` at the start of the terminal prompt.

## 4. Install the Python packages

Upgrade pip and install the project requirements:

```powershell
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Wait until installation finishes. If installation fails, read the final error line; do not install random packages one by one before checking the error.

## 5. Create your environment file

Create `.env` from the example:

```powershell
Copy-Item .env.example .env
```

Open `.env` in VS Code. It stores server-side settings. Do not commit this file to GitHub.

At minimum, configure:

```env
SCAMMAP_ADMIN_TOKEN=PUT_A_LONG_RANDOM_SECRET_HERE
CORS_ORIGINS=http://localhost:8000,http://127.0.0.1:8000
DEMO_MODE=1
ALLOW_DEMO_LICENSE_ISSUE=0
TRUST_PROXY_HEADERS=0
SCAMMAP_DATA_DIR=./data
SCAMMAP_LICENSE_FILE=./data/license.json
VT_API_KEY=
JUDGE_API_COMPAT=0
```

### Generate a strong admin token

This token protects administrative actions. It is **not** the licence private key. Generate a random value in PowerShell:

```powershell
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$adminToken = [Convert]::ToBase64String($bytes)
$adminToken
```

Copy the printed value into `.env` after `SCAMMAP_ADMIN_TOKEN=`. Keep it private. Restart the server after changing `.env`.

## 6. Generate a new random licence private/public key pair (Ed25519)

ScamMap uses an Ed25519 digital signature for licence validation. The **private key signs** a licence; the **public key verifies** it. The private key must never be placed in the public app folder, GitHub, frontend JavaScript, or a public hosting service.

> Important: Generating a new key pair changes which licences the app trusts. The bundled demo licence was signed using the prototype's original key. If you generate your own key pair, you must issue a new licence with that private key and configure the matching public key as shown below.

From the ScamMap project root, with `.venv` activated and dependencies installed, run:

```powershell
python scripts/generate_license_keys.py
```

The script creates:

- `secrets/license_private_key.pem` — **secret signing key**. Keep a protected offline backup. Do not share it.
- `secrets/license_public_key.txt` — public verification key. It is safe to configure on the ScamMap server.

The script also prints a `SCAMMAP_PUBLIC_KEY=...` line. Copy that complete line into `.env` (replace any old `SCAMMAP_PUBLIC_KEY` value; add it if it is not present).

Example format only — use the actual value printed by your script:

```env
SCAMMAP_PUBLIC_KEY=PASTE_THE_GENERATED_BASE64_PUBLIC_KEY_HERE
```

The key-generation script deliberately refuses to overwrite an existing key. If you already generated keys, keep using that pair. Losing the private key means you cannot issue further licences that match that public key. If you intentionally need a replacement, back up the old key and coordinate a public-key change plus a newly signed licence.

### Issue a signed licence using your new private key

After generating the key pair and adding the matching public key to `.env`, issue a licence. For a 30-day hackathon/demo licence:

```powershell
python scripts/issue_license.py --licensee "ScamMap Hackathon Demo" --days 30 --output data/license.json
```

You can change `--licensee` and `--days`. The resulting `data/license.json` is signed using `secrets/license_private_key.pem`. Keep the private key safe; only the signed JSON licence needs to be copied to the app server.

For a local demo using the bundled prototype licence instead, do not generate a new key pair. Keep the default public key and set `DEMO_MODE=1`. Do not mix a newly generated public key with the old bundled demo licence — signature validation will fail.

### Licence settings by environment

- **Local demo:** `DEMO_MODE=1`, `ALLOW_DEMO_LICENSE_ISSUE=0`. This lets the app install its bundled demo licence if no licence file exists.
- **Your own signed licence:** set `DEMO_MODE=0`, `ALLOW_DEMO_LICENSE_ISSUE=0`, configure `SCAMMAP_PUBLIC_KEY` and make sure `SCAMMAP_LICENSE_FILE` points to the signed JSON file.
- **Public deployment:** never enable demo licence issuing. Keep the private key offline/on a separate issuer machine. The app server needs only the public key and signed licence.

The private key generated here is a licence-signing key, not a TLS/HTTPS key and not an API key.

## 7. Optional: connect VirusTotal

VirusTotal adds URL reputation information to URL lookups. It is optional; ScamMap's core reporting and lookup work without it.

1. Obtain an API key from your VirusTotal account.
2. Open `.env`.
3. Set:

```env
VT_API_KEY=PASTE_YOUR_VIRUSTOTAL_API_KEY_HERE
```

4. Save `.env` and restart ScamMap.
5. Open the **Check** screen and test a URL.

Keep this key in `.env` only. Never paste it into `app/static`, browser JavaScript, HTML, screenshots or a public repository. VirusTotal may receive URLs submitted for analysis; do not submit confidential/private URLs. A clean or undetected result does not prove a URL is safe.

## 8. Optional: enable SMS screenshot OCR

Screenshot evidence checking uses Python packages plus the native Tesseract OCR application. On Windows, install a supported Tesseract distribution, ensure `tesseract.exe` is on your system `PATH`, and restart VS Code/PowerShell.

If using Windows Package Manager:

```powershell
winget install --id UB-Mannheim.TesseractOCR -e
```

If Tesseract is unavailable, screenshot evidence may be marked **INCONCLUSIVE**; it will not be treated as a successful evidence match. OCR is a screening aid, not proof that an image is genuine.

## 9. Optional: train the added report-matching model

The project includes a small synthetic dataset for an advisory screenshot/report matching feature. Train it from the project root:

```powershell
python scripts/train_report_match_model.py
```

The generated model is stored under `app/models/`. The dataset is synthetic and small; it demonstrates the integration flow and must not be described as a production-accurate fraud detector.

To train the separate message classifier, if the corresponding dataset is available in your project, run:

```powershell
python scripts/train_message_model.py
```

## 10. Start the ScamMap server

For local use, run one Uvicorn worker:

```powershell
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 1
```

Keep this terminal open while using the application.

Open the frontend in your browser:

```text
http://127.0.0.1:8000
```

Useful API documentation:

```text
http://127.0.0.1:8000/docs
```

Health check:

```text
http://127.0.0.1:8000/health
```

Stop the server with **Ctrl + C** in the terminal.

## 11. Use the dark interactive map

1. Start the server using Step 10.
2. Open `http://127.0.0.1:8000`.
3. Open the **Map** section in the existing dashboard.
4. Choose **All India** to see the national overview.
5. Select a city to zoom into supported locality/area clusters.
6. Select a marker to view its locality, city, pincode, report count, scam category and severity.
7. Use the heat-layer control to turn the density overlay on or off.
8. Use the time filters and existing alerts/area list to explore report activity.

The map reads aggregated area-level data from FastAPI's existing `GET /map` endpoint. It uses supported locality coordinates, not the reporter's exact GPS location. Do not add phone numbers, UPI IDs, reporter identifiers, raw SMS content or exact reporter coordinates to map popups.

The dark basemap uses CARTO Dark Matter tiles through Leaflet. Add your CARTO basemap key to `.env` as `CARTO_API_KEY`; the app reads it from the server's `/public-config` endpoint and appends it to CARTO tile requests. CARTO keys used by browser tile requests are visible to visitors, so use a client/public key and restrict it to your hosted domain in CARTO. Do not reuse a secret server credential. The browser also needs internet access for CARTO tiles and Leaflet/CDN assets. Keep map attribution visible and follow CARTO's current plan and tile-usage terms.

### Configure CARTO and VirusTotal keys while hosting

1. Copy `.env.example` to `.env` if you have not already done so.
2. Paste your CARTO basemap key into `CARTO_API_KEY=`. This is a browser-facing tile key; restrict it to your deployed domain in CARTO.
3. Paste your VirusTotal API key into `VT_API_KEY=`. This key is used only by the FastAPI backend and is never sent to the browser.
4. Save `.env`, restart FastAPI, and reload the dashboard. The CARTO map key is optional during local development; without it, the app attempts the public tile URL and CARTO may reject the request depending on its current policy.
5. Never paste either key into `app.js`, `index.html`, GitHub, screenshots, or public logs. Add `.env` to `.gitignore` and rotate a key if it is exposed.

Example `.env` entries (replace the placeholders; do not include quotes unless required by your shell):

```env
CARTO_API_KEY=your_carto_basemap_key
VT_API_KEY=your_virustotal_api_key
```

VirusTotal remains server-side. CARTO's tile key must reach the browser for tile requests, so treat it as a public/client key and rely on provider-side domain restrictions rather than assuming it is secret.

## 12. Test the project

With the virtual environment activated, run:

```powershell
python -m pytest -q
```

A passing test suite checks the covered automated behaviours; it does not guarantee production security, official judging compliance or real-world fraud-detection accuracy.

## 13. Admin actions

Admin endpoints require the token configured in `SCAMMAP_ADMIN_TOKEN`. Example PowerShell request:

```powershell
$headers = @{ "X-Admin-Token" = "YOUR_ADMIN_TOKEN_FROM_ENV" }
Invoke-RestMethod -Method Get -Uri http://127.0.0.1:8000/admin/challenges -Headers $headers
```

Do not put the admin token into frontend code or browser local storage. Do not expose admin routes publicly without authentication and network restrictions.

## 14. Hackathon judging compatibility (isolated judge instance only)

Some judging harnesses simulate multiple reporters from one machine and may call `POST /admin/reset` without credentials. If the official harness explicitly requires this compatibility mode, set:

```env
JUDGE_API_COMPAT=1
```

Use it only on an isolated, temporary judging deployment restricted to the event network or an allowlisted proxy. In this mode, the reset endpoint can be unauthenticated and client-supplied reporter identity fields may be honoured. Anyone who can reach the endpoint may reset data or spoof identities. For normal use and public deployment, keep:

```env
JUDGE_API_COMPAT=0
```

## 15. Hosting / VS Code port forwarding

For a quick private demonstration:

1. Start ScamMap on port `8000`.
2. In VS Code, open the **Ports** panel.
3. Select **Forward a Port** and enter `8000`.
4. Set port visibility to **Private** unless you intentionally want public access.
5. Open the forwarded address shown by VS Code.

A forwarded port is a way to reach your running development server; it is not a production deployment. Before making the app public, use HTTPS, a proper deployment host/reverse proxy, strong secrets, restricted CORS origins, backups, monitoring, and a security review. Do not expose the admin token, licence private key, database, `.env` or evidence storage.

If the browser uses a different forwarded origin, add that exact trusted origin to `CORS_ORIGINS` only if the frontend actually makes cross-origin API calls. Do not use `*` as a shortcut.

## 16. Important project behaviour and privacy notes

- A submitted report is initially **REPORTED · MARKED FOR VERIFICATION**. This means it was received and queued for checks; it does not mean the report is proven true.
- Community reports, SMS similarity, OCR, screenshot metadata and external URL reputation are screening signals. They are not individually proof of fraud.
- The trust engine uses corroboration and credibility signals; high-impact accusations should be reviewed before definitive public claims.
- The app redacts common phone, UPI, URL, OTP/PIN and long-number patterns from stored message text. Redaction is not guaranteed to catch every secret; users should not submit passwords, OTPs, full account details or other confidential information.
- Anonymous reporting is pseudonymous, not an absolute guarantee of anonymity. Server/network logs, hosting provider records and abuse-prevention controls may still process technical metadata.
- The map displays aggregated supported areas. It should not display reporters' precise GPS coordinates or private report evidence.
- VirusTotal is optional and external. A URL may be shared with VirusTotal when submitted for analysis.
- Keep one Uvicorn worker for this prototype. Rate limits and some caches are process-local; multiple workers/replicas do not share them.

## 17. Common problems

### `py` or `python` is not recognised

Reinstall Python 3.11/3.12 and tick **Add Python to PATH**. Restart VS Code.

### `Activate.ps1 cannot be loaded`

Run this in the current PowerShell terminal:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

### `No module named fastapi` or another package

Check that `(.venv)` appears in the prompt, then run:

```powershell
python -m pip install -r requirements.txt
```

### Port 8000 is already in use

Stop the old server with Ctrl+C. If it is running in another terminal, close it. Or use another port:

```powershell
python -m uvicorn app.main:app --host 127.0.0.1 --port 8001 --workers 1
```

Then open `http://127.0.0.1:8001`.

### Map is blank or tiles do not load

Check internet access, browser developer console, and whether public tile/CDN requests are blocked by your network. Confirm that the browser can reach the tile provider. The backend can still run even if the external basemap is unavailable.

### Licence is invalid after generating a new key

The public key in `.env` must match the private key used to sign `data/license.json`. Generate a new licence with `scripts/issue_license.py`, verify `SCAMMAP_PUBLIC_KEY` is the matching value, then restart the server. Do not use the old bundled demo licence with a newly generated key.

### `LICENSE_MISSING` or `LICENSE_EXPIRED`

Check `SCAMMAP_LICENSE_FILE`, confirm the signed JSON file exists at that location, and check its expiry date. For a local prototype demo, use the original bundled public key with `DEMO_MODE=1`, or issue a new licence using your own matching key pair.

## 18. Important files and folders

```text
ScamMap/
├── app/
│   ├── main.py                 # FastAPI application and API routes
│   ├── db.py                   # SQLite persistence
│   ├── trust.py                # Trust and verification logic
│   ├── areas.py                # Supported locality/pincode coordinates
│   ├── license.py              # Ed25519 licence verification
│   └── static/                 # Existing frontend HTML, CSS, JavaScript
├── data/                       # Runtime database, licence and ML dataset
├── docs/                       # Project documentation and UI reference
├── keys/                       # Public-key reference material
├── licenses/                   # Bundled demo licences
├── scripts/
│   ├── generate_license_keys.py # Generate a fresh Ed25519 keypair
│   ├── issue_license.py         # Sign a licence offline
│   ├── train_message_model.py
│   └── train_report_match_model.py
├── tests/                      # Automated tests
├── .env.example                # Environment variable template
├── requirements.txt            # Python dependencies
└── README.md                   # This guide
```

---

**Recommended first run:** complete Steps 1–5, leave the bundled demo keypair unchanged, keep `DEMO_MODE=1`, start the server, and test the dashboard/map. Generate your own licence key pair only when you are ready to replace the prototype's bundled licence with your own signed licence.
