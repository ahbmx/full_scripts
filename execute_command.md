```python
import subprocess
from pathlib import Path

def run_or_read_command(command, output_file, recreate=False):
    output_path = Path(output_file)

    if output_path.exists() and not recreate:
        return output_path.read_text()
    
    if recreate and output_path.exists():
        output_path.unlink()
    
    process = subprocess.Popen(
        command,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        shell=isinstance(command, str)
    )
    
    stdout, stderr = process.communicate()
    
    content = (stdout or "") + (stderr or "")

    output_path.write_text(content)
    
    return content.splitlines()
```

### ✅ Usage:

```python
# File does not exist, will create it
output = run_or_read_command("python --version", "output.log")
print(output)

# File exists, read contents instead of running
output = run_or_read_command("python --version", "output.log")
print(output)

# Force recreate and rerun command
output = run_or_read_command("python --version", "output.log", recreate=True)
print(output)
```

This now **returns the raw output exactly as the command produced it** and writes it to the file without any extra labels.

---

If you want, I can also make it **write output to the file in real-time while still capturing it**, which is very useful for commands that take a long time. It’s a small tweak but much more user-friendly for long-running processes. Do you want me to do that?



Perfect! We can make a version that **returns the SSH command output as a list of lines**, similar to how you split local command output. This works with your `run_or_read_ssh_command` logic:

* Reads the file if it exists and `recreate=False`.
* Otherwise, runs the SSH command, writes output to file, and returns it as a **list of lines**.

Here’s the implementation:

```python
import paramiko
from pathlib import Path

def run_or_read_ssh_command_lines(host, username, password=None, key_filepath=None,
                                  command=None, output_file=None, recreate=False, port=22):
    """
    Runs an SSH command, writes stdout and stderr to a file,
    or reads the file if it already exists, returning output as a list of lines.

    Parameters:
        host (str): SSH server hostname or IP.
        username (str): SSH username.
        password (str): SSH password (optional if using key).
        key_filepath (str or Path): Path to private key file (optional).
        command (str): Command to execute on the remote host.
        output_file (str or Path): Path to the output file.
        recreate (bool): If True, overwrite the file if it exists.
        port (int): SSH port (default 22).

    Returns:
        List[str]: Lines of combined stdout and stderr output.
    """
    output_path = Path(output_file)
    
    # If file exists and recreate is False, read lines and return
    if output_path.exists() and not recreate:
        return output_path.read_text().splitlines()
    
    # If recreate is True and file exists, delete it
    if recreate and output_path.exists():
        output_path.unlink()
    
    # Create SSH client
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    # Connect using password or key
    if key_filepath:
        key = paramiko.RSAKey.from_private_key_file(key_filepath)
        client.connect(hostname=host, username=username, pkey=key, port=port)
    else:
        client.connect(hostname=host, username=username, password=password, port=port)
    
    # Execute the command
    stdin, stdout, stderr = client.exec_command(command)
    
    # Read stdout and stderr
    out = stdout.read().decode()
    err = stderr.read().decode()
    
    # Combine output and split into lines
    content = (out or "") + (err or "")
    lines = content.splitlines()
    
    # Write to file
    output_path.write_text(content)
    
    # Close connection
    client.close()
    
    return lines
```

### ✅ Usage Example:

```python
lines = run_or_read_ssh_command_lines(
    host="192.168.1.100",
    username="user",
    password="password",
    command="ls -l /home/user",
    output_file="ssh_output.log",
    recreate=False
)

# Process each line
for line in lines:
    print(f"SSH line: {line}")
```

**Behavior:**

1. Returns **a list of lines** from the SSH command output.
2. Reads the file if it exists (`recreate=False`) instead of re-running the command.
3. Otherwise, runs the command, writes output to the file, and returns the lines.
4. Both `stdout` and `stderr` are included.

---

If you want, I can also make a **hybrid version that first reads the file if it exists, but still allows streaming new output line by line if the file doesn’t exist**, combining the best of both approaches for SSH commands.

Do you want me to do that?


