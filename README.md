# MCP Deployment on OpenShift Local

This project documents the deployment of Model Context Protocol (MCP) servers on OpenShift Local (CRC).

## What We Built

1. **ToolHive MCP Servers** - Deployed using the ToolHive Kubernetes Operator
2. **Google Workspace MCP Server** - Custom deployment for Gmail, Calendar, Drive, etc.
3. **ToolHive Studio** - Desktop UI for local MCP server management

---

## Prerequisites

All prerequisites were verified and available:

| Requirement | Version |
|-------------|---------|
| Helm | v3.19.0 |
| OpenShift CLI (`oc`) | v4.19.9 |
| OpenShift Local (CRC) | Server v4.19.8 |
| `cluster-admin` access | ✅ (logged in as `kubeadmin`) |
| Node.js/npx | ✅ (for MCP Inspector) |

---

## Part 1: ToolHive Operator Deployment

### Step 1: Create the Project
```bash
oc new-project toolhive-system
```

### Step 2: Clone ToolHive Repository
```bash
git clone https://github.com/stacklok/toolhive
cd toolhive
git checkout toolhive-operator-0.2.18
```

### Step 3: Install Operator CRDs
```bash
helm upgrade -i toolhive-operator-crds toolhive/deploy/charts/operator-crds
```

### Step 4: Install the Operator

We created a custom values file (`toolhive-values-crc.yaml`) with reduced resources for CRC's limited memory:

```bash
helm upgrade -i toolhive-operator toolhive/deploy/charts/operator \
  --values toolhive-values-crc.yaml
```

**Note:** The default values required too much memory. We reduced:
- Memory requests: 64Mi (from 192Mi)
- Memory limits: 128Mi (from 384Mi)

### Step 5: Deploy Example MCP Servers

**Fetch server (streamable-http transport):**
```bash
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_fetch.yaml
oc expose service mcp-fetch-proxy
```

**Yardstick server (stdio transport):**
```bash
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_yardstick_stdio.yaml
oc expose service mcp-yardstick-proxy
```

**Note:** Yardstick needed resource reduction via patch due to memory constraints:
```bash
oc patch mcpserver yardstick -n toolhive-system --type=merge \
  -p '{"spec":{"resources":{"requests":{"cpu":"5m","memory":"32Mi"},"limits":{"cpu":"50m","memory":"64Mi"}}}}'
```

---

## Part 2: Google Workspace MCP Server Deployment

### Step 1: Create Google OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project or select existing
3. Enable APIs: Gmail, Calendar, Drive, Docs, Sheets, etc.
4. Go to **APIs & Services** → **Credentials**
5. Create **OAuth 2.0 Client ID** → Choose **Desktop app** type
6. Download the JSON credentials file

**Important:** The credentials file is stored locally as `client_secret_*.json` and is in `.gitignore`.

### Step 2: Build Container Image in OpenShift

```bash
# Clone the repository
git clone https://github.com/taylorwilsdon/google_workspace_mcp.git

# Create build config
oc new-build --name=workspace-mcp --binary --strategy=docker -n toolhive-system

# Build from source
oc start-build workspace-mcp --from-dir=google_workspace_mcp --follow -n toolhive-system
```

### Step 3: Deploy the Server

We created `workspace-mcp-openshift.yaml` with:
- Kubernetes Secret for OAuth credentials
- Deployment with proper resource limits
- Service for internal access

```bash
oc apply -f workspace-mcp-openshift.yaml
```

**Key configuration:**
- Memory: 384Mi limit (Python app needs more than typical)
- Health probe delay: 120s initial (uv downloads packages on startup)
- Environment variables for OAuth and external URL

### Step 4: Expose the Route

```bash
oc expose service workspace-mcp -n toolhive-system
```

### Step 5: OAuth Authentication

The server uses Desktop OAuth flow. When calling tools:

1. Server returns an authorization URL
2. **Port-forward is required** for the OAuth callback:
   ```bash
   oc port-forward svc/workspace-mcp 8000:8000 -n toolhive-system
   ```
3. Open the authorization URL in browser
4. Sign in with Google account
5. Callback routes through port-forward to the pod
6. Retry the tool call

---

## Part 3: Testing with MCP Inspector

### Start the Inspector
```bash
npx @modelcontextprotocol/inspector
```

### Connect to ToolHive Fetch Server
- **Transport Type:** Streamable HTTP
- **URL:** `http://mcp-fetch-proxy-toolhive-system.apps-crc.testing/mcp`
- **Connection Type:** Via Proxy
- **Session Token:** (from terminal running inspector)

### Connect to Google Workspace MCP
- **Transport Type:** Streamable HTTP
- **URL:** `http://workspace-mcp-toolhive-system.apps-crc.testing/mcp`
- **Connection Type:** Via Proxy
- **Session Token:** (from terminal running inspector)

---

## Part 4: ToolHive Studio (Desktop UI)

Downloaded and installed ToolHive Studio v0.14.1 for local MCP server management:
- Download: `~/Downloads/ToolHive-arm64.dmg`
- Provides GUI for browsing, installing, and managing MCP servers locally

---

## Current Status

### Running Services

```bash
oc get pods -n toolhive-system
```

| Pod | Status | Purpose |
|-----|--------|---------|
| `toolhive-operator-*` | Running | Manages MCPServer CRDs |
| `workspace-mcp-*` | Running | Google Workspace MCP server |

### Available Routes

```bash
oc get routes -n toolhive-system
```

| Route | URL |
|-------|-----|
| workspace-mcp | `http://workspace-mcp-toolhive-system.apps-crc.testing` |

### Scaled Down (to free memory)

The following were scaled down due to CRC memory constraints:
- fetch MCP server
- yardstick MCP server

To restore them:
```bash
# Re-create fetch
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_fetch.yaml
oc expose service mcp-fetch-proxy

# Re-create yardstick  
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_yardstick_stdio.yaml
oc expose service mcp-yardstick-proxy
```

---

## Picking Up Where You Left Off

### 1. Start OpenShift Local
```bash
crc start
```

### 2. Login to OpenShift
```bash
oc login -u kubeadmin -p <password> https://api.crc.testing:6443
```

### 3. Check Status
```bash
oc get pods -n toolhive-system
oc get routes -n toolhive-system
```

### 4. Test Google Workspace MCP

Start port-forward (needed for OAuth):
```bash
oc port-forward svc/workspace-mcp 8000:8000 -n toolhive-system &
```

Start MCP Inspector:
```bash
npx @modelcontextprotocol/inspector
```

Connect to: `http://workspace-mcp-toolhive-system.apps-crc.testing/mcp`

### 5. View Logs
```bash
oc logs -f deployment/workspace-mcp -n toolhive-system
```

---

## Files in This Project

| File | Purpose | Git Status |
|------|---------|------------|
| `.design/procedure.md` | Original ToolHive deployment guide | ✅ Track |
| `README.md` | This file | ✅ Track |
| `.gitignore` | Protects sensitive files | ✅ Track |
| `client_secret_*.json` | Google OAuth credentials | 🚫 Ignored |
| `workspace-mcp-*.yaml` | Deployment files with secrets | 🚫 Ignored |
| `*-values*.yaml` | Helm values with secrets | 🚫 Ignored |
| `toolhive/` | Cloned ToolHive repo | 🚫 Ignored |
| `google_workspace_mcp/` | Cloned Google Workspace MCP repo | 🚫 Ignored |
| `toolhive-studio/` | Cloned ToolHive Studio repo | 🚫 Ignored |

---

## Troubleshooting

### Memory Issues (Pods Pending)
CRC has limited memory (~10GB). Solutions:
1. Scale down unused deployments
2. Reduce resource requests/limits
3. Check: `oc describe pod <pod-name> -n toolhive-system`

### OAuth Callback Fails
Ensure port-forward is running:
```bash
oc port-forward svc/workspace-mcp 8000:8000 -n toolhive-system
```

### Pod OOMKilled
Increase memory limit in deployment:
```yaml
resources:
  limits:
    memory: 384Mi  # or higher
```

### Health Probe Failures
Increase initial delay for Python apps that download packages:
```yaml
livenessProbe:
  initialDelaySeconds: 120
```

---

## References

- [ToolHive Documentation](https://docs.toolhive.dev/)
- [ToolHive GitHub](https://github.com/stacklok/toolhive)
- [Google Workspace MCP](https://github.com/taylorwilsdon/google_workspace_mcp)
- [MCP Protocol](https://modelcontextprotocol.io/)
- [Red Hat Article: Deploy MCP on OpenShift](https://developers.redhat.com/articles/2025/10/01/how-deploy-mcp-servers-openshift-using-toolhive)

