# Write-up

## Analyze, choose, and justify the appropriate resource option for deploying the app

The CMS app is a Python Flask web application. It uses Azure SQL Database for articles and users, Azure Blob Storage for images, and Microsoft Entra ID (MSAL) for sign-in. The app itself is stateless apart from the session cache, so it fits either a Virtual Machine (IaaS) or Azure App Service (PaaS). Both options are compared below.

### Option 1: Azure Virtual Machine (IaaS)

| Criterion | Analysis |
|---|---|
| **Costs** | Billed per hour for the VM whether or not it is busy. Additional costs include the OS disk, public IP, and network security. A Linux B2s-class VM costs roughly 30-40 USD/month. Hidden costs are the administrator time for OS patching, security hardening, Python/Nginx/Gunicorn setup, monitoring, and backups. A load balancer and a second VM are needed for redundancy, which doubles the compute cost. |
| **Scalability** | Vertical scaling (resizing the VM) requires a restart and downtime. Horizontal scaling requires a Virtual Machine Scale Set, a load balancer or Application Gateway, and a custom image or startup scripts so new instances have the app installed. Autoscale is possible but must be designed, configured, and maintained manually. |
| **Availability** | A single VM is a single point of failure (99.9% SLA only with premium disks). Higher availability requires two or more VMs in an Availability Set or across Availability Zones (up to 99.99%) behind a load balancer. The customer is responsible for OS updates, reboots, and recovery from failures. |
| **Workflow** | Full control over the OS, runtime, and installed software. Deployment is manual or scripted: SSH in, install Python, pip dependencies, the ODBC driver, and a web server/reverse proxy, then copy the code (for example with `git pull`, rsync, or a pipeline). CI/CD must be built by the team. Patching, TLS certificates, log rotation, and monitoring agents are all the team's responsibility. |

### Option 2: Azure App Service (PaaS)

| Criterion | Analysis |
|---|---|
| **Costs** | Billed per App Service Plan, not per app. A Free/F1 or Basic B1 plan (about 13 USD/month) is enough for a development or small CMS. Standard/Premium plans (roughly 70+ USD/month) add autoscale, deployment slots, and backups. The OS, runtime patching, load balancing, and TLS are included, so operational cost is low. Multiple apps can share one plan. |
| **Scalability** | Vertical scaling by changing the plan tier takes effect in seconds with no reimaging. Horizontal scaling (scale out) adds instances with a slider or autoscale rules based on CPU, memory, or schedule (Standard tier and above). The platform load-balances instances automatically. |
| **Availability** | Built-in SLA of 99.95% on Basic and higher tiers. The platform handles hardware failures, OS patching, and instance restarts. Multiple instances, health checks, and zone redundancy (Premium v3) improve availability. Deployment slots allow zero-downtime releases and rollback. |
| **Workflow** | Code is deployed directly from GitHub, Azure DevOps, ZIP deploy, or the VS Code/Azure CLI (`az webapp up`) with no server setup. The Python runtime and web server are managed by the platform. Configuration (SQL, Blob, and Entra ID secrets) is set as application settings or Key Vault references. Logging, metrics, and Application Insights are integrated. The team focuses on application code rather than infrastructure. |

### Decision: Azure App Service

I choose **Azure App Service**. The CMS is a standard, stateless Flask web app. Its data lives in Azure SQL and Blob Storage, and it needs no custom OS configuration, unusual drivers, or background daemons. App Service is cheaper to operate because the platform handles patching, load balancing, and TLS, so there is no VM administration overhead.

It also gives better scalability and availability with less effort. Scale up/out is a setting change, autoscale is built in, and the SLA is 99.95% on Basic and higher tiers without designing a load-balanced VM pair. The deployment workflow (Git/CI-CD, deployment slots, app settings for secrets) lets a small team ship changes quickly and safely, which matters more for a CMS used by staff than low-level infrastructure control.

## Assess app changes that would change your decision

### How the app changes per this decision

- **Configuration:** Secrets and connection details (SQL, Blob Storage, Entra ID, `SECRET_KEY`) are set as App Service application settings and read via `os.environ` in `config.py`.
- **Sessions:** `SESSION_TYPE = "filesystem"` uses local instance storage, so scaling out needs a shared store such as Azure Cache for Redis (or ARR affinity).
- **Networking and auth:** The Entra ID redirect URI (`/getAToken`) must point to the App Service HTTPS URL, and the SQL firewall must allow App Service.
- **Less control:** No root access or custom OS packages, accepted in exchange for managed patching, scaling, and availability.

### Other needs to change to suit the application requirements

- Enforce HTTPS-only and store secrets in Key Vault (with a managed identity where possible).
- Enable Application Insights for monitoring.
- Use a Standard tier or above for autoscale, deployment slots, and backups, with CI/CD deploying to a staging slot.

### Changes that would lead me to choose a Virtual Machine instead

I would switch to a VM if the app needed:

- Custom OS-level software or drivers that App Service does not support.
- Long-running background services or root/OS-level access.
- Strict compliance or network control over the host.
- Sustained high load where reserved VMs are cheaper and the team can administer them.

In that case, I would use at least two VMs in an Availability Set or Zones behind a load balancer, with automated configuration and a CI/CD pipeline.
