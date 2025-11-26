Ah! Got it — you don’t want to log the command or SSH outputs themselves, but instead **log the “progress” of the operation**, e.g., whether the file exists, whether the command is being executed, when it’s done, etc.

We can modify the two helper functions so that the optional `logger` is used for **status messages only**, not the output content. Here’s how that would look:

---

### Updated `functionHelper.py`

```python
import subprocess
import paramiko
from pathlib import Path

def run_or_read_command_lines(command, output_file, recreate=False, logger=None):
    """
    Runs a local command using subprocess, writes stdout and stderr to a file,
    or reads the file if it exists, returning output as a list of lines.
    Logs progress if a logger is provided.
    """
    output_path = Path(output_file)
    
    if output_path.exists() and not recreate:
        if logger:
            logger.info(f"File '{output_file}' exists. Reading contents instead of running command.")
        lines = output_path.read_text().splitlines()
        if logger:
            logger.info(f"Read {len(lines)} lines from '{output_file}'.")
        return lines
    
    if recreate and output_path.exists():
        if logger:
            logger.info(f"Recreating file '{output_file}'. Deleting existing file.")
        output_path.unlink()
    
    if logger:
        logger.info(f"Running local command: {command}")
    
    process = subprocess.Popen(
        command,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        shell=isinstance(command, str)
    )
    
    stdout, stderr = process.communicate()
    content = (stdout or "") + (stderr or "")
    lines = content.splitlines()
    
    output_path.write_text(content)
    
    if logger:
        logger.info(f"Command finished. Output written to '{output_file}' ({len(lines)} lines).")
    
    return lines


def run_or_read_ssh_command_lines(host, username, password=None, key_filepath=None,
                                  command=None, output_file=None, recreate=False,
                                  port=22, logger=None):
    """
    Runs an SSH command using paramiko, writes stdout and stderr to a file,
    or reads the file if it exists, returning output as a list of lines.
    Logs progress if a logger is provided.
    """
    output_path = Path(output_file)
    
    if output_path.exists() and not recreate:
        if logger:
            logger.info(f"File '{output_file}' exists. Reading contents instead of running SSH command.")
        lines = output_path.read_text().splitlines()
        if logger:
            logger.info(f"Read {len(lines)} lines from '{output_file}'.")
        return lines
    
    if recreate and output_path.exists():
        if logger:
            logger.info(f"Recreating file '{output_file}'. Deleting existing file.")
        output_path.unlink()
    
    if logger:
        logger.info(f"Connecting to SSH host {host}:{port} as {username}")
    
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    if key_filepath:
        key = paramiko.RSAKey.from_private_key_file(key_filepath)
        client.connect(hostname=host, username=username, pkey=key, port=port)
    else:
        client.connect(hostname=host, username=username, password=password, port=port)
    
    if logger:
        logger.info(f"Running SSH command: {command}")
    
    stdin, stdout, stderr = client.exec_command(command)
    out = stdout.read().decode()
    err = stderr.read().decode()
    
    content = (out or "") + (err or "")
    lines = content.splitlines()
    output_path.write_text(content)
    
    client.close()
    
    if logger:
        logger.info(f"SSH command finished. Output written to '{output_file}' ({len(lines)} lines).")
    
    return lines
```

---

### ✅ Example Usage

```python
from functionHelper import run_or_read_command_lines, run_or_read_ssh_command_lines
from logHelper import create_logger

logger = create_logger(log_file="progress.log")

# Local command
lines = run_or_read_command_lines("python --version", "output.log", logger=logger)

# SSH command
ssh_lines = run_or_read_ssh_command_lines(
    host="192.168.1.100",
    username="user",
    password="password",
    command="ls -l /home/user",
    output_file="ssh_output.log",
    logger=logger
)
```

---

### ✅ Behavior:

* **Logs progress only**:

  * When a file exists and will be read.
  * When a file is being recreated.
  * When a command/SSH command starts and finishes.
  * How many lines were written.

* **Output content itself is not logged**, only the status messages.

---

If you want, I can also **add timestamps and elapsed time for commands** in the logs, so you can track how long each local or SSH command took — very useful for progress monitoring.

Do you want me to add that?
