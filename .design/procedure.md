# How to Deploy MCP Servers on OpenShift Using ToolHive

> **Source:** [Red Hat Developers](https://developers.redhat.com/articles/2025/10/01/how-deploy-mcp-servers-openshift-using-toolhive)
>
> **Published:** October 1, 2025
>
> **Authors:** Roddie Kieley, Jakub Hrozek

---

## Overview

The Model Context Protocol (MCP) was introduced by Anthropic in November 2024 to standardize how AI systems interact with large language models (LLMs). It has since been described as being similar to USB-C, allowing one thing to connect to another via the same interface and protocol.

Since it was introduced, many developers have "kicked the tires" and found genuine utility in what it offers. As a result, the open source community has seen rapid adoption and growth. This has resulted in an explosion of available MCP servers that developers can quickly and easily obtain and run locally for development and testing. More recently, both Anthropic and the wider MCP community have become focused not just on utility and function but also on deployment, orchestration, and security. A recent protocol specification update in June of this year addresses that well accepted gap.

---

## From the Developer to the Enterprise: The ToolHive Project

While the number of MCP servers available continues to grow, efforts to allow developers to utilize MCP servers locally as well as be able to deploy and manage them. One such project, **ToolHive**, from the team at Stacklok, aims to bridge these two areas by helping developers run MCP servers both locally and in Kubernetes.

Getting up and running with Kubernetes is not always straightforward, but the project includes kind-based examples in the repository and Helm charts for deployment on a Kubernetes cluster. For the local developer, a GUI integrates MCP servers into common AI developer tools like Cursor and Claude Code. For Kubernetes and Red Hat OpenShift, there's an operator that controls an `MCPServer` custom resource definition (CRD).

While Fedora is my daily driver, my Kubernetes distribution of choice is OKD (Origin Kubernetes Distribution), which is based on CentOS and forms the basis of OpenShift. That said, these instructions should also work equally well against Red Hat OpenShift Local.

---

## The ToolHive Operator on OpenShift

### Prerequisites

Before you begin, make sure you have the following:

- Helm installed locally
- Access to an OpenShift-based Kubernetes cluster (either OKD or OpenShift Local)
- OpenShift CLI (`oc`) installed locally
- `cluster-admin` access is required because you will need to install CRDs for the operator
- MCP Inspector availability (either from source or package)

### Installation TL;DR

Follow these steps for a quick installation:

```bash
# 1. Login as kubeadmin (command as provided via the OKD web console)
oc login

# 2. Create the toolhive-system project
oc new-project toolhive-system

# 3. Clone the ToolHive repository
git clone https://github.com/stacklok/toolhive

# 4. Checkout the operator release
git checkout toolhive-operator-0.2.18

# 5. Install the operator CRDs
helm upgrade -i toolhive-operator-crds toolhive/deploy/charts/operator-crds

# 6. Install the operator with OpenShift values
helm upgrade -i toolhive-operator toolhive/deploy/charts/operator --values toolhive/deploy/charts/operator/values-openshift.yaml

# 7. Create the fetch MCP server
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_fetch.yaml

# 8. Expose the fetch service
oc expose service mcp-fetch-proxy

# 9. Create the yardstick MCP server
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_yardstick_stdio.yaml

# 10. Expose the yardstick service
oc expose service mcp-yardstick-proxy

# 11. Run the MCP Inspector
npx @modelcontextprotocol/inspector
```

---

## ToolHive Operator

The ToolHive operator implements the well-known controller pattern where the operator watches and reconciles a custom resource definition, specifically the `MCPServer` CRD. This CRD is defined to enable the quick and easy deployment of MCP servers that don't require much configuration. It also exposes the full Kubernetes `PodTemplateSpec` structure to enable more complex use cases if needed.

In simpler cases, you need to specify things like the MCP transport type and the port we want to expose the MCP server Service on. For more complex use cases, you can customize the Pod's environment or pass required data via Volumes, for example.

---

## Example MCP Servers

Two examples are presented:

1. **Yardstick MCP server** - A simple echo server that returns a requested string while conforming to the Model Context Protocol
2. **gofetch MCP server** - Retrieves data from a URL and returns it

While each of these are basic MCP servers, they conform to the protocol and, importantly, utilize different transports:
- **yardstick** uses the `stdio` transport, which local developers might be most familiar with
- **gofetch** uses the `streamable-http` transport

> **Note:** HTTP SSE (server-sent events) is also supported but is deprecated from the Model Context Protocol point of view.

### Yardstick

Custom resource YAML:

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: yardstick
  namespace: toolhive-system
spec:
  image: ghcr.io/stackloklabs/yardstick/yardstick-server:0.0.2
  transport: stdio
  port: 8080
  permissionProfile:
    type: builtin
    name: network
  resources:
    limits:
      cpu: "100m"
      memory: "128Mi"
    requests:
      cpu: "50m"
      memory: "64Mi"
```

### gofetch

Custom resource YAML:

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: fetch
  namespace: toolhive-system
spec:
  image: ghcr.io/stackloklabs/gofetch/server
  transport: streamable-http
  port: 8080
  targetPort: 8080
  permissionProfile:
    type: builtin
    name: network
  resources:
    limits:
      cpu: "100m"
      memory: "128Mi"
    requests:
      cpu: "50m"
      memory: "64Mi"
```

---

## Stdio Proxy Architecture

The ToolHive operator creates a sidecar proxy that converts between HTTP/SSE and `stdio` transports. This enables MCP clients to communicate with `stdio`-based MCP servers over the network.

The HTTP proxy handles the protocol conversion between HTTP requests and JSON-RPC messages in both directions, enabling HTTP-based MCP clients to seamlessly interact with `stdio`-based MCP servers running in Kubernetes.

### Yardstick Deployment

The example is included in the ToolHive Git repository:

```bash
oc create -f toolhive/examples/operator/mcp-servers/mcpserver_yardstick_stdio.yaml
```

After creation, you can check to ensure the pods have been successfully created. Note two `yardstick` pods instead of one: one is the proxy, and the other is the MCP server itself.

```bash
oc get pods
```

Example output:

```
NAME                                 READY   STATUS    RESTARTS   AGE
toolhive-operator-85b7965d6c-jz7xp   1/1     Running   0          16m
yardstick-0                          1/1     Running   0          2m
yardstick-6b9f44f4f5-rv9ww           1/1     Running   0          2m
```

While the `yardstick` MCP server uses the `stdio` transport, that is transparent to the end user in this case. You can expose a Route to the Service:

```bash
oc expose service mcp-yardstick-proxy
```

Output:

```
route.route.openshift.io/mcp-yardstick-proxy exposed
```

Check the route:

```bash
oc get route
```

Example output:

```
NAME                  HOST/PORT                                                PATH   SERVICES              PORT   TERMINATION   WILDCARD
mcp-yardstick-proxy   mcp-yardstick-proxy-toolhive-system.apps.okd.kieley.io          mcp-yardstick-proxy   http                 None
```

Note the HOST/PORT combination. The `stdio` traffic is proxied via the SSE endpoint.

---

## Conclusion

In this article, you deployed CRDs and an operator on OpenShift via Helm charts and container images from the ToolHive project that can instantiate and proxy MCP servers. This `toolhive` operator reconciled the `MCPServer` custom resource definition instances, of which you deployed two: yardstick, and gofetch.

- The **gofetch** server was the more network-native MCP server, using a `streamable-http` transport
- The **yardstick** MCP server was a more traditional server with a `stdio` transport, which is more familiar on the desktop

In each case, the MCP server was correctly proxied via the `proxyrunner` instance, allowing you to connect via an OpenShift Route from "outside" the cluster.

ToolHive and its operator provide powerful capabilities for deploying MCP servers on OpenShift:
- Proxying `stdio` via the network
- Supporting authentication and authorization
- Adding telemetry

ToolHive can operationalize your favorite MCP servers for production use cases.

---

## References and Background Information

- [OpenShift Local Getting Started](https://developers.redhat.com/products/openshift-local/getting-started)
- [OKD: Origin Kubernetes Distribution](https://www.okd.io/)
- [Using OpenShift Local or OKD](https://developers.redhat.com/articles/2024/04/18/using-openshift-local-or-okd)
- [OpenShift Local releases](https://github.com/crc-org/crc/releases)
- [OpenShift Security Constraints documentation](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html)
- [HAProxy](https://www.haproxy.org/)
- [OpenShift Documentation Configuring Routes](https://docs.openshift.com/container-platform/latest/networking/routes/route-configuration.html)
- [Red Hat Console for OpenShift Local](https://console.redhat.com/openshift/create/local)
- [Red Hat Developer Subscriptions for Individuals](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux)
- [Installing OpenShift on a single node](https://docs.openshift.com/container-platform/latest/installing/installing_sno/install-sno-installing-sno.html)
- [Helm - The package manager for Kubernetes](https://helm.sh/)
- [Operator Framework FAQ](https://operatorframework.io/faq/)
- [MCP - Model Context Protocol](https://modelcontextprotocol.io/)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- [ToolHive GitHub repository](https://github.com/stacklok/toolhive)
- [ToolHive documentation](https://docs.toolhive.dev/)
- [ToolHive MCPServer CRD specification](https://docs.toolhive.dev/docs/kubernetes/mcpserver-crd/)
- [ToolHive Operator CRDs Helm Chart](https://github.com/stacklok/toolhive/tree/main/deploy/charts/operator-crds)
- [ToolHive Operator Helm Chart](https://github.com/stacklok/toolhive/tree/main/deploy/charts/operator)
- [Quickstart: ToolHive Kubernetes Operator](https://docs.toolhive.dev/docs/kubernetes/quickstart/)
- [ToolHive Helm Chart OpenShift Values](https://github.com/stacklok/toolhive/blob/main/deploy/charts/operator/values-openshift.yaml)
- [Fetch MCPServer CRD instance example](https://github.com/stacklok/toolhive/blob/main/examples/operator/mcp-servers/mcpserver_fetch.yaml)
- [Yardstick MCPServer CRD instance example](https://github.com/stacklok/toolhive/blob/main/examples/operator/mcp-servers/mcpserver_yardstick_stdio.yaml)

