# Running Jupyter Notebooks on HPC Compute Nodes with VS Code

A complete guide for running Jupyter Lab on GPU compute nodes using VS Code Remote-SSH with automatic port selection and secure token authentication. For connecting to HPC in the log-in node in VS Code, see [VS Code on HPC](https://wanggroup.org/hpc/docs/vscode).

## Table of Contents
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Detailed Setup](#detailed-setup)
- [Usage Instructions](#usage-instructions)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- VS Code with [Remote-SSH extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) installed
- Access to HPC cluster with SLURM scheduler
- Jupyter Lab installed in your Python environment
- SSH access configured to your HPC cluster

---

## Quick Start

```bash
# 1. Create the startup script
cat > ~/start_jupyter.sh << 'EOF'
#!/bin/bash
find_available_port() {
    local port
    for port in {8888..8988}; do
        if ! ss -tuln | grep -q ":$port "; then
            echo $port
            return 0
        fi
    done
    echo "No available ports found" >&2
    return 1
}
PORT=$(find_available_port)
if [ -z "$PORT" ]; then
    echo "Error: Could not find available port"
    exit 1
fi
rm -rf ~/.local/share/jupyter/runtime/kernel-*.json 2>/dev/null
echo "=========================================="
echo "Starting Jupyter Lab on port $PORT"
echo "=========================================="
jupyter lab --no-browser --ip=0.0.0.0 --port=$PORT \
    --ServerApp.allow_origin='*' \
    --ServerApp.allow_remote_access=True \
    --ServerApp.disable_check_xsrf=True
EOF

# 2. Make it executable
chmod +x ~/start_jupyter.sh

# 3. Request compute node and start Jupyter
srun --constraint=“Replace with available nodes” mem=12G —pty bash
./start_jupyter.sh
```

Then in VS Code:
1. Forward the port shown in output
2. Connect notebook to `http://localhost:PORT/lab?token=...`

---

## Detailed Setup

### Step 1: Create the Startup Script

Create a script `start_jupyter.sh` in your home directory with the following content:

```bash
#!/bin/bash

# Find an available port between 8888-8988
find_available_port() {
    local port
    for port in {8888..8988}; do
        if ! ss -tuln | grep -q ":$port "; then
            echo $port
            return 0
        fi
    done
    echo "No available ports found" >&2
    return 1
}

# Get available port
PORT=$(find_available_port)

if [ -z "$PORT" ]; then
    echo "Error: Could not find available port"
    exit 1
fi

# Clean up old kernel state to prevent "kernel does not exist" errors
rm -rf ~/.local/share/jupyter/runtime/kernel-*.json 2>/dev/null

echo "=========================================="
echo "Starting Jupyter Lab on port $PORT"
echo "=========================================="

# Start Jupyter with token authentication and VS Code compatibility
jupyter lab --no-browser --ip=0.0.0.0 --port=$PORT \
    --ServerApp.allow_origin='*' \
    --ServerApp.allow_remote_access=True \
    --ServerApp.disable_check_xsrf=True
```

Make the script executable:
```bash
chmod +x ~/start_jupyter.sh
```

### Step 2: Understand the Script Parameters

| Parameter | Purpose |
|-----------|---------|
| `--no-browser` | Don't open browser (we're using VS Code) |
| `--ip=0.0.0.0` | Bind to all interfaces (allows login node to connect) |
| `--port=$PORT` | Use automatically selected available port |
| `--ServerApp.allow_origin='*'` | Allow connections from VS Code |
| `--ServerApp.allow_remote_access=True` | Enable remote connections |
| `--ServerApp.disable_check_xsrf=True` | Disable XSRF checks for VS Code compatibility |

---

## Usage Instructions

### Step 1: Connect to HPC via VS Code Remote-SSH

1. Open VS Code
2. Press `F1` or `Ctrl+Shift+P` (Windows/Linux) / `Cmd+Shift+P` (Mac)
3. Type "Remote-SSH: Connect to Host"
4. Select your HPC login node from the list or enter connection details

### Step 2: Request a Compute Node with GPU

Open a terminal in VS Code and run:

```bash
srun --constraint=“Replace with available nodes” mem=12G —pty bash
```

**Verify you're on a compute node:**
```bash
hostname  # Should show compute-node-XXX, not login node
```

## Step 3: Start Jupyter Lab

```bash
./start_jupyter.sh
```

**You should see output like:**
```
==========================================
Starting Jupyter Lab on port 8891
==========================================
[I 2026-01-28 11:30:45.123 ServerApp] Jupyter Server 2.x.x is running at:
[I 2026-01-28 11:30:45.123 ServerApp] http://127.0.0.1:8891/lab?token=abc123def456ghi789jkl...
```

**📝 Copy the entire URL with token!**

### Step 4: Forward the Port in VS Code

1. In VS Code, open the **PORTS** tab (bottom panel, next to TERMINAL)
2. The port should auto-forward, or click **"Forward a Port"** 
3. Enter the port number shown in Jupyter output (e.g., `8891`)
4. Verify the port appears in the PORTS list

### Step 5: Connect Your Notebook to Jupyter

1. Open your `.ipynb` file in VS Code
2. Click **"Select Kernel"** in the top-right corner
3. Choose **"Existing Jupyter Server"**
4. Select **"Enter the URL of the running Jupyter server"**
5. Paste the URL, **replacing `127.0.0.1` with `localhost`**:
   ```
   http://localhost:8891/lab?token=abc123def456ghi789jkl...
   ```
6. Press Enter

✅ **VS Code will save the connection** - you won't need to re-enter the token for this session!

---

## Key Features

| Feature | Benefit |
|---------|---------|
| 🔄 **Automatic port selection** | No conflicts with other users |
| 🔒 **Secure token authentication** | Protects your connection |
| 🔌 **VS Code compatible** | Works seamlessly with VS Code's Jupyter extension |
| 🧹 **Clean kernel state** | Prevents "kernel does not exist" errors |
| 🖥️ **GPU compute nodes** | Runs on compute nodes, not login nodes |

---

## Security Notes

The script disables XSRF (cross-site request forgery) checks for VS Code compatibility. **This is safe because:**

- ✅ Connection is secured by SSH tunnel (only accessible through your SSH session)
- ✅ Token authentication is still active and required
- ✅ Port is not exposed to the internet
- ✅ Access is restricted to your authenticated SSH session

---

## Troubleshooting

### ❌ "Kernel does not exist" Error

**Symptom:** Error when running cells about kernel not existing

**Solution:**
1. Click the kernel name in top-right corner
2. Select **"Select Another Kernel"**
3. Choose your Python environment again (creates fresh kernel)

**Prevention:** The script automatically cleans old kernel state on startup

---

### ❌ 403 Forbidden Errors

**Symptom:** `403 GET /api/kernelspecs` errors in Jupyter output

**Solutions:**
1. ✅ Verify you're using the **full URL with token**
2. ✅ Check the script includes `--ServerApp.disable_check_xsrf=True`
3. ✅ Ensure `--ip=0.0.0.0` is set (not `127.0.0.1`)
4. ✅ Try restarting Jupyter with the updated script

---

### ❌ Connection Timeout

**Symptom:** VS Code can't connect to Jupyter server

**Solutions:**
1. Verify your `srun` job is still running:
   ```bash
   squeue -u $USER
   ```
2. Check you're on the correct compute node:
   ```bash
   hostname
   ```
3. Verify port forwarding in VS Code PORTS tab
4. Try manually forwarding the port if auto-forward didn't work

---

### ❌ Port Already in Use

**Symptom:** "Address already in use" error

**Solution:** The script automatically finds an available port. If this happens:
1. Kill any existing Jupyter processes:
   ```bash
   pkill -u $USER jupyter
   ```
2. Run the script again

---

### ❌ Job Time Limit Exceeded

**Symptom:** Jupyter suddenly stops, kernel disconnects

**Solution:** Request more time when starting:
```bash
srun --pty -t 8:00:00 -c 4 --mem=16G --gres=gpu:1 bash  # 8 hours
```

---

## Important Best Practices

### ⚠️ DO:
- ✅ Always work on compute nodes, not login nodes
- ✅ Request appropriate resources for your task
- ✅ Save your work frequently
- ✅ Monitor your job time limit
- ✅ Clean up finished jobs: `scancel <job_id>`

### ⚠️ DON'T:
- ❌ Run Jupyter on login nodes (use compute nodes)
- ❌ Request more resources than needed (wastes cluster resources)
- ❌ Leave jobs running after you're done
- ❌ Share your token with others

---

**Last Updated:** January 2026
