Perfect! Let’s create an updated **mock script** that:

1. Adds a **`device` column** to each DataFrame automatically.
2. Returns a **flat dictionary of merged DataFrames** so you can access them easily by name.
3. Keeps the **parallelization logic intact**.

---

### **v1_mock_flat.py**

```python
import time
import random
import pandas as pd
from concurrent.futures import ThreadPoolExecutor, as_completed

# ---------- MOCK HELPERS ----------
def mock_switchshow(device, credentials):
    time.sleep(random.uniform(0.1, 0.3))
    return pd.DataFrame({
        "port": ["1", "2", "3"],
        "status": ["up", "down", "up"]
    })

def mock_portshow(device, credentials):
    time.sleep(random.uniform(0.1, 0.3))
    return pd.DataFrame({
        "port": ["A", "B"],
        "speed": ["16G", "32G"]
    })

def mock_arrayinfo(device, credentials):
    time.sleep(random.uniform(0.1, 0.2))
    return pd.DataFrame({
        "volume": ["vol1", "vol2"],
        "size_gb": [100, 200]
    })

# ---------- MOCK INVENTORY ----------
inventory = {
    "brocade": {
        "devices": ["sw1", "sw2", "sw3"],
        "user": "admin",
        "password": "pass"
    },
    "powermax": {
        "devices": ["pm1", "pm2"],
        "user": "admin",
        "password": "pass"
    }
}

# Collector functions per family
FAMILY_FUNCTIONS = {
    "brocade": {
        "switchshow": mock_switchshow,
        "portshow": mock_portshow
    },
    "powermax": {
        "arrayinfo": mock_arrayinfo
    }
}

# ---------- UTILITY FUNCTIONS ----------

def run_functions_on_device(device, functions, credentials):
    """Run all functions sequentially on a device, add device column."""
    results = {}
    for func_name, func in functions.items():
        try:
            df = func(device, credentials)
            df['device'] = device  # add device column
            results[func_name] = df
        except Exception as e:
            results[func_name] = pd.DataFrame({"device": [device], "error": [str(e)]})
    return results

def run_family(family_name, devices, functions, credentials):
    """Run all devices in a family concurrently, functions per device sequentially."""
    merged_results = {fn: [] for fn in functions}
    with ThreadPoolExecutor(max_workers=len(devices)) as executor:
        future_to_device = {
            executor.submit(run_functions_on_device, device, functions, credentials): device
            for device in devices
        }
        for future in as_completed(future_to_device):
            device_results = future.result()
            for fn, df in device_results.items():
                merged_results[fn].append(df)
    # Merge DataFrames per function
    for fn in merged_results:
        if merged_results[fn]:
            merged_results[fn] = pd.concat(merged_results[fn], ignore_index=True)
        else:
            merged_results[fn] = pd.DataFrame()
    return merged_results

def run_all_families(inventory, family_functions):
    """Run all families concurrently and return flat dict of merged DataFrames."""
    nested_results = {}
    with ThreadPoolExecutor() as executor:
        future_to_family = {
            executor.submit(
                run_family,
                family,
                inventory[family]["devices"],
                family_functions.get(family, {}),
                {
                    "user": inventory[family]["user"],
                    "password": inventory[family]["password"]
                }
            ): family
            for family in inventory
        }

        for future in as_completed(future_to_family):
            family = future_to_family[future]
            nested_results[family] = future.result()

    # Flatten results to { "family_function": DataFrame }
    flat_results = {}
    for family, funcs in nested_results.items():
        for func_name, df in funcs.items():
            flat_results[f"{family}_{func_name}"] = df
    return flat_results

def send_to_postgres(df, table_name, if_exists="replace"):
    """Mock insertion: just print DataFrame info"""
    print(f"\n[INFO] Table '{table_name}' would be inserted with {len(df)} rows:")
    print(df)

# ---------- MAIN ----------

def main():
    results = run_all_families(inventory, FAMILY_FUNCTIONS)

    # Access each DataFrame individually
    switchshow_df = results["brocade_switchshow"]
    portshow_df = results["brocade_portshow"]
    arrayinfo_df = results["powermax_arrayinfo"]

    # Example: send to PostgreSQL (mock)
    for table_name, df in results.items():
        send_to_postgres(df, table_name)

    # Example: use in reportHelper
    # reportHelper.generate_switch_report(switchshow_df, portshow_df)

if __name__ == "__main__":
    main()
```

---

### ✅ Features

1. **Device column included automatically** in each DataFrame.
2. **Flat dictionary of merged DataFrames** for easy access:

   ```python
   results["brocade_switchshow"]
   results["powermax_arrayinfo"]
   ```
3. **Concurrent execution** per family and per device.
4. Fully self-contained **mock environment** — no real devices needed.
5. Ready to integrate with **reportHelper** or PostgreSQL.

---

If you want, I can also **add real-ish delays, random errors, and retries**, so you can test **error handling** and **robustness** like in production SAN environments.

Do you want me to do that?
