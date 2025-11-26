```python
import subprocess as sp
from pathlib import Path

def run_or_read_command_lines(cmd, output_file, recreate=False):
    output_path = Path(output_file)
    
    if output_path.exists() and not recreate:
        return output_path.read_text().splitlines()
    
    if recreate and output_path.exists():
        output_path.unlink()
    
    p = sp.Popen(cmd, stdout=sp.PIPE, stderr=sp.STDOUT, text=True, shell=isinstance(cmd, str), universal_newlines=True )
    stdout = p.communicate()
    content = (stdout or "")
    lines = content.splitlines()
    output_path.write_text(content)
    return lines
```

---

```python
import paramiko
from pathlib import Path

def run_or_read_ssh_command_lines(host, username, password=None, cmd=None, output_file=None, recreate=False, port=22):
    output_path = Path(output_file)

    if output_path.exists() and not recreate:
        return output_path.read_text().splitlines()
    
    if recreate and output_path.exists():
        output_path.unlink()
    
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    stdin, stdout, stderr = client.exec_command(cmd)
    
    out = stdout.read().decode()
    err = stderr.read().decode()
    
    content = (out or "") + (err or "")
    lines = content.splitlines()
    
    output_path.write_text(content)
    
    client.close()
    
    return lines
```






