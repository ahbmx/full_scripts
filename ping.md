Perfect! Since you’re on **Linux**, we can simplify the ping command and add a `last_seen` timestamp column that updates when the ping succeeds. Here’s a complete SQLAlchemy example:

---

# ✅ **SQLAlchemy + Ping + Last Seen Timestamp**

```python
import subprocess
import time
from datetime import datetime
from sqlalchemy import create_engine, text

# -----------------------------------------
# Ping with retries (Linux)
# -----------------------------------------
def ping_with_retries(ip: str, retries: int = 3, delay: int = 3) -> bool:
    """
    Ping an IP with retries. Returns True if any ping succeeds, False otherwise.
    Linux version with 1-second timeout.
    """
    ping_cmd = ["ping", "-c", "1", "-W", "1", ip]  # -c 1 = 1 packet, -W 1 = 1 sec timeout

    for attempt in range(1, retries + 1):
        result = subprocess.run(
            ping_cmd,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL
        )
        if result.returncode == 0:
            return True
        if attempt < retries:
            time.sleep(delay)
    return False


# -----------------------------------------
# Update st_ping and last_seen using SQLAlchemy
# -----------------------------------------
def update_ping_status():
    engine = create_engine("postgresql+psycopg2://user:password@host:5432/dbname")

    with engine.begin() as conn:
        result = conn.execute(text("SELECT id, st_ip FROM storage_arrays"))

        for row in result:
            record_id = row.id
            ip_address = row.st_ip

            print(f"Pinging {ip_address} with retries...")
            success = ping_with_retries(ip_address)

            status = "success" if success else "fail"
            last_seen = datetime.now() if success else None

            conn.execute(
                text(
                    """
                    UPDATE storage_arrays 
                    SET st_ping = :status, last_seen = :last_seen
                    WHERE id = :rid
                    """
                ),
                {"status": status, "last_seen": last_seen, "rid": record_id}
            )

            print(f"Updated ID {record_id}: st_ping = {status}, last_seen = {last_seen}")


if __name__ == "__main__":
    update_ping_status()
```

---

# 📝 **Notes**

1. **Table schema assumption**:

```sql
CREATE TABLE storage_arrays (
    id SERIAL PRIMARY KEY,
    st_ip TEXT,
    st_ping TEXT,
    last_seen TIMESTAMP
);
```

2. `last_seen` is updated **only on successful ping**.
3. `st_ping` stores `"success"` or `"fail"`.
4. The script uses **Linux ping** with 1-second timeout and 3 retries with 3-second intervals.

---

If you want, I can also **make it fully parallel** so all IPs are pinged simultaneously, which is much faster for large inventories. Do you want me to do that?
