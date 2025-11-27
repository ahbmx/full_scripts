Below is a **full, clean Python script** that:

### ✔ Loads two Excel files

* `storage_arrays.xlsx`
* `storage_servers.xlsx`

### ✔ Detects column types and sizes for PostgreSQL (via SQLAlchemy types)

### ✔ Encrypts the column `st_password` using **Fernet**

### ✔ Uses an environment variable **`ST_TOKEN`** as the Fernet key

### ✔ Creates tables in PostgreSQL

### ✔ Pushes data using **`pandas.DataFrame.to_sql()`**

Everything is self-contained and production-ready.

---

# ✅ Full Python Script

```python
import os
import pandas as pd
from cryptography.fernet import Fernet

from sqlalchemy import (
    create_engine, MetaData, Table, Column,
    Integer, BigInteger, Float, Boolean, Text
)
from sqlalchemy.dialects.postgresql import VARCHAR, DOUBLE_PRECISION
from sqlalchemy.types import TIMESTAMP


# ------------------------------------------------------------
# 1. Read Excel files
# ------------------------------------------------------------
def load_data():
    df_arrays = pd.read_excel("storage_arrays.xlsx")
    df_servers = pd.read_excel("storage_servers.xlsx")
    return df_arrays, df_servers


# ------------------------------------------------------------
# 2. Encrypt st_password column using Fernet
# ------------------------------------------------------------
def encrypt_passwords(df):
    """
    Encrypts the st_password column using Fernet key in ST_TOKEN env variable.
    """
    key = os.getenv("ST_TOKEN")
    if key is None:
        raise ValueError("Environment variable ST_TOKEN is not set!")

    fernet = Fernet(key)

    if "st_password" in df.columns:
        df["st_password"] = df["st_password"].astype(str).apply(
            lambda x: fernet.encrypt(x.encode()).decode()
        )

    return df


# ------------------------------------------------------------
# 3. Analyze DataFrame and create SQLAlchemy type mappings
# ------------------------------------------------------------
def analyze_dataframe_for_sqlalchemy(df: pd.DataFrame):
    column_defs = {}

    for col in df.columns:
        series = df[col]
        dtype = series.dtype

        # Strings
        if dtype == object or pd.api.types.is_string_dtype(series):
            max_len = series.astype(str).str.len().max()
            if max_len and max_len > 0:
                column_defs[col] = VARCHAR(int(max_len))
            else:
                column_defs[col] = Text()

        # Integers
        elif pd.api.types.is_integer_dtype(series):
            column_defs[col] = BigInteger()

        # Floats
        elif pd.api.types.is_float_dtype(series):
            column_defs[col] = DOUBLE_PRECISION()

        # Boolean
        elif pd.api.types.is_bool_dtype(series):
            column_defs[col] = Boolean()

        # Date / DateTime
        elif pd.api.types.is_datetime64_any_dtype(series):
            column_defs[col] = TIMESTAMP()

        # Fallback
        else:
            column_defs[col] = Text()

    return column_defs


# ------------------------------------------------------------
# 4. Create SQLAlchemy table dynamically
# ------------------------------------------------------------
def create_sqlalchemy_table(table_name: str, column_defs: dict, engine):
    metadata = MetaData()

    columns = []
    for col_name, col_type in column_defs.items():
        columns.append(Column(col_name, col_type))

    table = Table(table_name, metadata, *columns, extend_existing=True)
    metadata.create_all(engine)

    return table


# ------------------------------------------------------------
# 5. Insert DataFrame into PostgreSQL using to_sql
# ------------------------------------------------------------
def dataframe_to_postgres(df, table_name, engine):
    # Detect SQLAlchemy column types
    column_defs = analyze_dataframe_for_sqlalchemy(df)

    # Create the SQL table
    create_sqlalchemy_table(table_name, column_defs, engine)

    # Insert data
    df.to_sql(
        table_name,
        engine,
        if_exists="append",
        index=False,
        dtype=column_defs
    )


# ------------------------------------------------------------
# 6. Main ETL flow
# ------------------------------------------------------------
def main():
    # Your PostgreSQL connection
    engine = create_engine("postgresql+psycopg2://user:password@localhost/mydb")

    # Load data
    df_arrays, df_servers = load_data()

    # Encrypt st_password
    df_arrays = encrypt_passwords(df_arrays)
    df_servers = encrypt_passwords(df_servers)

    # Upload to PostgreSQL
    dataframe_to_postgres(df_arrays, "storage_arrays", engine)
    dataframe_to_postgres(df_servers, "storage_servers", engine)

    print("✓ Upload completed successfully.")


if __name__ == "__main__":
    main()
```

---

# 🔐 Important Notes About Encryption

1. **ST_TOKEN must be a valid Fernet key**, e.g.:

```bash
export ST_TOKEN=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
```

2. Encrypted `st_password` becomes a URL-safe base64 string stored in PostgreSQL.

3. Decrypt example:

```python
fernet = Fernet(os.getenv("ST_TOKEN"))
plaintext = fernet.decrypt(db_value.encode()).decode()
```

---

# Want Enhancements?

I can also add:

✔ Automatic index creation on `st_device`
✔ ORM class generation
✔ Logging + exception handling
✔ Parallel loads for large files
✔ Automatic primary key detection

Just tell me!
