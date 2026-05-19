Docker Desktop Kubernetes Issue (Windows + WSL2)
📌 Environment
Windows + WSL2
Ubuntu installed in WSL
Docker Desktop enabled Kubernetes
kubectl used for cluster access
Helm not working initially
🚨 Original Problem Symptoms
1. WSL networking failure
Wsl/Service/0x80072747
2. Docker CLI config failure
docker cli config: failed to write file
exit status 0xffffffff
3. kubectl error
http://localhost:8080/api
connection refused
4. Kubernetes stuck
Docker Desktop → “Starting Kubernetes…”
5. kubectl context missing
current-context is not set
🧩 Root Cause Chain (IMPORTANT)

This is the actual dependency chain:

Windows Networking Stack
        ↓
WSL2 Virtual Network (Ubuntu + Docker WSL distros)
        ↓
Docker Desktop internal VM (docker-desktop)
        ↓
Kubernetes API Server bootstrap
        ↓
kubectl connects via kubeconfig context
💥 What broke
❌ 1. WSL networking stack crashed
caused by socket buffer exhaustion / network reset error
broke virtual NAT networking
❌ 2. Docker WSL integration got corrupted
docker CLI config failed
Kubernetes VM could not initialize networking
❌ 3. kubeconfig not generated
kubectl had no cluster context
fallback → localhost:8080
🔧 Why Kubernetes got stuck

Docker Desktop Kubernetes needs:

internal VM startup
network bridge creation
API server boot (port 6443)
kubeconfig context injection

Because WSL networking was broken earlier:

👉 VM could not fully initialize
👉 Kubernetes stayed stuck at “Starting…”

🧪 Why kubectl showed localhost:8080

When no context exists:

kubectl default behavior
→ tries localhost:8080
→ no cluster running there
→ connection refused
🟢 Final State After Fix

Once resolved:

Kubernetes VM booted correctly
kubeconfig generated
context created:
docker-desktop

Now:

kubectl → kubeconfig → Docker Desktop cluster → Kubernetes API