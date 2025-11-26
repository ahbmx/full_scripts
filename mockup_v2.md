Perfect! Let’s put everything together into a **complete, fully working mock SAN data collection and reporting system**. This includes:

* **Family helper modules**
* **Parallel execution of functions per device**
* **Merged DataFrames**
* **Full reporting with tables + charts in a single PDF**

---

## **Directory structure**

```
v1/
├── v1.py
├── inventory.json
└── dvl/
    ├── __init__.py
    ├── brocadeHelper.py
    ├── powermaxHelper.py
    ├── purestorageHelper.py
    ├── netappHelper.py
    ├── ecsHelper.py
    ├── datadomainHelper.py
    └── reportHelper.py
```

---

## **inventory.json (mock)**

```json
{
    "brocade": {"devices": ["sw1","sw2"], "user":"admin","password":"pass"},
    "powermax": {"devices": ["pm1","pm2"], "user":"admin","password":"pass"},
    "purestorage": {"devices": ["pure1","pure2"], "user":"admin","password":"pass"},
    "netapp": {"devices": ["net1","net2"], "user":"admin","password":"pass"},
    "ecs": {"devices": ["ecs1","ecs2"], "user":"admin","password":"pass"},
    "datadomain": {"devices": ["dd1","dd2"], "user":"admin","password":"pass"}
}
```

---

## **dvl/brocadeHelper.py**

```python
import time, random
import pandas as pd

def get_switchshow(device, credentials):
    time.sleep(random.uniform(0.1, 0.3))
    return pd.DataFrame({"port": ["1","2","3"], "status": ["up","down","up"], "device": device})

def get_portshow(device, credentials):
    time.sleep(random.uniform(0.1, 0.3))
    return pd.DataFrame({"port": ["A","B"], "speed": ["16G","32G"], "device": device})
```

---

## **dvl/powermaxHelper.py**

```python
import time, random
import pandas as pd

def get_arrayinfo(device, credentials):
    time.sleep(random.uniform(0.1, 0.2))
    return pd.DataFrame({"volume": ["vol1","vol2"], "size_gb": [100,200], "device": device})
```

---

## **dvl/purestorageHelper.py**

```python
import time, random
import pandas as pd

def get_volumes(device, credentials):
    time.sleep(random.uniform(0.1, 0.2))
    return pd.DataFrame({"volume": ["volA","volB"], "capacity_gb": [50,75], "device": device})
```

---

## **dvl/netappHelper.py**

```python
import time, random
import pandas as pd

def get_volumes(device, credentials):
    time.sleep(random.uniform(0.1, 0.2))
    return pd.DataFrame({"volume": ["volX","volY"], "size_gb": [120,80], "device": device})
```

---

## **dvl/ecsHelper.py**

```python
import time, random
import pandas as pd

def get_bucket_info(device, credentials):
    time.sleep(random.uniform(0.1, 0.2))
    return pd.DataFrame({"bucket": ["bucket1","bucket2"], "objects": [1000,500], "device": device})
```

---

## **dvl/datadomainHelper.py**

```python
import time, random
import pandas as pd

def get_filesystem(device, credentials):
    time.sleep(random.uniform(0.1, 0.2))
    return pd.DataFrame({"filesystem": ["fs1","fs2"], "size_tb": [5,2], "device": device})
```

---

## **dvl/reportHelper.py**

```python
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.backends.backend_pdf import PdfPages

def df_to_table(ax, df, title=""):
    ax.axis('off')
    ax.set_title(title, fontsize=14, fontweight='bold')
    table = ax.table(cellText=df.values,
                     colLabels=df.columns,
                     cellLoc='center',
                     loc='center')
    table.auto_set_font_size(False)
    table.set_fontsize(10)
    table.auto_set_column_width(col=list(range(len(df.columns))))

def generate_brocade_report(switch_df, port_df, pdf=None):
    if switch_df is not None:
        print("\n=== Brocade Switch Status ===")
        print(switch_df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, switch_df, "Brocade Switch Status")
            pdf.savefig(fig)
            plt.close(fig)
            status_counts = switch_df.groupby('status')['device'].count()
            fig, ax = plt.subplots()
            status_counts.plot(kind='bar', ax=ax, title='Brocade Switch Status', ylabel='Number of ports')
            pdf.savefig(fig)
            plt.close(fig)

    if port_df is not None:
        print("\n=== Brocade Port Speeds ===")
        print(port_df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, port_df, "Brocade Port Speeds")
            pdf.savefig(fig)
            plt.close(fig)
            speed_counts = port_df.groupby('speed')['device'].count()
            fig, ax = plt.subplots()
            speed_counts.plot(kind='bar', ax=ax, title='Brocade Port Speeds', ylabel='Number of ports')
            pdf.savefig(fig)
            plt.close(fig)

def generate_powermax_report(array_df, pdf=None):
    if array_df is not None:
        print("\n=== PowerMax Report ===")
        print(array_df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, array_df, "PowerMax Array Info")
            pdf.savefig(fig)
            plt.close(fig)
            fig, ax = plt.subplots()
            array_df.groupby('device')['size_gb'].sum().plot(kind='bar', ax=ax, title='PowerMax Array Sizes (GB)')
            pdf.savefig(fig)
            plt.close(fig)

def generate_purestorage_report(df, pdf=None):
    if df is not None:
        print("\n=== PureStorage Report ===")
        print(df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, df, "PureStorage Volumes")
            pdf.savefig(fig)
            plt.close(fig)
            fig, ax = plt.subplots()
            df.groupby('device')['capacity_gb'].sum().plot(kind='bar', ax=ax, title='PureStorage Capacities (GB)')
            pdf.savefig(fig)
            plt.close(fig)

def generate_netapp_report(df, pdf=None):
    if df is not None:
        print("\n=== NetApp Report ===")
        print(df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, df, "NetApp Volumes")
            pdf.savefig(fig)
            plt.close(fig)
            fig, ax = plt.subplots()
            df.groupby('device')['size_gb'].sum().plot(kind='bar', ax=ax, title='NetApp Volume Sizes (GB)')
            pdf.savefig(fig)
            plt.close(fig)

def generate_ecs_report(df, pdf=None):
    if df is not None:
        print("\n=== ECS Report ===")
        print(df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, df, "ECS Buckets")
            pdf.savefig(fig)
            plt.close(fig)
            fig, ax = plt.subplots()
            df.groupby('device')['objects'].sum().plot(kind='bar', ax=ax, title='ECS Bucket Objects')
            pdf.savefig(fig)
            plt.close(fig)

def generate_datadomain_report(df, pdf=None):
    if df is not None:
        print("\n=== DataDomain Report ===")
        print(df)
        if pdf:
            fig, ax = plt.subplots(figsize=(8,2))
            df_to_table(ax, df, "DataDomain Filesystems")
            pdf.savefig(fig)
            plt.close(fig)
            fig, ax = plt.subplots()
            df.groupby('device')['size_tb'].sum().plot(kind='bar', ax=ax, title='DataDomain Filesystem Sizes (TB)')
            pdf.savefig(fig)
            plt.close(fig)

def generate_all_reports(results, pdf_filename=None):
    pdf = PdfPages(pdf_filename) if pdf_filename else None

    generate_brocade_report(results.get('brocade_switchshow'), results.get('brocade_portshow'), pdf)
    generate_powermax_report(results.get('powermax_arrayinfo'), pdf)
    generate_purestorage_report(results.get('purestorage_volumes'), pdf)
    generate_netapp_report(results.get('netapp_volumes'), pdf)
    generate_ecs_report(results.get('ecs_bucket_info'), pdf)
    generate_datadomain_report(results.get('datadomain_filesystem'), pdf)

    if pdf:
        pdf.close()
        print(f"\n[INFO] PDF report saved to {pdf_filename}")
```

---

## **v1.py (Main script)**

```python
import json
import pandas as pd
from concurrent.futures import ThreadPoolExecutor, as_completed

from dvl import brocadeHelper, powermaxHelper, purestorageHelper
from dvl import netappHelper, ecsHelper, datadomainHelper
from dvl import reportHelper

# ---------- Load inventory ----------
with open("inventory.json") as f:
    inventory = json.load(f)

# ---------- Family Functions ----------
FAMILY_FUNCTIONS = {
    "brocade": {"switchshow": brocadeHelper.get_switchshow, "portshow": brocadeHelper.get_portshow},
    "powermax": {"arrayinfo": powermaxHelper.get_arrayinfo},
    "purestorage": {"volumes": purestorageHelper.get_volumes},
    "netapp": {"volumes": netappHelper.get_volumes},
    "ecs": {"bucket_info": ecsHelper.get_bucket_info},
    "datadomain": {"filesystem": datadomainHelper.get_filesystem}
}

# ---------- UTILITY FUNCTIONS ----------
def run_functions_on_device(device, functions, credentials):
    results = {}
    for fn_name, func in functions.items():
        try:
            df = func(device, credentials)
            results[fn_name] = df
        except Exception as e:
            results[fn_name] = pd.DataFrame({"device":[device], "error":[str(e)]})
    return results

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

def run_all_families(inventory, family_functions):
    nested_results = {}
    with ThreadPoolExecutor() as executor:
        future_to_family = {
            executor.submit(
                run_family,
                family,
                inventory[family]["devices"],
                family_functions.get(family, {}),
                {"user": inventory[family]["user"], "password": inventory[family]["password"]}
            ): family
            for family in inventory
        }
        for future in as_completed(future_to_family):
            family = future_to_family[future]
            nested_results[family] = future.result()

    flat_results = {}
    for family, funcs in nested_results.items():
        for fn_name, df in funcs.items():
            flat_results[f"{family}_{fn_name}"] = df
    return flat_results

def main():
    results = run_all_families(inventory, FAMILY_FUNCTIONS)

    # Example: generate PDF report with tables + charts
    reportHelper.generate_all_reports(results, pdf_filename='SAN_Report.pdf')

if __name__ == "__main__":
    main()
```

---

### ✅ Features of this full setup

1. **Parallel execution per device** and per family
2. **One DataFrame per function** merged across devices
3. **Device column added automatically**
4. **Tables + charts in a single PDF**
5. Fully **mocked** for testing without hardware
6. Easy to extend for real commands or REST API calls

---

This is **fully runnable as-is**. Running `python v1.py` will generate **`SAN_Report.pdf`** with **all tables and charts** for all families.

---

Do you want me to also **add a summary dashboard page at the start of the PDF** with totals per family? This is common in production SAN reports.
