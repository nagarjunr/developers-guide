# Enterprise Python Setup: Corporate Networks, Proxies, and SSL Certificates

A complete guide to setting up Python development environments in enterprise/corporate networks with SSL certificates, proxy configurations, and secure package installation.

## Table of Contents

1. [The Corporate Network Challenge](#the-corporate-network-challenge)
2. [Understanding Corporate SSL Inspection](#understanding-corporate-ssl-inspection)
3. [Why We Need Both System and Corporate CAs](#why-we-need-both-system-and-corporate-cas)
4. [Combined CA Bundle Solution](#combined-ca-bundle-solution)
5. [Environment Configuration](#environment-configuration)
6. [Python Application Patterns](#python-application-patterns)
7. [Virtual Environment Setup](#virtual-environment-setup)
8. [Package Installation](#package-installation)
9. [Complete Setup Script](#complete-setup-script)
10. [Troubleshooting](#troubleshooting)
11. [Best Practices](#best-practices)

---

## The Corporate Network Challenge

**Common scenario:** You're developing Python applications in a corporate environment and encounter:

```
❌ SSL: CERTIFICATE_VERIFY_FAILED
❌ Connection refused: pypi.org
❌ Timeout when installing packages
❌ Git clone fails with SSL error
❌ MCP server fails to start
❌ API requests fail with certificate errors
```

**Why this happens:**
- Corporate networks use **proxy servers** for internet access
- Proxy servers use **SSL inspection** (man-in-the-middle)
- Proxy re-signs certificates with company's internal CA
- Python doesn't trust these certificates by default
- Applications reject the "untrusted" certificate

**This guide solves:**
✅ SSL certificate verification errors
✅ Proxy configuration for pip, requests, and all Python apps
✅ Virtual environment isolation
✅ Secure package installation in corporate environments
✅ MCP servers, web apps, CLI tools, and more

---

## Understanding Corporate SSL Inspection

### Normal SSL Flow (Home/Personal Network)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Python App]:::green -->|1️⃣ HTTPS Request| B[pypi.org]:::blue
    B -->|2️⃣ Certificate<br/>Signed by DigiCert| A
    A -->|3️⃣ ✓ Trusts DigiCert| C[Success ✓]:::teal

    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
```

**Flow:**
1. Your Python app makes HTTPS request to pypi.org
2. PyPI returns certificate signed by DigiCert (public CA)
3. Python trusts DigiCert → Connection succeeds ✓

### Corporate Network with SSL Inspection

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Python App]:::orange -->|1️⃣ HTTPS Request| P[Corporate Proxy]:::red
    P -->|2️⃣ Intercepts<br/>& Inspects| S[pypi.org]:::blue
    S -->|3️⃣ Real Certificate<br/>DigiCert| P
    P -->|4️⃣ New Certificate<br/>CompanyCA| A
    A -->|5️⃣ ✗ Unknown CA| X[Rejected ✗]:::purple

    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef red fill:#E74C3C,stroke:#A93226,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef purple fill:#9B59B6,stroke:#6C3483,color:#fff
```

**Flow:**
1. Your Python app makes HTTPS request
2. Corporate proxy intercepts and inspects traffic (authorized MITM)
3. Proxy receives real certificate from PyPI (DigiCert)
4. Proxy sends NEW certificate signed by CompanyCA to your app
5. Python doesn't trust CompanyCA → Connection rejected ✗

**What the proxy does:**
1. **Intercepts** HTTPS connection
2. **Decrypts** traffic (authorized MITM for security inspection)
3. **Inspects** content for threats, data loss prevention
4. **Re-encrypts** with corporate CA certificate
5. **Forwards** to your application

**Result:** Python sees certificate signed by unknown CA → **Rejection**

---

## Why We Need Both System and Corporate CAs

### The Problem with Using Only Corporate CA

Your Python applications connect to **two types** of services:

#### 1. **Public Services** (PyPI, GitHub, NPM, OpenAI, etc.)

These use certificates signed by **public certificate authorities**:

| Service | CA Provider |
|---------|-------------|
| pypi.org | DigiCert |
| github.com | DigiCert |
| api.openai.com | DigiCert |
| npmjs.com | Let's Encrypt |
| docker.io | DigiCert |

**What happens with corporate CA only:**
```python
# Using only corporate CA
CA_BUNDLE = "/path/to/corporate-ca.pem"

requests.get("https://pypi.org", verify=CA_BUNDLE)
# ❌ SSLError: certificate verify failed
# Why? DigiCert signed pypi.org's cert, but corporate CA doesn't trust DigiCert
```

#### 2. **Internal Services** (Behind Corporate Proxy)

When the proxy intercepts traffic, it re-signs with corporate CA:

```python
# Through corporate proxy
requests.get("https://internal-jira.company.com", verify=CA_BUNDLE)
# ✓ Works - corporate CA signed this certificate
```

### The Solution: Combined Bundle

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
flowchart TD
    A[System CA Bundle<br/>certifi<br/>195+ Public CAs]:::blue
    B[Corporate CA<br/>CompanyCA]:::orange
    C[Combined CA Bundle<br/>~/.ssl/combined-ca-bundle.pem]:::green
    D[Public Services<br/>pypi.org, github.com<br/>api.openai.com]:::teal
    E[Internal Services<br/>jira.company.com<br/>confluence.company.com]:::teal

    A -->|1️⃣ Merge| C
    B -->|2️⃣ Merge| C
    C -->|3️⃣ Trusts Both| D
    C -->|3️⃣ Trusts Both| E

    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
```

**How it works:**
1. Merge system CA bundle (certifi with 195+ public CAs like DigiCert, Let's Encrypt)
2. Add corporate CA certificate
3. Combined bundle trusts BOTH public and internal services ✓

### Real-World Example

```python
import requests

# 1. Public service (needs system CAs)
response = requests.get("https://pypi.org/pypi/requests/json")
# Certificate chain: pypi.org → DigiCert → DigiCert Root CA
# Need: DigiCert Root CA from system bundle ✓

# 2. Internal service through proxy (needs corporate CA)
response = requests.get("https://jira.company.com/rest/api/2/myself")
# Certificate chain: jira.company.com → Corporate Proxy → CompanyCA
# Need: CompanyCA from corporate bundle ✓

# 3. Combined bundle trusts BOTH
combined_bundle = "~/.ssl/combined-ca-bundle.pem"
requests.get("https://pypi.org", verify=combined_bundle)  # ✓ Works
requests.get("https://jira.company.com", verify=combined_bundle)  # ✓ Works
```

### Key Takeaway

**System certificates alone:** ✓ Public services | ✗ Corporate proxy
**Corporate CA alone:** ✗ Public services | ✓ Corporate proxy
**Combined bundle:** ✓ Public services | ✓ Corporate proxy

---

## Combined CA Bundle Solution

### Step 1: Locate Corporate CA Certificate

Corporate IT usually provides CA certificates as `.pem`, `.crt`, or `.cer` files.

**Common locations:**
- Email from IT department
- Company intranet download page
- Pre-installed on company laptops
- Windows Certificate Store (export to file)

**Example:** Download and save to `~/.ssl/corporate-ca.pem`

### Step 2: Create Combined CA Bundle

**Automated script:**

```bash
#!/bin/bash
# create_combined_ca_bundle.sh

# Get Python's certifi bundle path
CERTIFI_BUNDLE=$(python3 -c "import certifi; print(certifi.where())")

# Path to corporate CA certificate
CORPORATE_CA="$HOME/.ssl/corporate-ca.pem"

# Output path for combined bundle
COMBINED_BUNDLE="$HOME/.ssl/combined-ca-bundle.pem"

# Verify files exist
if [ ! -f "$CERTIFI_BUNDLE" ]; then
    echo "❌ Certifi bundle not found. Install: pip install certifi"
    exit 1
fi

if [ ! -f "$CORPORATE_CA" ]; then
    echo "❌ Corporate CA not found: $CORPORATE_CA"
    echo "   Contact IT or download from company intranet"
    exit 1
fi

# Create output directory
mkdir -p "$(dirname "$COMBINED_BUNDLE")"

# Backup existing bundle
if [ -f "$COMBINED_BUNDLE" ]; then
    cp "$COMBINED_BUNDLE" "${COMBINED_BUNDLE}.backup.$(date +%Y%m%d)"
fi

# Create combined bundle
cat "$CERTIFI_BUNDLE" "$CORPORATE_CA" > "$COMBINED_BUNDLE"

# Count certificates
CERTIFI_COUNT=$(grep -c "BEGIN CERTIFICATE" "$CERTIFI_BUNDLE")
CORPORATE_COUNT=$(grep -c "BEGIN CERTIFICATE" "$CORPORATE_CA")
COMBINED_COUNT=$(grep -c "BEGIN CERTIFICATE" "$COMBINED_BUNDLE")

echo "✅ Created combined CA bundle: $COMBINED_BUNDLE"
echo "   Certifi:   $CERTIFI_COUNT certificates"
echo "   Corporate: $CORPORATE_COUNT certificates"
echo "   Combined:  $COMBINED_COUNT certificates"
echo "   Size: $(du -h "$COMBINED_BUNDLE" | cut -f1)"
```

**Run:**
```bash
chmod +x create_combined_ca_bundle.sh
./create_combined_ca_bundle.sh
```

---

## Environment Configuration

### Shell Environment Variables

**File: `~/.zshrc` or `~/.bashrc`**

```bash
# ============================================================================
# Corporate Network Configuration
# ============================================================================

# SSL Certificates - Combined bundle (public + corporate CAs)
export COMBINED_CA_BUNDLE=~/.ssl/combined-ca-bundle.pem
export SSL_CERT_FILE=$COMBINED_CA_BUNDLE
export REQUESTS_CA_BUNDLE=$COMBINED_CA_BUNDLE
export CURL_CA_BUNDLE=$COMBINED_CA_BUNDLE

# Language-specific CA paths
export NODE_EXTRA_CA_CERTS=$COMBINED_CA_BUNDLE  # Node.js
export AWS_CA_BUNDLE=~/.ssl/corporate-ca.pem     # AWS SDK (may need corporate CA only)

# Corporate Proxy Configuration
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,*.company.com,10.0.0.0/8"

# Lowercase versions (some tools check these)
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
export no_proxy="$NO_PROXY"

# Git SSL Configuration
export GIT_SSL_CAINFO=$COMBINED_CA_BUNDLE
```

**Apply changes:**
```bash
source ~/.zshrc  # or source ~/.bashrc
```

### Verify Configuration

```bash
# Check environment variables
echo "SSL_CERT_FILE: $SSL_CERT_FILE"
echo "HTTP_PROXY: $HTTP_PROXY"
echo "NO_PROXY: $NO_PROXY"

# Test SSL with curl
curl -I https://pypi.org

# Test SSL with Python
python3 -c "import requests; print(requests.get('https://pypi.org').status_code)"
```

---

## Python Application Patterns

### Pattern 1: General HTTP Requests (requests library)

```python
import requests
import os

# Get CA bundle from environment
CA_BUNDLE = os.path.expanduser(os.getenv(
    "COMBINED_CA_BUNDLE",
    "~/.ssl/combined-ca-bundle.pem"
))

# Single request
response = requests.get(
    "https://api.example.com/data",
    verify=CA_BUNDLE
)

# Session (recommended for multiple requests)
session = requests.Session()
session.verify = CA_BUNDLE
session.headers.update({"Authorization": "Bearer token"})

response = session.get("https://api.example.com/data")
```

### Pattern 2: Web Applications (FastAPI, Flask, Django)

```python
import os
import requests
from fastapi import FastAPI

app = FastAPI()

# Configure CA bundle globally
CA_BUNDLE = os.path.expanduser(os.getenv(
    "COMBINED_CA_BUNDLE",
    "~/.ssl/combined-ca-bundle.pem"
))

# Create session for external API calls
http_session = requests.Session()
http_session.verify = CA_BUNDLE

@app.get("/fetch-data")
async def fetch_external_data():
    """Endpoint that calls external API."""
    response = http_session.get("https://api.external.com/data")
    return response.json()
```

### Pattern 3: MCP Servers (Model Context Protocol)

```python
from atlassian import Jira  # or any other API client
import os
import logging

# SSL verification - use combined CA bundle
CA_BUNDLE = os.path.expanduser(os.getenv(
    "COMBINED_CA_BUNDLE",
    "~/.ssl/combined-ca-bundle.pem"
))

# Initialize client with SSL verification
jira = Jira(
    url=os.getenv("JIRA_URL"),
    token=os.getenv("JIRA_TOKEN"),
    verify_ssl=CA_BUNDLE  # Pass CA bundle to client
)

# Validate connection
try:
    jira.get("/rest/api/2/myself")
    logging.info("✓ Connected to Jira with SSL verification enabled")
except Exception as e:
    logging.error(f"✗ Failed to connect: {e}")
    raise

# MCP configuration in .vscode/mcp.json:
# {
#   "servers": {
#     "jira-mcp-server": {
#       "command": "uv",
#       "args": ["--directory", "/path/to/mcp", "run", "server.py"],
#       "env": {
#         "JIRA_URL": "https://jira.company.com",
#         "JIRA_TOKEN": "your-token",
#         "COMBINED_CA_BUNDLE": "/Users/username/.ssl/combined-ca-bundle.pem",
#         "HTTP_PROXY": "http://proxy.company.com:8080",
#         "HTTPS_PROXY": "http://proxy.company.com:8080"
#       }
#     }
#   }
# }
```

### Pattern 4: CLI Tools

```python
#!/usr/bin/env python3
"""CLI tool that makes HTTPS requests."""

import os
import requests
import click

CA_BUNDLE = os.path.expanduser(os.getenv(
    "COMBINED_CA_BUNDLE",
    "~/.ssl/combined-ca-bundle.pem"
))

@click.command()
@click.argument('url')
def fetch(url):
    """Fetch data from URL with proper SSL verification."""
    try:
        response = requests.get(url, verify=CA_BUNDLE)
        click.echo(f"✓ Status: {response.status_code}")
        click.echo(response.text)
    except requests.exceptions.SSLError as e:
        click.echo(f"✗ SSL Error: {e}", err=True)
        click.echo(f"CA Bundle: {CA_BUNDLE}", err=True)
        raise SystemExit(1)

if __name__ == "__main__":
    fetch()
```

### Pattern 5: Background Workers/Celery Tasks

```python
from celery import Celery
import requests
import os

app = Celery('tasks')

# Configure CA bundle for all tasks
CA_BUNDLE = os.path.expanduser(os.getenv(
    "COMBINED_CA_BUNDLE",
    "~/.ssl/combined-ca-bundle.pem"
))

# Create shared session
http_session = requests.Session()
http_session.verify = CA_BUNDLE

@app.task
def fetch_external_data(api_url):
    """Background task to fetch data from external API."""
    response = http_session.get(api_url)
    return response.json()
```

### Pattern 6: Testing with pytest

```python
# conftest.py
import pytest
import os

@pytest.fixture(scope="session")
def ca_bundle():
    """Provide CA bundle path for all tests."""
    return os.path.expanduser(os.getenv(
        "COMBINED_CA_BUNDLE",
        "~/.ssl/combined-ca-bundle.pem"
    ))

@pytest.fixture(scope="session")
def http_session(ca_bundle):
    """Provide configured requests session."""
    import requests
    session = requests.Session()
    session.verify = ca_bundle
    return session

# test_api.py
def test_api_call(http_session):
    """Test external API call with proper SSL."""
    response = http_session.get("https://api.example.com/health")
    assert response.status_code == 200
```

---

## Virtual Environment Setup

### Step 1: Create Virtual Environment

```bash
# Use specific Python version
python3.11 -m venv venv

# Activate (Unix/macOS)
source venv/bin/activate

# Activate (Windows)
venv\Scripts\activate

# Verify
which python  # Should show: /path/to/project/venv/bin/python
python --version
```

### Step 2: Upgrade pip

```bash
python -m pip install --upgrade pip
pip --version
```

---

## Package Installation

### Step 1: Install certifi

```bash
# Install certifi first (may need --trusted-host for first install)
pip install --upgrade certifi \
    --trusted-host pypi.org \
    --trusted-host files.pythonhosted.org
```

### Step 2: Recreate Combined Bundle

After installing certifi in your venv, recreate the combined bundle:

```bash
# Run the script again to use venv's certifi
./create_combined_ca_bundle.sh
```

### Step 3: Install Dependencies

```bash
# Now pip will use the combined CA bundle (from environment variables)
pip install -r requirements.txt

# Or install individual packages
pip install requests fastapi httpx
```

---

## Complete Setup Script

**File: `setup.sh`**

```bash
#!/bin/bash
# Enterprise Python setup script
# Handles venv, CA certs, proxies, and dependencies

set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$PROJECT_ROOT"

echo "=== Enterprise Python Setup ==="

# 1. Choose Python version
PYTHON_BIN="python3"
if command -v python3.12 >/dev/null 2>&1; then
    PYTHON_BIN="python3.12"
elif command -v python3.11 >/dev/null 2>&1; then
    PYTHON_BIN="python3.11"
fi

echo "[1/5] Using Python: $PYTHON_BIN ($($PYTHON_BIN --version))"

# 2. Create virtual environment
VENV_DIR="venv"
if [ ! -d "$VENV_DIR" ]; then
    echo "[2/5] Creating virtual environment..."
    "$PYTHON_BIN" -m venv "$VENV_DIR"
else
    echo "[2/5] Virtual environment already exists"
fi

# Activate venv
source "$VENV_DIR/bin/activate"

# 3. Upgrade pip and install certifi
echo "[3/5] Installing certifi..."
PIP_TRUSTED_HOSTS=("--trusted-host" "pypi.org" "--trusted-host" "files.pythonhosted.org")
python -m pip install --upgrade pip certifi "${PIP_TRUSTED_HOSTS[@]}" >/dev/null 2>&1

# 4. Create combined CA bundle
echo "[4/5] Creating combined CA bundle..."
CERTIFI_BUNDLE=$(python -c 'import certifi; print(certifi.where())')
CORPORATE_CA="${CORPORATE_CA_PATH:-$HOME/.ssl/corporate-ca.pem}"
COMBINED_BUNDLE="$HOME/.ssl/combined-ca-bundle.pem"

mkdir -p "$(dirname "$COMBINED_BUNDLE")"

if [ -f "$CORPORATE_CA" ]; then
    cat "$CERTIFI_BUNDLE" "$CORPORATE_CA" > "$COMBINED_BUNDLE"
    echo "    ✓ Created: $COMBINED_BUNDLE"
else
    echo "    ⚠️  Corporate CA not found: $CORPORATE_CA"
    echo "    → Using certifi bundle only"
    cp "$CERTIFI_BUNDLE" "$COMBINED_BUNDLE"
fi

# 5. Set environment variables
echo "[5/5] Configuring environment..."
export SSL_CERT_FILE="$COMBINED_BUNDLE"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
export COMBINED_CA_BUNDLE="$SSL_CERT_FILE"

# Load .env if exists
if [ -f .env ]; then
    set -a
    source .env
    set +a
fi

echo "    - SSL_CERT_FILE: $SSL_CERT_FILE"
echo "    - HTTP_PROXY: ${HTTP_PROXY:-<not set>}"

# 6. Install dependencies
if [ -f requirements.txt ]; then
    echo "[6/5] Installing dependencies..."
    python -m pip install -r requirements.txt --quiet
    echo "    ✓ Dependencies installed"
fi

echo ""
echo "✅ Setup complete!"
echo ""
echo "Activate the virtual environment:"
echo "  source venv/bin/activate"
```

---

## Troubleshooting

### Issue 1: SSL Certificate Verify Failed

**Symptoms:**
```
ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED]
```

**Solutions:**

1. **Verify combined bundle exists:**
   ```bash
   ls -lh ~/.ssl/combined-ca-bundle.pem
   # Should show file ~300-400KB
   ```

2. **Check certificate count:**
   ```bash
   grep -c "BEGIN CERTIFICATE" ~/.ssl/combined-ca-bundle.pem
   # Should show 195+ certificates
   ```

3. **Test with curl:**
   ```bash
   curl --cacert ~/.ssl/combined-ca-bundle.pem https://pypi.org
   ```

4. **Test with Python:**
   ```python
   import requests
   response = requests.get(
       "https://pypi.org",
       verify="/Users/username/.ssl/combined-ca-bundle.pem"
   )
   print(response.status_code)  # Should print 200
   ```

5. **Check environment variables:**
   ```bash
   echo $SSL_CERT_FILE
   echo $REQUESTS_CA_BUNDLE
   # Both should point to combined bundle
   ```

### Issue 2: Proxy Connection Failed

**Symptoms:**
```
ProxyError: HTTPSConnectionPool(host='pypi.org', port=443): Max retries exceeded
```

**Solutions:**

1. **Verify proxy settings:**
   ```bash
   echo $HTTP_PROXY
   echo $HTTPS_PROXY
   # Should show proxy URL
   ```

2. **Test proxy with curl:**
   ```bash
   curl -I https://pypi.org
   # Should return 200 OK
   ```

3. **Check NO_PROXY:**
   ```bash
   echo $NO_PROXY
   # Should include localhost,127.0.0.1
   ```

4. **Add credentials if needed:**
   ```bash
   export HTTP_PROXY="http://username:password@proxy.company.com:8080"
   ```

### Issue 3: MCP Server Not Loading Environment

**Problem:** MCP server in VS Code doesn't see environment variables from shell.

**Solution:** Hardcode in `.vscode/mcp.json`:

```json
{
  "servers": {
    "my-mcp-server": {
      "command": "uv",
      "args": ["run", "server.py"],
      "env": {
        "COMBINED_CA_BUNDLE": "/Users/username/.ssl/combined-ca-bundle.pem",
        "HTTP_PROXY": "http://proxy.company.com:8080",
        "HTTPS_PROXY": "http://proxy.company.com:8080"
      }
    }
  }
}
```

**Note:** VS Code doesn't automatically load shell environment variables for MCP servers.

### Issue 4: Certificate Expired

**Symptoms:**
```
SSLError: certificate has expired
```

**Solution:** Get updated corporate CA from IT, then recreate combined bundle:

```bash
# Update corporate CA
# Then recreate bundle
./create_combined_ca_bundle.sh
```

---

## Best Practices

### 1. Automate Setup

Create `setup.sh` script (see above) for reproducible setup across:
- New machines
- Team onboarding
- CI/CD environments
- Docker containers

### 2. Document in README

```markdown
## Corporate Network Setup

This project requires additional configuration for corporate networks.

### Prerequisites
- Python 3.11+
- Corporate CA certificate (contact IT)
- Proxy configuration

### Quick Start

1. Get corporate CA certificate from IT
2. Save to `~/.ssl/corporate-ca.pem`
3. Run setup:
   ```bash
   ./setup.sh
   source venv/bin/activate
   ```

### Troubleshooting
See [docs/ENTERPRISE_SETUP.md](docs/ENTERPRISE_SETUP.md)
```

### 3. Use Environment Files

**.env.example:**
```bash
# API Configuration
API_URL=https://api.company.com
API_TOKEN=your-token-here

# SSL Configuration
COMBINED_CA_BUNDLE=/Users/username/.ssl/combined-ca-bundle.pem

# Proxy Configuration (for corporate network)
HTTP_PROXY=http://proxy.company.com:8080
HTTPS_PROXY=http://proxy.company.com:8080
NO_PROXY=localhost,127.0.0.1,*.company.com
```

### 4. Never Commit Secrets

**.gitignore:**
```
# Virtual environments
venv/
.venv/

# SSL certificates
*.pem
*.crt
*.cer
certs/

# Environment files
.env
.env.local

# MCP configuration with hardcoded tokens
.vscode/mcp.json
```

### 5. Update Bundle Regularly

Run after:
- Python/certifi upgrades: `pip install --upgrade certifi`
- Corporate CA updates
- Switching projects

```bash
# Update script
./create_combined_ca_bundle.sh
```

### 6. Test Before Deploying

```python
#!/usr/bin/env python3
"""Test enterprise setup."""

import sys
import os

def test_ca_bundle():
    """Test CA bundle exists."""
    bundle = os.path.expanduser("~/.ssl/combined-ca-bundle.pem")
    if os.path.exists(bundle):
        print(f"✓ CA bundle exists: {bundle}")
        return True
    print(f"✗ CA bundle missing: {bundle}")
    return False

def test_ssl():
    """Test SSL connection."""
    import requests
    try:
        response = requests.get("https://pypi.org", timeout=5)
        print(f"✓ SSL test passed: {response.status_code}")
        return True
    except Exception as e:
        print(f"✗ SSL test failed: {e}")
        return False

if __name__ == "__main__":
    results = [test_ca_bundle(), test_ssl()]
    sys.exit(0 if all(results) else 1)
```

---

## Summary

**Key Components:**

1. **Combined CA Bundle** = System CAs (certifi) + Corporate CA
2. **Environment Variables** = `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `HTTP_PROXY`
3. **Python Code Pattern** = Pass `CA_BUNDLE` to `verify` parameter
4. **Automation** = Setup script for reproducibility

**Checklist:**

- [ ] Corporate CA certificate obtained from IT
- [ ] Combined CA bundle created at `~/.ssl/combined-ca-bundle.pem`
- [ ] Environment variables set in `~/.zshrc` or `~/.bashrc`
- [ ] Virtual environment created and activated
- [ ] certifi installed in venv
- [ ] Combined bundle recreated with venv's certifi
- [ ] Dependencies installed successfully
- [ ] Test script passes
- [ ] Documentation updated

**Universal Pattern for All Python Apps:**

```python
import os

# Get CA bundle (works everywhere)
CA_BUNDLE = os.path.expanduser(os.getenv(
    "COMBINED_CA_BUNDLE",
    "~/.ssl/combined-ca-bundle.pem"
))

# Use in your code
# - requests: verify=CA_BUNDLE
# - httpx: verify=CA_BUNDLE
# - urllib3: ca_certs=CA_BUNDLE
# - API clients: verify_ssl=CA_BUNDLE
```

---

## Related Documentation

- [UV Corporate Proxy Setup](./uv-corporate-proxy-setup.md) - UV-specific configuration
- [Model Context Protocol](../genai/model-context-protocol.md) - Introduction to MCP
- [Modern Python Tooling](../shared/modern-python-tooling.md) - Ruff, Mypy, Pre-commit

## External Resources

- [Python SSL Documentation](https://docs.python.org/3/library/ssl.html)
- [Requests SSL Verification](https://requests.readthedocs.io/en/latest/user/advanced/#ssl-cert-verification)
- [Certifi Documentation](https://github.com/certifi/python-certifi)
- [Pip Proxy Configuration](https://pip.pypa.io/en/stable/user_guide/#using-a-proxy-server)

---

**Created:** 2026-02-06
**Updated:** 2026-04-06
**Tags:** #python #enterprise #ssl #certificates #proxy #devops #corporate #mcp #requests #pip
