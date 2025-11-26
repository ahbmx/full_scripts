```python
import paramiko
from pathlib import Path

def run_or_read_ssh_command_lines(host, username, password=None, key_filepath=None,
                                  command=None, output_file=None, recreate=False, port=22):
    """
    Runs an SSH command, writes stdout and stderr to a file,
    or reads the file if it already exists, returning output as a list of lines.
    Optimized to avoid reading the file twice.
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
    
    # Write to file once
    output_path.write_text(content)
    
    client.close()
    
    # Return the lines directly
    return lines
```

---

```python
import subprocess
from pathlib import Path

def run_or_read_command_lines(command, output_file, recreate=False):
    """
    Runs a local shell command using subprocess, writes stdout and stderr to a file,
    or reads the file if it already exists, returning output as a list of lines.
    Optimized to avoid reading the file twice.
    
    Parameters:
        command (str or list): The command to execute.
        output_file (str or Path): Path to the output file.
        recreate (bool): If True, overwrite the file if it exists.
        
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
    
    # Run the subprocess
    process = subprocess.Popen(
        command,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        shell=isinstance(command, str)
    )
    
    stdout, stderr = process.communicate()
    
    # Combine stdout and stderr
    content = (stdout or "") + (stderr or "")
    
    # Split into lines before writing
    lines = content.splitlines()
    
    # Write content to file
    output_path.write_text(content)
    
    return lines

```


