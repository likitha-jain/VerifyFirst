# ScamMap text model integration

Use the **CSV** for Python training; keep the XLSX for manual review. Both original dataset files are copied unchanged into `data/ml/`.

## Train (PowerShell)
```powershell
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python scripts\train_message_model.py
```

The script reads all 50 rows and creates `app/models/message_classifier.joblib` plus `training_metrics.json`. It does not edit the source dataset.

## Run the existing FastAPI app
```powershell
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 1
```

POST JSON to `http://127.0.0.1:8000/ml/analyze-message`: `{"message_text":"Everyone in our group says this number is a scam. I have no screenshot."}`. Swagger: `http://127.0.0.1:8000/docs`.

## What it predicts
- `SYNTHETIC_SCAM_MESSAGE`
- `COORDINATED_UNVERIFIED_REPORT`

**Safety/quality limit:** 50 synthetic examples are far too few to establish real-world accuracy. This endpoint is a demo screening signal only. It must not mark a phone number `KNOWN_SCAM`, raise a public alert, or alter trust scores. Keep evidence validation, reporter independence, duplicate/burst checks and human review authoritative.
