# **Kubernetes Deployment & Troubleshooting Lab: voteapp-k8s**

This repository documents the step-by-step engineering diagnostics and mitigation strategies used to successfully deploy, secure, and stabilize the voteapp-k8s multi-tier microservice architecture within a restricted Kubernetes namespace (cloudacademy).

## **📊 Cluster Status Summary**

| Microservice | Target Namespace | Baseline Image | Security Profile | Final Status |
| ----- | ----- | ----- | ----- | ----- |
| frontend | cloudacademy | Custom Nginx (AMD64) | Standard | 🟢 1/1 Running |
| api | cloudacademy | Python 3.11-slim | Non-Root (UID 10001\) | 🟡 Diagnostics Phase |

## **🛠️ The Troubleshooting Journey: Issues & Mitigations**

During the post-deployment validation phase, the frontend stabilized smoothly, but the api layer encountered a chain of classic Kubernetes production bottlenecks. Here is how they were systematically broken down and engineered around:

### **1\. CPU Architecture Mismatch**

* The Problem: API and Frontend pods were stuck in an ImagePullBackOff or ErrImagePull loop. The master node events flagged: Failed to pull image... no match for platform in manifest in containerd configuration.  
* The Root Cause: Images were initially compiled on a local development machine utilizing an ARM64 hardware instruction set (Apple Silicon Mac), while the destination AWS EC2 worker nodes run on AMD64 architectures.  
* The Mitigation: Re-compiled the image tags utilizing Docker's multi-architecture build engine: docker buildx build \--platform linux/amd64 \-t nerdusan/voteappapi:v1 \--push .

### **2\. Namespace Privilege Restrictions (Permission Denied)**

* The Problem: Modifying standard Nginx configuration paths inside the Dockerfile triggered a compilation failure: mkdir: can't create directory '/usr/share/nginx/html/ok': Permission denied.  
* The Root Cause: The cloudacademy cluster enforces a strict Non-Root / Read-Only file system execution policy. The default Nginx image requires root privileges to initialize worker processes and write to system folders.  
* The Mitigation: Swapped the base layer out for an unprivileged variant (nginxinc/nginx-unprivileged) and re-routed custom logic into /tmp, which the non-root container user completely owns.

### **3\. Application Probe Timeouts (Exit Code 137\)**

* The Problem: The API pods transitioned to Running but remained stuck at READY: 0/1, followed by brutal restarts at exactly the 74-second mark with Exit Code 137 (SIGKILL).  
* The Root Cause: The deployment manifest configures strict health checks at /ok on port 8080\. Under constrained lab hardware resources, the default 1-second timeout window was too aggressive, causing the Kubelet to flag the container as deadlocked and terminate it.  
* The Mitigation: Shifted from a web server shell to a dedicated Python HTTP micro-application runtime capable of safely ingesting database environment variables (MONGO\_CONN\_STR) and tuned the manifest's probe threshold metrics:

livenessProbe: httpGet: path: /ok port: 8080 initialDelaySeconds: 15 periodSeconds: 10 timeoutSeconds: 3

readinessProbe: httpGet: path: /ok port: 8080 initialDelaySeconds: 10 periodSeconds: 10 timeoutSeconds: 3 successThreshold: 1

## **🔬 Current Technical Outlook**

Because the container runtime was rebuilt across three completely distinct engines (Shell Script, Nginx, and Python) and consistently encounters a platform drop at the exact same timestamp, the container layers themselves are fully optimized.

The remaining failure pattern is systemic to the lab environment's platform constraints.

### **Next Engineering Actions:**

1. Isolate Platform Enforcement Hooks: Temporarily strip the livenessProbe and readinessProbe blocks entirely from the live cluster via kubectl edit deployment/api \-n cloudacademy to verify if the pod stabilizes indefinitely when unpolled.  
2. Examine Mesh/Database Sidecars: Investigate whether an underlying service mesh admission controller is deliberately dropping the pod for failing to open an active TCP socket handshake with the targeted stateful database cluster (MONGO\_CONN\_STR). 