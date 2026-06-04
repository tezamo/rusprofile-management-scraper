<img src="head.gif" alt="head" width="800" height="200">

# RusProfile Management History Scraper

A Python scraper that extracts the **complete management history** of Russian companies from [rusprofile.ru](https://www.rusprofile.ru), using a list of company INNs as input.

---

## What It Does

For each company INN provided, the scraper:

1. Resolves the INN to a RusProfile internal company ID via their search
2. Fetches the full management change history from the RusProfile API
3. Falls back to the company summary page for companies with no recorded history (current manager only)
4. Outputs a clean CSV with one row per manager appointment

---

## Output Format

`management_history.csv`

| Column | Description |
|---|---|
| `inn` | Company INN |
| `company_name` | Company name from RusProfile |
| `manager_name` | Full name of the manager |
| `start_date` | Appointment date (YYYY-MM-DD) |
| `end_date` | End date (YYYY-MM-DD), empty if current |
| `manager_gender` | `Male`, `Female`, or `Unknown` — filled by `gender.py` |

A company will appear multiple times if it had multiple managers.

---

## Two-Script Pipeline

This project consists of two scripts that run sequentially:

```
inns.csv  →  [scraper.py]  →  management_history.csv  →  [gender.py]  →  management_history_with_gender.csv
```

---

## Script 1 — scraper.py

Fetches management history from RusProfile. See full details below.

---

## Script 2 — gender.py

Fills the `manager_gender` column using Russian morphological analysis and a foreign name dictionary.

### Requirements

```bash
pip install pymorphy3
```

### Usage

```bash
python gender.py
```

Set `INPUT_CSV` at the top of `gender.py` to point to the scraper output:

```python
INPUT_CSV  = "management_history.csv"
OUTPUT_CSV = "management_history_with_gender.csv"
```

### How Gender Detection Works

Detection runs through four strategies in order, stopping at the first confident result:

**Strategy 1 — Russian patronymics (highest confidence)**
Patronymic suffixes are unambiguous gender markers in Russian names:
- Male: `ович`, `евич`, `ьевич`, `ич`
- Female: `овна`, `евна`, `ьевна`, `на`

**Strategy 2 — Foreign name dictionary**
An explicit lookup table for international names that pymorphy3 cannot parse correctly (Iranian, Turkish, Chinese, European, etc.). Extend `FOREIGN_FIRST_NAMES` in `gender.py` to add more:

```python
FOREIGN_FIRST_NAMES = {
    "хассан": "Male",
    "мустафа": "Male",
    "таня": "Female",
    # add more as needed
}
```

**Strategy 3 — pymorphy3 morphological scoring**
Each word in the name is parsed for grammatical gender. Male and female scores are accumulated; the winning side by a margin of 2+ wins.

**Strategy 4 — Suffix heuristics**
Fallback patterns for remaining foreign names based on common Turkish, Iranian, and European name endings.

### Gender Coverage

| Result | Meaning |
|---|---|
| `Male` | Detected with confidence |
| `Female` | Detected with confidence |
| `Unknown` | Could not determine (typically rare foreign names) |

---

## Example Output

```csv
inn,company_name,manager_name,start_date,end_date,manager_gender
0216006326,ООО "Давлекановский КХП №1",Фаттахов Альберт Фанитович,,2017-01-23,
0216006326,ООО "Давлекановский КХП №1",Шершнев Александр Михайлович,2017-01-23,2018-10-08,
0216006326,ООО "Давлекановский КХП №1",Курганский Владимир Григорьевич,2018-10-08,2019-11-11,
0216006326,ООО "Давлекановский КХП №1",Ларионов Сергей Васильевич,2019-11-11,2020-05-15,
0216006326,ООО "Давлекановский КХП №1",Курганский Владимир Григорьевич,2020-05-15,2021-10-26,
0216006326,ООО "Давлекановский КХП №1",Матвеева Ирина Викторовна,2021-10-26,2023-05-22,
```

---

## Requirements

```
requests     # scraper.py
pymorphy3    # gender.py
```

Install with:

```bash
pip install requests
pip install requests pymorphy3
```

No browser or Selenium needed — this uses direct HTTP requests to the RusProfile API.

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/yourusername/rusprofile-management-scraper.git
cd rusprofile-management-scraper
```

### 2. Prepare your INN list

Create a file called `inns.csv` with one INN per line: 
``` 
0216006326  
0105082709  
0250004304
```

Supported encodings: UTF-8, UTF-16, CP1251. Both 10-digit (legal entity) and 12-digit (individual entrepreneur) INNs are accepted, though rusprofile only indexes legal entities.

### 3. Configure your session cookies

The scraper requires an active **paid RusProfile subscription** to access management history. You need to copy your session cookies from the browser.

**Steps:**
1. Log in to [rusprofile.ru](https://www.rusprofile.ru) in Chrome
2. Open DevTools → **Application** → **Cookies** → `www.rusprofile.ru`
3. Copy the values for the following cookies into the `COOKIES` dict at the top of `scraper.py`:

```python
COOKIES = {
    "sessid": "YOUR_SESSID",
    "__Host-csrf-token": "YOUR_CSRF_TOKEN",
    "email": "your_email%40example.com",
    "login": "...",
    "fbb_s": "1",
    "fbb_u": "...",
    "sp": "...",
}

CSRF = "YOUR_CSRF_TOKEN"  # same as __Host-csrf-token
```

> ⚠️ **Cookies expire** with your browser session. If you get empty results or CSRF errors, refresh the cookies from DevTools and update the script.

### 4. Run

```bash
python scraper.py
```

---

## How It Works

### Step 1 — Company ID resolution
RusProfile uses its own internal numeric IDs (e.g. `344568`), not INNs or OGRNs, in its API. The scraper searches by INN and extracts the correct ID from the search result card (`<a class="list-element__title">`), then verifies it by confirming the INN appears on the company page.

### Step 2 — Management history API
The scraper calls:
POST https://www.rusprofile.ru/ajax_auth.php
?action=history_finder&id={rp_id}&origin=egrul&group=ceo&task=list

This returns a JSON structure of dated change events. Each event is either:
- **"Новое лицо"** → manager appointment (start date)
- **"больше не является"** → manager departure (end date)
- **"должность изменена"** → position title change

### Step 3 — Smart name matching
The same person can appear with different name spellings across events (e.g. a typo in one EGRUL filing). The scraper uses the **person URL slug** (e.g. `/person/tagirzyanova-iv-164508469150`) as the unique identifier, not the display name. This means two different spellings of the same person are correctly merged into one record, with the correct start and end dates.

### Step 4 — Repeated tenures
A person who served, left, and returned (e.g. Курганский Владимир Григорьевич in the example above) gets **one row per tenure**, matched by finding the chronologically next end-date after each start-date for that person slug.

### Step 5 — Fallback for companies with no history
If a company has no recorded management change events (common for older or simpler companies), the scraper falls back to the company summary page and extracts the current manager's name, role, and start date from the `Руководитель` section.

---

## Configuration

At the top of `scraper.py`:

| Variable | Default | Description |
|---|---|---|
| `INN_FILE` | `inns.csv` | Path to input INN list |
| `OUTPUT_CSV` | `management_history.csv` | Output file path |
| `ERROR_LOG` | `rusprofile_errors.log` | Log for failed INNs |
| `DELAY` | `2` | Seconds between requests |

---

## Output Files

| File | Description |
|---|---|
| `management_history.csv` | Main output — all manager records |
| `rusprofile_errors.log` | INNs that failed with reason and timestamp |

---

## Known Limitations

- **Requires a paid RusProfile subscription** — the management history API is behind a paywall
- **Session cookies expire** — typically last a few days; must be refreshed manually
- **History depth** — RusProfile only shows changes recorded in EGRUL; appointments before their tracking began appear as end-only records (no start date)
- **Gender column** — not provided by the RusProfile API; left empty
- **Foreign companies / IPs** — 12-digit individual entrepreneur INNs and foreign entity identifiers will not be found on RusProfile
- **Rate limiting** — a 2-second delay between requests is enforced to avoid being blocked

---

## Notes on Data Quality

- `start_date` is empty for managers who were already in post before RusProfile began tracking changes for that company
- `end_date` is empty for the current active manager
- When the same person served multiple terms, each term appears as a separate row
- Company names are taken from RusProfile and may contain HTML entities (e.g. `&quot;`) — clean with `html.unescape()` if needed

---

## License

MIT