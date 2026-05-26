# MCS Phase 0 — Environment Setup Agent Prompt (v2 Final)

## Context

You are setting up the development environment for MCS (Music Creator Studio), a closed-loop adaptive music generation control system built in Python 3.11. This is completely separate from any other system. The GitHub repository and all secrets have already been created manually. Your job is environment initialization only — no audio logic, no business logic, no module code.

**Stop on first failure. Print a clear diagnostic. Do not continue to the next task.**

---

## Preconditions (Check Before Executing Anything)

1. Confirm Python version is exactly 3.11:

```bash
python3.11 --version
```

If not available, print `[FAIL] Python 3.11 not found` and stop.

2. Confirm a `.env` file exists in the repo root. If it does not exist, print:

```
[FAIL] .env file not found. Create it from .env.template and populate all values before running.
```

Then stop. Do not auto-create it.

3. Confirm `.env` contains non-empty values for all 6 required keys:
   - `DRIVE_CREDENTIALS_1`, `DRIVE_CREDENTIALS_2`, `DRIVE_CREDENTIALS_3`
   - `DRIVE_FOLDER_ID_1`, `DRIVE_FOLDER_ID_2`, `DRIVE_FOLDER_ID_3`

If any are missing or empty, print which ones and stop.

---

## Task 1 — Local Folder Structure

Create the following structure in the repo root. Use `mkdir -p` — skip silently if folder already exists:

```
/backend
/frontend
/data
/data/sessions
/temp
/docs
```

Create these files only if they do not already exist:

- `/backend/__init__.py` — empty
- `/data/sessions/.gitkeep` — empty
- `/temp/.gitkeep` — empty
- `/docs/index.html`:

```html
<html><body><h1>MCS</h1></body></html>
```

- `.gitignore`:

```
temp/
*.pyc
__pycache__/
.env
credentials/
```

- `.env.template`:

```
# Full service account JSON (single-line string) for Google Drive account 1
DRIVE_CREDENTIALS_1=

# Full service account JSON (single-line string) for Google Drive account 2
DRIVE_CREDENTIALS_2=

# Full service account JSON (single-line string) for Google Drive account 3
DRIVE_CREDENTIALS_3=

# Google Drive folder ID for account 1
DRIVE_FOLDER_ID_1=

# Google Drive folder ID for account 2
DRIVE_FOLDER_ID_2=

# Google Drive folder ID for account 3
DRIVE_FOLDER_ID_3=
```

- `.python-version`:

```
3.11
```

---

## Task 2 — System Dependencies

Confirm `ffmpeg` is installed and available in PATH:

```bash
ffmpeg -version
```

If not found, print:

```
[FAIL] ffmpeg not found. Install with: sudo apt-get update && sudo apt-get install -y ffmpeg
```

Then stop.

---

## Task 3 — Python Dependencies

Create `requirements.txt` in the repo root with exactly these pinned versions:

```
librosa==0.10.2.post1
pydub==0.25.1
google-api-python-client==2.126.0
google-auth==2.29.0
google-auth-httplib2==0.2.0
python-dotenv==1.0.1
numpy==1.26.4
scipy==1.11.4
```

Upgrade pip before installing:

```bash
python3.11 -m pip install --upgrade pip
```

Then install:

```bash
pip install -r requirements.txt
```

Create `/backend/verify_install.py`:

```python
from pathlib import Path
ROOT = Path(__file__).resolve().parents[1]

packages = [
    "librosa",
    "pydub",
    "googleapiclient",
    "google.auth",
    "dotenv",
    "numpy",
    "scipy",
]

failed = False
for package in packages:
    try:
        __import__(package)
        print(f"[OK] {package}")
    except ImportError as e:
        print(f"[FAIL] {package} — {e}")
        failed = True

import sys
sys.exit(1 if failed else 0)
```

Run it:

```bash
python3.11 backend/verify_install.py
```

If exit code is non-zero, stop.

---

## Task 4 — Google Drive Authentication Test

Create `/backend/verify_drive.py`:

```python
import sys
import json
import os
from pathlib import Path
from dotenv import load_dotenv
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

ROOT = Path(__file__).resolve().parents[1]
load_dotenv(ROOT / ".env")

SCOPES = ["https://www.googleapis.com/auth/drive"]
failed = False

for i in range(1, 4):
    cred_raw = os.getenv(f"DRIVE_CREDENTIALS_{i}", "")
    folder_id = os.getenv(f"DRIVE_FOLDER_ID_{i}", "")

    if not cred_raw or not folder_id:
        print(f"[FAIL] Drive account {i} — credentials or folder ID missing")
        failed = True
        continue

    try:
        cred_dict = json.loads(cred_raw)
    except json.JSONDecodeError as e:
        print(f"[FAIL] Drive account {i} — malformed JSON. Ensure credentials are a single-line JSON string with no line breaks or unescaped quotes. Error: {e}")
        failed = True
        continue

    try:
        credentials = service_account.Credentials.from_service_account_info(
            cred_dict, scopes=SCOPES
        )
        service = build("drive", "v3", credentials=credentials)
        service.files().list(
            q=f"'{folder_id}' in parents",
            pageSize=1,
            fields="files(id, name)"
        ).execute()
        print(f"[OK] Drive account {i} authenticated — folder accessible")
    except HttpError as e:
        print(f"[FAIL] Drive account {i} — HTTP error: {e}")
        failed = True
    except Exception as e:
        print(f"[FAIL] Drive account {i} — {e}")
        failed = True

sys.exit(1 if failed else 0)
```

Run it:

```bash
python3.11 backend/verify_drive.py
```

If exit code is non-zero, stop.

---

## Task 5 — GitHub Actions Workflow Stub

Create `/.github/workflows/mcs_main.yml`:

```yaml
name: MCS Main

on:
  push:
    branches: [main]
  workflow_dispatch:

defaults:
  run:
    working-directory: .

jobs:
  environment-check:
    runs-on: ubuntu-latest

    env:
      DRIVE_CREDENTIALS_1: ${{ secrets.DRIVE_CREDENTIALS_1 }}
      DRIVE_CREDENTIALS_2: ${{ secrets.DRIVE_CREDENTIALS_2 }}
      DRIVE_CREDENTIALS_3: ${{ secrets.DRIVE_CREDENTIALS_3 }}
      DRIVE_FOLDER_ID_1: ${{ secrets.DRIVE_FOLDER_ID_1 }}
      DRIVE_FOLDER_ID_2: ${{ secrets.DRIVE_FOLDER_ID_2 }}
      DRIVE_FOLDER_ID_3: ${{ secrets.DRIVE_FOLDER_ID_3 }}

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install ffmpeg
        run: sudo apt-get update && sudo apt-get install -y ffmpeg

      - name: Upgrade pip
        run: python3.11 -m pip install --upgrade pip

      - name: Install Python dependencies
        run: pip install -r requirements.txt

      - name: Verify Python installs
        run: python backend/verify_install.py

      - name: Verify Drive authentication
        run: python backend/verify_drive.py
```

Each step fails the workflow automatically if the script exits non-zero — this is GitHub Actions default behavior and requires no extra configuration.

---

## Completion Gate

Phase 0 is complete when ALL of the following are true:

1. `verify_install.py` exits code 0 locally — all `[OK]`
2. `verify_drive.py` exits code 0 locally — all 3 accounts print `[OK]`
3. GitHub Actions workflow runs clean — both verify steps pass in CI
4. Folder structure matches Task 1 exactly
5. `.env` is not committed — confirm with `git status` before any push
6. `.env.template` and `.python-version` are committed and tracked

---

## What You Must Not Do

- Do not write any module logic, audio code, or business logic
- Do not install any packages not listed in Task 3
- Do not commit `.env` or any real credentials
- Do not modify GitHub Pages settings programmatically
- Do not create files outside the structure defined in Task 1
- Do not continue past any failed task — stop and print diagnostic