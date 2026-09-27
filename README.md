# Stansted Airport Parking Market Audit

Automated price intelligence system for CSL Stansted Airport car parking. Scrapes competitor pricing across 256 search combinations daily, generates formatted Excel reports, and emails results to operations teams.

## Overview

This system monitors the parking market at Stansted Airport by:
- Running 256 automated searches across **16 check-in dates** (next 16 days) × **16 stay durations** (1–16 days)
- Extracting prices from 19+ parking operators (CSL, Peak Park, SMG, Maple, JetParks, etc.)
- Generating a formatted Excel workbook with two sheets:
  - **Pricing Matrix**: Raw pricing data with colour-coded suppliers
  - **Competitive Analysis**: Summary statistics and % gap vs CSL pricing
- Emailing the report via SMTP to operations at 06:00 and 14:00 UTC daily
- Logging all activity for audit trail and troubleshooting

**Typical runtime**: 25–30 minutes for full 256-search audit

## Current Status

✅ **Working**
- Web scraping and price extraction (18+ suppliers captured)
- Excel report generation with native tables and conditional formatting
- Cron scheduling (06:00 and 14:00 UTC daily)
- Startup logging for cron visibility

🔧 **In Progress**
- Email delivery via Krystal self-hosted SMTP (markets@markclulow.com)
- SMTP credentials configured; testing integration

## Architecture

```
parking_audit_email.py
├── Dependencies
│   ├── Playwright (headless Chromium)
│   ├── openpyxl (Excel generation)
│   └── smtplib (SMTP email)
├── Core functions
│   ├── run_all_searches() – 256 Looking4.com searches
│   ├── extract_prices_from_page() – Supplier price extraction
│   ├── create_spreadsheet() – Excel report generation
│   └── send_email_smtp() – SMTP email delivery
└── Logging
    ├── audit_YYYYMMDD_HHMMSS.log (execution details)
    ├── email_sends.log (delivery audit trail)
    └── cron_startup.log (cron visibility)
```

### Key Design Decisions

- **Playwright + headless Chromium**: Handles async JavaScript rendering on Looking4.com
- **domcontentloaded + wait_for_function**: Waits for actual supplier data (not just DOM ready)
- **Text-based extraction**: Regex price matching (`£\d+(?:\.\d{2})?`) against page text
- **Supplier normalisation**: Handles "and" vs "&" mismatches in names
- **Retry logic**: 1s then 3s backoff on extraction timeouts
- **SMTP over Gmail**: Removed Gmail app password auth for production reliability
- **Safety thresholds**: Aborts email if >10% empty-price rows or CSL missing

## Setup

### Prerequisites

- Python 3.10+
- VPS or server with cron support
- Krystal (or compatible SMTP) email account

### Installation

```bash
git clone https://github.com/yourusername/parking-audit.git
cd parking-audit
python3 -m venv venv
source venv/bin/activate
pip install playwright openpyxl
python3 -m playwright install chromium
```

### Configuration

Set environment variables in `~/.parking_audit_env`:

```bash
export PARKING_AUDIT_OUTPUT="/home/ccadmin/parking-audit/reports"
export PARKING_AUDIT_SMTP_HOST="dualla-lon.krystal.uk"
export PARKING_AUDIT_SMTP_PORT="465"
export PARKING_AUDIT_SMTP_USER="markets@markclulow.com"
export PARKING_AUDIT_SMTP_PASSWORD="your-password-here"
```

Then load before running:
```bash
source ~/.parking_audit_env
```

### Cron Setup

Add to `crontab -e`:

```bash
0 6 * * * source /home/ccadmin/.parking_audit_env && cd /home/ccadmin/parking-audit && python3 parking_audit_email.py >> /home/ccadmin/parking-audit/cron_execution.log 2>&1
0 14 * * * source /home/ccadmin/.parking_audit_env && cd /home/ccadmin/parking-audit && python3 parking_audit_email.py >> /home/ccadmin/parking-audit/cron_execution.log 2>&1
```

## Usage

### Manual Full Audit

```bash
source ~/.parking_audit_env
python3 parking_audit_email.py
```

Output: `reports/Stansted_Parking_Audit_YYYYMMDD_HHMMSS.xlsx`

### Dry-Run Test (Single URL)

```bash
python3 parking_audit_email.py --test-url "https://booking.parking.looking4.com/search/?entryDate=2026-09-27&entryTime=10%3A00&exitDate=2026-09-28&exitTime=17%3A00&terminal=dontKnow&airport=STN&groups=%7B%22adult%22%3A1%7D"
```

Exits `0` if CSL and MAG products found; `1` if data incomplete.

### Check Logs

```bash
# Startup visibility (cron firing)
tail -20 reports/parking_audit_logs/cron_startup.log

# Latest audit execution
tail -50 reports/parking_audit_logs/audit_*.log | head -100

# Email delivery trail
cat reports/parking_audit_logs/email_sends.log
```

## Output

### Excel Spreadsheet

**Pricing Matrix sheet**
- Columns: Check-in date, Check-out date, Total days (inclusive), then one column per supplier
- Colour-coded supplier headers (CSL purple, MAG blue variants, competitors in brand colours)
- Red cells: Competitor prices undercut CSL (data quality highlight)
- Yellow cells: No prices extracted for that row (timeout/error)
- Native Excel table with sorting/filtering enabled

**Competitive Analysis sheet**
- Supplier | Avg Price (1-day) | Avg Price (7-day) | Avg Price (14-day) | % vs CSL
- CSL highlighted in purple
- Red cells: Competitors priced below CSL
- Useful for quick competitive positioning assessment

### Log Files

| File | Purpose |
|------|---------|
| `audit_YYYYMMDD_HHMMSS.log` | Full execution transcript (progress, errors, timing) |
| `email_sends.log` | SMTP delivery attempts (one line per run) |
| `cron_startup.log` | Cron visibility: PID, timestamp, SMTP config check |

## Suppliers Tracked

**CSL (in-house)**
- CSL Park & Ride
- CSL Meet & Greet

**MAG (owned/operated)**
- Short Stay Blue Zone
- Mid Stay
- Long Stay
- Short Stay Premium (Red, Yellow, Orange zones)
- Meet & Greet - onsite

**Competitors**
- SMG Parking - Park & Ride - Uncovered
- Peak Park & Ride
- Stansted Easy Meet & Greet (Full Flex, Wash & Wheels, EV Charging)
- Maple Parking - Park & Ride
- JetParks - Park & Ride
- I Love Park & Ride / Meet & Greet
- Bubble Park & Deliver - Return Meet - Non-flex

## Known Issues & Limitations

- **Async rendering**: Some searches occasionally timeout if Looking4.com's supplier service is slow; retry logic helps but not foolproof
- **Email setup**: Currently migrating from Gmail app password to self-hosted SMTP; email delivery being verified
- **Price extraction**: Regex-based matching; occasional false positives if supplier names appear in page text outside results (rare)
- **Browser crashes**: Playwright occasionally hangs on page load; script includes 60s timeout and graceful error handling

## Roadmap

**Phase 2: Data Analysis**
- Calculate % vs CSL for all suppliers across all date/duration combinations
- Identify pricing trends by check-in date and stay duration
- Flag market anomalies (underpricing, seasonal spikes)
- Export summary CSV for BI tools

**Phase 3: Additional Data Sources**
- Compare Park and Travel Supermarket
- Scrape Holiday Extras
- Integrate MAG API (if available) for owned-property pricing validation

**Phase 4: Reporting & Alerts**
- Dashboard showing pricing trends and competitor positioning
- Alert threshold: notify if competitor undercuts CSL by >X%
- Monthly summary report for pricing strategy review

**Future: Eliminate Web Scraping**
- Migrate to MAG API for authoritative internal pricing
- Use purchased API feeds for competitor benchmarking (when cost-justified)

## Troubleshooting

### Script fails to start in cron
Check `cron_startup.log`:
```bash
tail -5 reports/parking_audit_logs/cron_startup.log
```
If nothing appears, cron is not firing. Verify crontab is installed and systemd cron service is running:
```bash
systemctl status cron
crontab -l
```

### No prices extracted (yellow rows everywhere)
Run `--test-url` with a current date:
```bash
python3 parking_audit_email.py --test-url "https://..."
```
If extraction fails, Looking4.com page structure may have changed. Check `diagnose_looking4.py` (in `/debug/` folder) for page layout inspection.

### SMTP email not sending
Check `email_sends.log`:
```bash
grep FAILED reports/parking_audit_logs/email_sends.log
```
Verify credentials in `~/.parking_audit_env`:
```bash
grep PARKING_AUDIT_SMTP ~/.parking_audit_env
```
Test SMTP manually:
```bash
python3 test_smtp.py
```

### Empty prices after successful extraction
Verify supplier names in `EXPECTED_SUPPLIERS` list match Looking4.com's actual text (handles "and" vs "&" normalisation, but names must be present). Review latest audit log:
```bash
grep "FOUND\|MISSING" reports/parking_audit_logs/audit_*.log | tail -20
```

## Development

Edit `CONFIG` dict at top of `parking_audit_email.py` to adjust:
- Check-in dates range: `check_in_days_ahead` (default 16)
- Stay durations: `stay_durations` (default 16)
- Timeouts: `page_timeout`, `wait_after_load`
- Safety thresholds: `max_empty_rate`
- SMTP details (or use environment variables)

## License

Proprietary – CSL Stansted Airport

## Contact

Mark Clulow  
+44 7716 172 372  
www.MarkClulow.com
