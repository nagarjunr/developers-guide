# Connecting to PostgreSQL in OpenShift Using Port Forwarding

A comprehensive guide to securely accessing PostgreSQL databases running in OpenShift/Kubernetes clusters from your local machine using port forwarding.

## Table of Contents

1. [Introduction](#introduction)
2. [Why Port Forwarding?](#why-port-forwarding)
3. [Prerequisites](#prerequisites)
4. [Basic Port Forwarding](#basic-port-forwarding)
5. [Connection Methods](#connection-methods)
6. [GUI Database Tools](#gui-database-tools)
7. [Command-Line Access](#command-line-access)
8. [Automated Connection Scripts](#automated-connection-scripts)
9. [Troubleshooting](#troubleshooting)
10. [Security Best Practices](#security-best-practices)
11. [Advanced Usage](#advanced-usage)

---

## Introduction

**Port forwarding** creates a secure tunnel from your local machine to a service running in OpenShift/Kubernetes. This is the recommended method for:

- Local development and testing
- Database administration
- Running migrations
- Debugging production issues
- Using GUI database tools

**Key Benefits:**
- ✅ Secure (encrypted through kubectl/oc)
- ✅ No exposed ports or routes needed
- ✅ Works with all database clients (psql, pgAdmin, DBeaver, etc.)
- ✅ No changes to cluster configuration required
- ✅ Automatically authenticated via your OpenShift login

---

## Why Port Forwarding?

### Traditional Approach vs Port Forwarding

**Without Port Forwarding:**
```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Your Machine]:::red --> B[Public Route/LoadBalancer]:::orange
    B --> C[OpenShift]:::blue
    C --> D[PostgreSQL]:::green

    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef red fill:#E74C3C,stroke:#A93226,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
```
❌ Requires exposing database to network
❌ Additional security configuration needed
❌ May violate security policies
❌ Complex firewall rules

**With Port Forwarding:**
```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Your Machine]:::green -.Secure Tunnel.-> B[OpenShift API]:::blue
    B --> C[PostgreSQL Service]:::teal

    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
```
✅ Encrypted connection via kubectl/oc
✅ Uses your OpenShift credentials
✅ No exposed ports
✅ Works from anywhere with cluster access

---

## Prerequisites

### Required Tools

1. **kubectl or oc CLI**

   ```bash
   # Install kubectl (Kubernetes CLI)
   # macOS
   brew install kubectl

   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

   # Windows (using Chocolatey)
   choco install kubernetes-cli

   # OR Install oc CLI (OpenShift CLI)
   # Download from: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/
   ```

2. **PostgreSQL Client (psql)**

   ```bash
   # macOS
   brew install postgresql

   # Ubuntu/Debian
   sudo apt install postgresql-client

   # Fedora/RHEL
   sudo dnf install postgresql

   # Windows
   # Download from: https://www.postgresql.org/download/windows/
   ```

3. **GUI Database Tool (Optional)**
   - pgAdmin: https://www.pgadmin.org/download/
   - DBeaver: https://dbeaver.io/download/
   - DataGrip: https://www.jetbrains.com/datagrip/

### Cluster Access

Ensure you're logged into your OpenShift/Kubernetes cluster:

```bash
# For OpenShift
oc login https://api.your-cluster.com --token=your_token

# For Kubernetes
kubectl config use-context your-cluster-context

# Verify connection
oc whoami  # OpenShift
kubectl cluster-info  # Kubernetes
```

---

## Basic Port Forwarding

### Step 1: Find Your PostgreSQL Service

First, identify your PostgreSQL service name:

```bash
# List all services in your namespace
oc get svc -n your-namespace

# OR with kubectl
kubectl get svc -n your-namespace

# Look for PostgreSQL service (example output):
# NAME               TYPE        CLUSTER-IP      PORT(S)
# postgres-service   ClusterIP   10.96.123.45    5432/TCP
# app-backend        ClusterIP   10.96.123.46    8001/TCP
```

### Step 2: Start Port Forwarding

Forward the PostgreSQL port to your local machine:

```bash
# Basic syntax
oc port-forward -n <namespace> svc/<service-name> <local-port>:<remote-port>

# Example: Forward remote port 5432 to local port 5432
oc port-forward -n my-app svc/postgres-service 5432:5432

# OR using kubectl
kubectl port-forward -n my-app svc/postgres-service 5432:5432
```

**Output:**
```
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

✅ **Port forwarding is now active!** Leave this terminal window open.

### Step 3: Connect from Another Terminal

Open a new terminal and connect:

```bash
# Using psql
psql -h localhost -p 5432 -U your_username -d your_database

# Example
psql -h localhost -p 5432 -U postgres -d myapp_db
```

You'll be prompted for the database password.

---

## Connection Methods

### Method 1: Using a Different Local Port

If port 5432 is already in use locally:

```bash
# Forward to a different local port (e.g., 5433)
oc port-forward -n my-app svc/postgres-service 5433:5432

# Connect using the new local port
psql -h localhost -p 5433 -U postgres -d myapp_db
```

### Method 2: Port Forwarding to a Pod

If you don't have a service, forward directly to a pod:

```bash
# List pods
oc get pods -n my-app

# Forward to specific pod
oc port-forward -n my-app postgres-0 5432:5432

# For StatefulSets (ordered pods)
oc port-forward -n my-app postgres-0 5432:5432  # Primary pod
```

### Method 3: Background Port Forwarding

Run port forwarding in the background:

```bash
# Start in background
oc port-forward -n my-app svc/postgres-service 5432:5432 &

# Save the process ID
PF_PID=$!

# Later, kill the port forward
kill $PF_PID
```

---

## GUI Database Tools

### Using pgAdmin

**pgAdmin** is the most popular PostgreSQL management tool with a graphical interface.

#### Step 1: Install pgAdmin

```bash
# macOS
brew install --cask pgadmin4

# Ubuntu/Debian
sudo apt install pgadmin4

# Fedora/RHEL
sudo dnf install pgadmin4

# Windows
# Download from: https://www.pgadmin.org/download/
```

#### Step 2: Start Port Forwarding

```bash
oc port-forward -n my-app svc/postgres-service 5432:5432
```

Leave this terminal open.

#### Step 3: Configure pgAdmin Connection

1. **Open pgAdmin**
2. **Right-click "Servers"** → **"Create"** → **"Server"**

3. **General Tab:**
   - Name: `My App Database` (any name you prefer)

4. **Connection Tab:**
   - Host name/address: `localhost`
   - Port: `5432`
   - Maintenance database: `postgres` (or your database name)
   - Username: `postgres` (or your username)
   - Password: (your database password)
   - Save password: ☑️ (optional, for convenience)

5. **Click "Save"**

#### Step 4: Using pgAdmin

Once connected:

**View Tables:**
- Expand: Servers → My App Database → Databases → your_database → Schemas → public → Tables

**Run Queries:**
- Tools → Query Tool (or press F5)
- Write SQL and execute

**Browse Data:**
- Right-click on table → View/Edit Data → All Rows

**Export Data:**
- Right-click on table → Import/Export Data

**Visual Query Builder:**
- Query Tool → Query Builder icon

**Database Diagram:**
- Right-click on database → Generate ERD

### Using DBeaver

**DBeaver** is a universal database tool supporting many databases.

#### Quick Setup

1. **Download DBeaver:** https://dbeaver.io/download/

2. **Start Port Forwarding:**
   ```bash
   oc port-forward -n my-app svc/postgres-service 5432:5432
   ```

3. **Create Connection:**
   - Database → New Database Connection
   - Select **PostgreSQL**
   - Host: `localhost`
   - Port: `5432`
   - Database: `your_database`
   - Username: `postgres`
   - Password: (your password)
   - Test Connection → Finish

4. **Features:**
   - ER diagrams
   - Visual query builder
   - Data import/export
   - SQL formatting
   - Git integration
   - Multiple database support

### Comparison: pgAdmin vs DBeaver

| Feature | pgAdmin | DBeaver |
|---------|---------|---------|
| **PostgreSQL-specific** | ✅ Excellent | ✅ Good |
| **Multi-database support** | ❌ PostgreSQL only | ✅ MySQL, Oracle, MongoDB, etc. |
| **ER Diagrams** | ✅ Yes | ✅ Yes |
| **Visual Query Builder** | ✅ Yes | ✅ Yes |
| **Free** | ✅ Open source | ✅ Free (Enterprise paid) |
| **Resource Usage** | 🐢 Heavier | 🚀 Lighter |
| **Best For** | PostgreSQL admins | Multi-database developers |

---

## Command-Line Access

### Method 1: psql via Port Forward

```bash
# Terminal 1: Start port forward
oc port-forward -n my-app svc/postgres-service 5432:5432

# Terminal 2: Connect with psql
psql -h localhost -p 5432 -U postgres -d myapp_db
```

### Method 2: Direct Pod Access

Execute psql directly in the PostgreSQL pod (no port forwarding needed):

```bash
# Get pod name
POD_NAME=$(oc get pods -n my-app -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Connect interactively
oc exec -it $POD_NAME -n my-app -- psql -U postgres -d myapp_db

# Run single command
oc exec $POD_NAME -n my-app -- psql -U postgres -d myapp_db -c "SELECT version();"

# Run SQL from file
oc exec -i $POD_NAME -n my-app -- psql -U postgres -d myapp_db < my-query.sql
```

### Method 3: Environment Variables

Set environment variables for easier connections:

```bash
# Set connection variables
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGDATABASE=myapp_db
export PGPASSWORD=your_password  # Caution: insecure, use .pgpass instead

# Now connect without arguments
psql
```

**Better: Use .pgpass file**

Create `~/.pgpass` (more secure):

```bash
# Format: hostname:port:database:username:password
echo "localhost:5432:myapp_db:postgres:your_password" > ~/.pgpass
chmod 600 ~/.pgpass

# Now connect without password prompt
psql -h localhost -p 5432 -U postgres -d myapp_db
```

---

## Automated Connection Scripts

### Script 1: Simple Port Forward Script

Create `connect-db.sh`:

```bash
#!/bin/bash

# Configuration
NAMESPACE="my-app"
SERVICE="postgres-service"
LOCAL_PORT=5432
REMOTE_PORT=5432

# Colors for output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}Starting PostgreSQL port forward...${NC}"
echo "Namespace: $NAMESPACE"
echo "Service: $SERVICE"
echo "Local Port: $LOCAL_PORT"
echo ""
echo -e "${YELLOW}Press Ctrl+C to stop${NC}"
echo ""

# Start port forward
oc port-forward -n $NAMESPACE svc/$SERVICE $LOCAL_PORT:$REMOTE_PORT
```

Make it executable:

```bash
chmod +x connect-db.sh
./connect-db.sh
```

### Script 2: Background Port Forward with Auto-Connect

Create `db-connect.sh`:

```bash
#!/bin/bash

NAMESPACE="my-app"
SERVICE="postgres-service"
DB_USER="postgres"
DB_NAME="myapp_db"
LOCAL_PORT=5432

# Start port forward in background
echo "Starting port forward..."
oc port-forward -n $NAMESPACE svc/$SERVICE $LOCAL_PORT:5432 > /dev/null 2>&1 &
PF_PID=$!

# Wait for port forward to be ready
sleep 2

# Check if port forward is running
if ! ps -p $PF_PID > /dev/null; then
   echo "Error: Port forward failed to start"
   exit 1
fi

echo "Port forward active (PID: $PF_PID)"
echo "Connecting to database..."

# Connect to database
psql -h localhost -p $LOCAL_PORT -U $DB_USER -d $DB_NAME

# Cleanup: Kill port forward when psql exits
kill $PF_PID 2>/dev/null
echo "Port forward stopped"
```

### Script 3: Get Database Password from Secret

Create `get-db-password.sh`:

```bash
#!/bin/bash

NAMESPACE="my-app"
SECRET_NAME="postgres-secret"

# Get password from Kubernetes secret
PASSWORD=$(oc get secret $SECRET_NAME -n $NAMESPACE -o jsonpath='{.data.password}' | base64 -d)

echo "Database Password: $PASSWORD"
echo ""
echo "Connection String:"
echo "postgresql://$DB_USER:$PASSWORD@localhost:5432/$DB_NAME"
```

### Script 4: Complete Connection Helper

Create `postgres-helper.sh`:

```bash
#!/bin/bash

NAMESPACE="my-app"
SERVICE="postgres-service"
SECRET_NAME="postgres-secret"
DB_USER="postgres"
DB_NAME="myapp_db"
LOCAL_PORT=5432

# Function to start port forward
start_forward() {
    echo "Starting port forward..."
    oc port-forward -n $NAMESPACE svc/$SERVICE $LOCAL_PORT:5432 > /dev/null 2>&1 &
    PF_PID=$!
    sleep 2

    if ps -p $PF_PID > /dev/null; then
        echo "✅ Port forward active (PID: $PF_PID)"
        return 0
    else
        echo "❌ Port forward failed"
        return 1
    fi
}

# Function to stop port forward
stop_forward() {
    if [ ! -z "$PF_PID" ]; then
        kill $PF_PID 2>/dev/null
        echo "Port forward stopped"
    fi
}

# Function to get password
get_password() {
    oc get secret $SECRET_NAME -n $NAMESPACE -o jsonpath='{.data.password}' | base64 -d
}

# Trap to cleanup on exit
trap stop_forward EXIT

# Main menu
case "$1" in
    connect)
        start_forward && \
        PGPASSWORD=$(get_password) psql -h localhost -p $LOCAL_PORT -U $DB_USER -d $DB_NAME
        ;;
    password)
        echo "Password: $(get_password)"
        ;;
    forward)
        start_forward
        echo "Port forward running. Press Ctrl+C to stop."
        wait $PF_PID
        ;;
    info)
        PASSWORD=$(get_password)
        echo "Connection Information:"
        echo "  Host: localhost"
        echo "  Port: $LOCAL_PORT"
        echo "  Database: $DB_NAME"
        echo "  Username: $DB_USER"
        echo "  Password: $PASSWORD"
        echo ""
        echo "Connection String:"
        echo "  postgresql://$DB_USER:$PASSWORD@localhost:$LOCAL_PORT/$DB_NAME"
        ;;
    *)
        echo "Usage: $0 {connect|password|forward|info}"
        echo ""
        echo "Commands:"
        echo "  connect   - Start port forward and connect with psql"
        echo "  password  - Display database password"
        echo "  forward   - Start port forward only"
        echo "  info      - Show connection information"
        exit 1
        ;;
esac
```

Make it executable:

```bash
chmod +x postgres-helper.sh

# Usage examples
./postgres-helper.sh connect   # Connect to database
./postgres-helper.sh password  # Show password
./postgres-helper.sh forward   # Just forward port
./postgres-helper.sh info      # Show all connection info
```

---

## Troubleshooting

### Issue 1: Port Already in Use

**Error:**
```
bind: address already in use
```

**Solutions:**

```bash
# Option 1: Use a different local port
oc port-forward -n my-app svc/postgres-service 5433:5432

# Connect using new port
psql -h localhost -p 5433 -U postgres -d myapp_db

# Option 2: Find and kill process using port 5432
lsof -i :5432
kill <PID>

# Option 3: Use netstat (Linux)
netstat -tuln | grep 5432
```

### Issue 2: Connection Timeout

**Error:**
```
psql: error: connection to server at "localhost", port 5432 failed: Connection refused
```

**Solutions:**

```bash
# 1. Verify port forward is running
ps aux | grep "port-forward"

# 2. Check service exists
oc get svc -n my-app

# 3. Check pod is running
oc get pods -n my-app -l app=postgres

# 4. Test service connectivity from inside cluster
oc run test-pod --image=postgres:14 --rm -it --restart=Never -- \
  psql -h postgres-service.my-app.svc.cluster.local -U postgres -d myapp_db

# 5. Check service endpoints
oc get endpoints postgres-service -n my-app
```

### Issue 3: Authentication Failed

**Error:**
```
psql: error: FATAL:  password authentication failed for user "postgres"
```

**Solutions:**

```bash
# 1. Get correct password from secret
oc get secret postgres-secret -n my-app -o jsonpath='{.data.password}' | base64 -d

# 2. Verify username
oc describe pod postgres-0 -n my-app | grep POSTGRES_USER

# 3. Check environment variables in pod
oc exec postgres-0 -n my-app -- env | grep POSTGRES

# 4. Reset password (if admin)
oc exec -it postgres-0 -n my-app -- psql -U postgres -c "ALTER USER postgres PASSWORD 'new_password';"
```

### Issue 4: Port Forward Keeps Disconnecting

**Symptoms:** Port forward drops after a few minutes of inactivity

**Solutions:**

```bash
# 1. Add keepalive to SSH config (if using SSH-based kubectl)
# Edit ~/.ssh/config
Host *
  ServerAliveInterval 60
  ServerAliveCountMax 3

# 2. Use automatic reconnect script
while true; do
  oc port-forward -n my-app svc/postgres-service 5432:5432
  echo "Port forward disconnected, reconnecting in 5s..."
  sleep 5
done

# 3. Use autossh (automatic reconnect)
# Install: brew install autossh (macOS) or apt install autossh (Linux)
autossh -M 0 -f -N -L 5432:localhost:5432 oc port-forward -n my-app svc/postgres-service 5432:5432
```

### Issue 5: SSL/TLS Certificate Errors

**Error:**
```
psql: error: SSL error: certificate verify failed
```

**Solutions:**

```bash
# Option 1: Disable SSL verification (dev only!)
psql "postgresql://postgres@localhost:5432/myapp_db?sslmode=disable"

# Option 2: Use require (skip verification)
psql "postgresql://postgres@localhost:5432/myapp_db?sslmode=require"

# Option 3: Trust server certificate
psql "postgresql://postgres@localhost:5432/myapp_db?sslmode=allow"
```

### Issue 6: Permission Denied

**Error:**
```
Error from server (Forbidden): services "postgres-service" is forbidden
```

**Solutions:**

```bash
# 1. Check your permissions
oc auth can-i get pods -n my-app
oc auth can-i get services -n my-app

# 2. Verify you're in the right namespace
oc project my-app

# 3. Request access from cluster admin
# Contact your OpenShift admin for permissions

# 4. Check RBAC roles
oc get rolebindings -n my-app
```

---

## Security Best Practices

### 1. Never Store Passwords in Scripts

**❌ Bad:**
```bash
# Don't hardcode passwords!
PGPASSWORD="my_secret_password" psql -h localhost -U postgres
```

**✅ Good:**
```bash
# Use .pgpass file
echo "localhost:5432:myapp_db:postgres:my_password" > ~/.pgpass
chmod 600 ~/.pgpass

# OR: Retrieve from secret
export PGPASSWORD=$(oc get secret postgres-secret -n my-app -o jsonpath='{.data.password}' | base64 -d)
psql -h localhost -U postgres -d myapp_db
unset PGPASSWORD  # Clear after use
```

### 2. Use Read-Only Users When Possible

```sql
-- Create read-only user
CREATE USER readonly_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE myapp_db TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- Use for queries
psql -h localhost -U readonly_user -d myapp_db
```

### 3. Limit Port Forward Duration

```bash
# Auto-kill port forward after 1 hour
timeout 3600 oc port-forward -n my-app svc/postgres-service 5432:5432

# OR: Schedule cleanup
(sleep 3600 && pkill -f "port-forward.*postgres") &
oc port-forward -n my-app svc/postgres-service 5432:5432
```

### 4. Audit Access

```bash
# Log port forward usage
echo "$(date): Port forward started by $(whoami)" >> ~/.portforward.log
oc port-forward -n my-app svc/postgres-service 5432:5432
```

### 5. Use Namespaces for Isolation

```bash
# Different namespaces for different environments
oc port-forward -n dev-app svc/postgres-service 5432:5432    # Dev
oc port-forward -n staging-app svc/postgres-service 5433:5432  # Staging
oc port-forward -n prod-app svc/postgres-service 5434:5432     # Production
```

---

## Advanced Usage

### Multiple Simultaneous Port Forwards

Forward multiple services at once:

```bash
# Terminal 1: Database
oc port-forward -n my-app svc/postgres-service 5432:5432

# Terminal 2: Backend API
oc port-forward -n my-app svc/backend-service 8001:8001

# Terminal 3: Frontend
oc port-forward -n my-app svc/frontend-service 8080:8080

# Now access:
# - Database: localhost:5432
# - Backend: localhost:8001
# - Frontend: localhost:8080
```

### Port Forwarding with SSH Tunnel

Combine with SSH for remote access:

```bash
# SSH to jump host, then port forward
ssh -L 5432:localhost:5432 jump-host \
  "oc port-forward -n my-app svc/postgres-service 5432:5432"

# Now connect from local machine
psql -h localhost -p 5432 -U postgres -d myapp_db
```

### Dynamic Port Allocation

Let kubectl choose a free port:

```bash
# Use :0 for automatic port assignment
oc port-forward -n my-app svc/postgres-service :5432

# Output will show assigned port:
# Forwarding from 127.0.0.1:54321 -> 5432
# Connect using the assigned port (54321 in this example)
```

### Port Forward with Background Process Manager

Use `tmux` or `screen` for persistent port forwards:

```bash
# Using tmux
tmux new-session -d -s db-forward "oc port-forward -n my-app svc/postgres-service 5432:5432"

# List tmux sessions
tmux ls

# Attach to session
tmux attach -t db-forward

# Detach: Press Ctrl+b, then d

# Kill session
tmux kill-session -t db-forward
```

### Monitoring Port Forward Health

```bash
#!/bin/bash

# Health check script
while true; do
  if ! lsof -i :5432 > /dev/null; then
    echo "Port forward down, restarting..."
    oc port-forward -n my-app svc/postgres-service 5432:5432 &
  fi
  sleep 30
done
```

---

## Quick Reference

### Essential Commands

```bash
# Start port forward
oc port-forward -n <namespace> svc/<service-name> <local-port>:<remote-port>

# Connect with psql
psql -h localhost -p <local-port> -U <username> -d <database>

# Get password from secret
oc get secret <secret-name> -n <namespace> -o jsonpath='{.data.password}' | base64 -d

# List services
oc get svc -n <namespace>

# Check pod status
oc get pods -n <namespace>

# Kill port forward
pkill -f "port-forward"
```

### Connection String Format

```
postgresql://username:password@localhost:port/database
```

Example:
```
postgresql://postgres:mypassword@localhost:5432/myapp_db
```

---

## Summary

**Port Forwarding Workflow:**

1. **Login to cluster:** `oc login https://your-cluster.com`
2. **Find service:** `oc get svc -n your-namespace`
3. **Start port forward:** `oc port-forward -n your-namespace svc/postgres-service 5432:5432`
4. **Get password:** `oc get secret postgres-secret -n your-namespace -o jsonpath='{.data.password}' | base64 -d`
5. **Connect:** `psql -h localhost -p 5432 -U postgres -d your_database`

**When to Use:**
- ✅ Local development
- ✅ Database administration (GUI tools)
- ✅ Running migrations
- ✅ Debugging issues
- ✅ Quick data queries

**When NOT to Use:**
- ❌ Production application traffic (use services/routes)
- ❌ Long-running background jobs
- ❌ High-volume data transfers
- ❌ Automated scripts (use in-cluster access)

**Best Practices:**
- Use read-only users when possible
- Store passwords in `.pgpass` or secrets
- Limit port forward duration
- Use different ports for different environments
- Always close port forwards when done

---

## Further Reading

- [Kubernetes Port Forwarding Documentation](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
- [OpenShift CLI Reference](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html)
- [PostgreSQL Connection Strings](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING)
- [pgAdmin Documentation](https://www.pgadmin.org/docs/)
- [DBeaver Documentation](https://github.com/dbeaver/dbeaver/wiki)

---

**Created:** 2026-02-06
**Tags:** #postgresql #openshift #kubernetes #port-forwarding #database #devops #kubectl #oc
