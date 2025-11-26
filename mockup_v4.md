Perfect! Let’s build a **complete testable SAN collector environment** including all your families, with mock functions, ephemeral DataFrames, logging, scheduler, and conditional reports. You’ll be able to run it immediately for testing.

---

## **Full Project Structure**

```
v1.py
inventory.json
dvl/
    __init__.py
    logHelper.py
    brocadeHelper.py
    powermaxHelper.py
    purestorageHelper.py
    netappHelper.py
    ecsHelper.py
    datadomainHelper.py
    reportHelper.py
```

---

## **inventory.json** (mock example)

```json
{
    "brocade": {
        "user": "admin",
        "password": "pass",
        "devices": ["sw1", "sw2", "sw3"]
    },
    "powermax": {
        "user": "admin",
        "password": "pass",
        "devices": ["pm1", "pm2"]
    },
    "purestorage": {
        "user": "admin",
        "password": "pass",
        "devices": ["ps1", "ps2"]
    },
    "netapp": {
        "user": "admin",
        "password": "pass",
        "devices": ["na1", "na2"]
    },
    "ecs": {
        "user": "admin",
        "password": "pass",
        "devices": ["ecs1", "ecs2"]
    },
    "datadomain": {
        "user": "admin",
        "password": "pass",
        "devices": ["dd1", "dd2"]
    }
}
```

---

## **dvl/logHelper.py**

```python
import logging

def setup_logger(name="SAN_Collector", log_file="san_collector.log"):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    if not logger.handlers:
        fh = logging.FileHandler(log_file)
        fh.setLevel(logging.INFO)
        ch = logging.StreamHandler()
        ch.setLevel(logging.INFO)
        formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
        fh.setFormatter(formatter)
        ch.setFormatter(formatter)
        logger.addHandler(fh)
        logger.addHandler(ch)
    return logger
```

---

## **dvl/brocadeHelper.py** (mock)

```python
import pandas as pd
import random, time

def get_switchshow(device, credentials):
    time.sleep(random.uniform(0.1,0.3))
    return pd.DataFrame([{"device":device,"status":"ok","uptime":random.randint(10,100)}])

def get_portshow(device, credentials):
    time.sleep(random.uniform(0.1,0.3))
    return pd.DataFrame([{"device":device,"port":i,"util":random.randint(0,100)} for i in range(1,5)])
```

---

## **dvl/powermaxHelper.py** (mock)

```python
import pandas as pd, time

def get_arrayinfo(device, credentials):
    time.sleep(0.2)
    return pd.DataFrame([{"device":device,"status":"healthy"}])
```

---

## **dvl/purestorageHelper.py** (mock)

```python
import pandas as pd, time

def get_volumes(device, credentials):
    time.sleep(0.2)
    return pd.DataFrame([{"device":device,"volume":f"vol{i}"} for i in range(1,3)])
```

---

## **dvl/netappHelper.py** (mock)

```python
import pandas as pd, time

def get_filesystems(device, credentials):
    time.sleep(0.2)
    return pd.DataFrame([{"device":device,"fs":f"fs{i}"} for i in range(1,4)])
```

---

## **dvl/ecsHelper.py** (mock)

```python
import pandas as pd, time

def get_bucket_info(device, credentials):
    time.sleep(0.2)
    return pd.DataFrame([{"device":device,"bucket":f"bucket{i}"} for i in range(1,3)])
```

---

## **dvl/datadomainHelper.py** (mock)

```python
import pandas as pd, time

def get_dd_info(device, credentials):
    time.sleep(0.2)
    return pd.DataFrame([{"device":device,"info":"ok"}])
```

---

## **dvl/reportHelper.py** (mock PDF report generator)

```python
def generate_all_reports(results, pdf_filename="report.pdf"):
    print(f"[REPORT] Generating PDF {pdf_filename} with {len(results)} DataFrames:")
    for name, df in results.items():
        print(f"  - {name}: {len(df)} rows")
```

---

## **v1.py** – Main script

```python
import json, time
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor, as_completed
from apscheduler.schedulers.background import BackgroundScheduler
import pandas as pd

from dvl import brocadeHelper, powermaxHelper, purestorageHelper, netappHelper, ecsHelper, datadomainHelper, reportHelper, logHelper

# ---------- Logger ----------
logger = logHelper.setup_logger()

# ---------- Load inventory ----------
with open("inventory.json") as f:
    inventory = json.load(f)
logger.info("Inventory loaded.")

# ---------- Family functions with schedule metadata ----------
FAMILY_FUNCTIONS = {
    "brocade": {
        "switchshow": {"func": brocadeHelper.get_switchshow, "interval_hours":1},
        "portshow": {"func": brocadeHelper.get_portshow, "daily":True}
    },
    "powermax": {
        "arrayinfo": {"func": powermaxHelper.get_arrayinfo, "interval_hours":1}
    },
    "purestorage": {
        "volumes": {"func": purestorageHelper.get_volumes, "daily":True}
    },
    "netapp": {
        "filesystems": {"func": netappHelper.get_filesystems, "weekly":True}
    },
    "ecs": {
        "bucket_info": {"func": ecsHelper.get_bucket_info, "weekly":True}
    },
    "datadomain": {
        "dd_info": {"func": datadomainHelper.get_dd_info, "interval_hours":1}
    }
}

# ---------- Run functions on a single device ----------
def run_functions_on_device(device, functions, credentials):
    results = {}
    for fn_name, meta in functions.items():
        try:
            df = meta["func"](device, credentials)
            results[fn_name] = df
        except Exception as e:
            results[fn_name] = pd.DataFrame([{"device":device,"error":str(e)}])
            logger.error(f"Error running {fn_name} on {device}: {e}")
    return results

# ---------- Run all functions for a family ----------
def run_family(family_name, devices, functions, credentials):
    merged_results = {fn: [] for fn in functions}
    with ThreadPoolExecutor(max_workers=len(devices)) as executor:
        future_to_device = {executor.submit(run_functions_on_device, d, functions, credentials): d for d in devices}
        for future in as_completed(future_to_device):
            device_results = future.result()
            for fn, df in device_results.items():
                merged_results[fn].append(df)
    for fn in merged_results:
        if merged_results[fn]:
            merged_results[fn] = pd.concat(merged_results[fn], ignore_index=True)
        else:
            merged_results[fn] = pd.DataFrame()
    return merged_results

# ---------- Run all families ----------
def run_all_families(selected_types=None):
    nested_results = {}
    families = selected_types if selected_types else inventory.keys()
    for family in families:
        functions = FAMILY_FUNCTIONS[family]
        nested_results[family] = run_family(
            family,
            inventory[family]["devices"],
            functions,
            {"user": inventory[family]["user"], "password": inventory[family]["password"]}
        )
    flat_results = {}
    for family, funcs in nested_results.items():
        for fn_name, df in funcs.items():
            key = f"{family}_{fn_name}"
            flat_results[key] = df
    return flat_results

# ---------- Scheduler tasks ----------
def refresh_hourly_tables():
    logger.info("Refreshing hourly tables...")
    results = run_all_families()
    hourly_results = {k:v for k,v in results.items() if any(meta.get("interval_hours") for fam, funcs in FAMILY_FUNCTIONS.items() for fn, meta in funcs.items() if f"{fam}_{fn}"==k)}
    for df in hourly_results.values():
        if not df.empty:
            # Insert/replace table in DB
            pass
    del results, hourly_results
    logger.info("Hourly tables refreshed.")

def run_daily_reports():
    logger.info("Running daily reports if data is available...")
    results = run_all_families()
    daily_results = {k:v for k,v in results.items() if any(meta.get("daily") for fam, funcs in FAMILY_FUNCTIONS.items() for fn, meta in funcs.items() if f"{fam}_{fn}"==k)}
    if any(not df.empty for df in daily_results.values()):
        reportHelper.generate_all_reports(daily_results, pdf_filename="SAN_Report_Daily.pdf")
        logger.info("Daily report generated.")
    else:
        logger.info("No new daily data. Skipping report.")
    del results, daily_results

def run_weekly_reports():
    logger.info("Running weekly reports if data is available...")
    results = run_all_families()
    weekly_results = {k:v for k,v in results.items() if any(meta.get("weekly") for fam, funcs in FAMILY_FUNCTIONS.items() for fn, meta in funcs.items() if f"{fam}_{fn}"==k)}
    if any(not df.empty for df in weekly_results.values()):
        reportHelper.generate_all_reports(weekly_results, pdf_filename="SAN_Report_Weekly.pdf")
        logger.info("Weekly report generated.")
    else:
        logger.info("No new weekly data. Skipping report.")
    del results, weekly_results

# ---------- Setup APScheduler ----------
scheduler = BackgroundScheduler()
scheduler.add_job(refresh_hourly_tables, 'interval', hours=1)
scheduler.add_job(run_daily_reports, 'cron', hour=1, minute=0)
scheduler.add_job(run_weekly_reports, 'cron', day_of_week='mon', hour=2, minute=0)
scheduler.start()

logger.info("Scheduler started. Press Ctrl+C to exit.")
try:
    while True:
        time.sleep(30)
except (KeyboardInterrupt, SystemExit):
    scheduler.shutdown()
    logger.info("Scheduler stopped.")
```

---

## **systemd Service File**

Create `/etc/systemd/system/san_collector.service`:

```ini
[Unit]
Description=SAN Data Collector Service
After=network.target

[Service]
User=yourusername
WorkingDirectory=/path/to/project
ExecStart=/usr/bin/python3 /path/to/project/v1.py
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

### Commands to enable service

```bash
sudo systemctl daemon-reload
sudo systemctl enable san_collector.service
sudo systemctl start san_collector.service
sudo systemctl status san_collector.service
```

---

### ✅ **Features**

1. Full mock environment for **brocade, powermax, purestorage, netapp, ecs, datadomain**.
2. **Ephemeral DataFrames**: automatically destroyed after each task.
3. **Reports** generated only if data exists.
4. **Logging** to console and file via `logHelper`.
5. **ThreadPoolExecutor**: parallel devices, one function per device at a time.
6. **APScheduler** handles hourly, daily, weekly jobs.
7. Runs persistently via **systemd** with auto-restart on failure/reboot.

---

This is a **complete, runnable test environment**.

I can also add **a small test script to simulate multiple runs and show reports generated conditionally**, if you want to verify behavior before deploying.

Do you want me to add that?
