# 🚀 Deploy 3-Tier Application on OpenShift  
**Repository:** `deploy-tier-application-backend-Database-proxy`  
**Platform:** Red Hat OpenShift (CRC Local Environment)

---

## 🧩 Project Overview
This repository demonstrates how to deploy a **3-Tier Web Application** on **Red Hat OpenShift** using declarative YAML manifests.

**Architecture:**
- **Database Layer:** MySQL  
- **Application Layer:** Golang backend API  
- **Proxy Layer:** Nginx reverse proxy  
- **Routing Layer:** OpenShift Route (public exposure)

All manifests are now organized under the `/OpenShift` directory.

---

## ⚙️ Components
| Layer | File(s) | Description |
|--------|----------|-------------|
| **Database** | `database_deployment.yaml`, `db_service.yaml`, `db-secret.yaml` | MySQL Deployment, Service, and Secret definition |
| **Backend** | `backend_deployment.yaml`, `backend_service.yaml` | Golang API that connects to MySQL securely through secrets |
| **Proxy** | `proxy_deployment.yaml`, `proxy_nodeport.yaml` | Nginx reverse proxy that forwards external traffic to the backend |
| **Routing** | `proxy_route.yaml` | OpenShift Route exposing the proxy over HTTPS |

---

## 🛠️ Deployment Steps on OpenShift
### 1. Create a dedicated project
```bash
oc new-project webapp
```
2. Apply all OpenShift manifests
```bash
oc apply -f OpenShift/
```
3. Grant necessary permissions
```bash
oc adm policy add-scc-to-user anyuid -z default -n webapp
oc get pods -n webapp
##Expected
backend-deployment   1/1   Running
database-deployment  1/1   Running
proxy-deployment     1/1   Running
5. Expose the proxy publicly
oc apply -f OpenShift/proxy_route.yaml

🔐 Secrets and Configuration

Database credentials are stored as a Kubernetes Secret:

oc create secret generic db-secret --from-literal=db-password=admin123 -n webapp

🌐 Accessing the Application

Once deployed, OpenShift automatically generates a public route.
oc get route -n webapp

Example output:

NAME           HOST/PORT                    SERVICES   PORT   TERMINATION   WILDCARD
webapp-route   webapp.apps-crc.testing      proxy      443    passthrough   None

You can then access the app directly via:
curl -k https://webapp.apps-crc.testing

 ["Blog post #0","Blog post #1","Blog post #2","Blog post #3","Blog post #4"]

```

🧰 Troubleshooting Guide

🔴 1. MySQL pod stuck in ContainerCreating
```bash
Error:

driver name kubevirt.io.hostpath-provisioner not found

```
Cause: HostPath provisioner not registered in CRC.
Fix: Replace the PersistentVolume with emptyDir in database_deployment.yaml.


🔴 2. MySQL permission denied
```bash
Error:

Can't create/write to file '/var/lib/mysql/is_writable' (OS errno 13)

```
Fix:
Add this inside the container spec:
```bash
securityContext:
  runAsUser: 0
  fsGroup: 0
```
🔴 3. Backend CrashLoopBackOff — cannot read secret
```bash
Error:

open /run/secrets/db-password: no such file or directory
```
Cause: /run/secrets is read-only in OpenShift.
Fix: Mount secret to a custom path /app/db-password and point environment variable:
```bash
env:
- name: DB_PASSWORD_FILE
  value: /app/db-password
```
### 🔴 5. **CRC VM stops automatically / OpenShift becomes unresponsive**

**Symptom:**  
The CRC virtual machine suddenly powers off or becomes unresponsive even though no `crc stop` command was executed.  
When checking system logs (`journalctl` or `/var/log/syslog`), you can see messages like:

oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),task=qemu-system-x86,pid=xxxx,uid=64055
libvirtd[1031]: End of file while reading data: Input/output error


**Cause:**  
This happens because the CRC VM consumes all available host memory.  
When the system runs out of RAM and has little or no swap space, the Linux Out-Of-Memory (OOM) killer terminates the CRC process to free memory.  
That’s why the CRC VM shuts down automatically.

**Fix:**  

1. **Increase CRC resources**
   ```bash
   sudo fallocate -l 8G /swapfile
	sudo chmod 600 /swapfile
	sudo mkswap /swapfile
	sudo swapon /swapfile
	crc stop
	crc start
   ```
Adding swap space provides the host OS with a “safety buffer” when RAM usage peaks.
Instead of killing the CRC process, the system temporarily moves inactive memory pages to the swap file, preventing the OOM condition and keeping the OpenShift cluster stable.
