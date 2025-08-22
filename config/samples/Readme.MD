# NiFi on Kubernetes with Ingress and Path Normalization

This repository contains Kubernetes manifests for deploying **Apache NiFi** using **NiFiKop** behind an **NGINX Ingress Controller**, with an intermediate **NGINX proxy layer** for path normalization.

---

## 📂 Repository Contents

- **Cluster.yaml** → NiFiKop `NifiCluster` CR definition  
- **Deployment.yaml** → Deployment for the intermediate NGINX proxy  
- **ConfigMap.yaml** → NGINX proxy configuration (`default.conf`)  
- **Ingress.yaml** → Ingress resource exposing NiFi via NGINX  

---

## 🏗️ Architecture Overview

[ Browser ]
│ https://myserver.example.com/nifidev/
▼
[ Ingress Controller ]
│ Host: myserver.example.com
▼
[ NGINX Proxy Pod (Path Normalization Layer) ]
│ Rewrites /nifidev/* → /*
│ Adds proxy headers (X-ProxyHost, X-ProxyScheme, Host, etc.)
▼
[ NiFi Cluster Service (NiFiKop) ]
│ Accepts requests at root /
▼
[ NiFi Web UI, API, Custom UIs, Viewers ]

---

## 🔑 Key Concepts

### 1. **Ingress**
- Routes requests for `https://myserver.example.com/nifidev/`  
- Forwards traffic to the intermediate NGINX proxy pod  
- Uses TLS termination at the Ingress  

### 2. **Path Normalization Layer (NGINX Proxy Pod)**
- Ensures NiFi always receives traffic **at the root path (`/`)**  
- Rewrites `/nifidev/… → /…` before forwarding to NiFi  
- Injects proxy headers required by NiFi:
  - `X-ProxyScheme`
  - `X-ProxyHost`
  - `X-ProxyPort`
  - `Host`
- Terminates TLS (optional, can run HTTPS → HTTPS)  

### 3. **NiFi (via NiFiKop)**
- Exposed only within the cluster on HTTPS  
- Configured with `nifi.web.proxy.host=myserver.example.com`  
- Expects requests on `/` (root), not on `/nifidev` or subpaths  

---

## ✅ Why Path Normalization?

NiFi is made up of multiple web applications (UI, API, custom UIs, viewers, etc.).  
If NiFi is only mapped at `/nifi`, features like **UpdateAttribute UI** won’t work because they’re served at `/update-attribute-ui-<version>`. 
If you want to use a context path and if your Cluster is behind a Reverse Proxy, you can access nifi through your reverse proxy at a context path of your choice. This is useful especially if you are using Internal Ingress on Kubernetes which usually is the case for Private Clusters on Cloud.  

The **path normalization layer** solves this by:
- Allowing users to access NiFi at `https://myserver.example.com/nifidev/`  
- Ensuring NiFi still sees requests at `/` internally  

---

## 🔧 How to Deploy

1. Apply the manifests in order:

```bash
kubectl apply -f Cluster.yaml
kubectl apply -f ConfigMap.yaml
kubectl apply -f Deployment.yaml
kubectl apply -f Ingress.yaml
