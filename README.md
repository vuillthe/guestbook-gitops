# Guestbook GitOps Monorepo

This repository contains both the Python guestbook application source code and its GitOps deployment configuration using Kustomize and ArgoCD.

## Structure

- `src/` - Python guestbook application source code
  - `app.py` - Main Flask application
  - `Dockerfile.alpine` - Container build configuration
  - `requirements.txt` - Python dependencies
  - `static/` - Static files (CSS, JS, images)
- `apps/guestbook/base/` - Kustomize configuration for deployment
  - `guestbook-*` - Main application manifests (deployment, service, configmap)
  - `nginx-*` - Static file server manifests (deployment, service, configmap)  
  - `ingress.yaml` - Traefik ingress configuration (shared)
  - `cert-issuer.yaml` - SSL certificate issuer configuration
  - `kustomization.yaml` - Kustomize orchestration file
- `argocd/` - ArgoCD Application definitions (includes PostgreSQL Helm chart)
- `.github/workflows/` - GitHub Actions for automated CI/CD

## Deployment Flow

1. **Code Change**: Developer pushes changes to `src/` folder
2. **Build Trigger**: GitHub Actions detects changes in `src/` and starts build
3. **Docker Build**: Application is containerized using `src/Dockerfile.alpine`
4. **Image Push**: Docker image is pushed to GitHub Container Registry
5. **GitOps Update**: Same workflow updates the image tag in `kustomization.yaml`
6. **ArgoCD Sync**: ArgoCD detects GitOps changes and deploys to Kubernetes
7. **Database**: PostgreSQL is deployed as a separate Helm chart via ArgoCD

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    guestbook namespace                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  🐍 App     │    │  🌐 Nginx   │    │  🐘 PostgreSQL │  │
│  │ guestbook   │    │ Static      │    │ Database    │     │
│  │ deployment  │    │ Server      │    │ (Helm)      │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                   │                   │          │
│         │                   │                   │          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ guestbook   │    │ nginx-static│    │ postgresql  │     │
│  │ service     │    │ service     │    │ service     │     │
│  │ :80         │    │ :80         │    │ :5432       │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                   │                              │
│         └───────────────────┴──────────┐                  │
│                                        │                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            🌍 Traefik Ingress                       │    │
│  │                                                     │    │
│  │  Routes:                                            │    │
│  │  • /        → guestbook-service (Python app)       │    │
│  │  • /static  → nginx-static-service (Static files)  │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
└──────────────────────────────┼──────────────────────────────┘
                               │
┌──────────────────────────────┼──────────────────────────────┐
│                   🔒 SSL/TLS │                              │
│                              │                              │
│  ┌─────────────┐             │    ┌─────────────┐           │
│  │ Let's       │             │    │ Certificate │           │
│  │ Encrypt     │             │    │ guestbook-  │           │
│  │ Issuer      │             │    │ tls-cert    │           │
│  └─────────────┘             │    └─────────────┘           │
└──────────────────────────────┼──────────────────────────────┘
                               │
                    ☁️ Internet (HTTPS)
                    gb-heg.duckdns.org
```

## 📦 Detailed Component Structure

### **🐍 Guestbook Application**
```
guestbook-deployment:
├── 📦 Container: guestbook
│   ├── 🔌 Port: 5000
│   ├── 🌐 Environment Variables:
│   │   ├── DB_HOST=postgresql
│   │   ├── DB_PORT=5432
│   │   ├── DB_NAME=guestbook
│   │   ├── DB_USER=guestbook
│   │   ├── DB_PASS=<from postgresql secret>
│   │   └── GUESTBOOK_SERVICE=postgres
│   ├── 🏥 Health Checks:
│   │   ├── Startup Probe: /health (10s delay, 5s interval, 6 failures = 30s max)
│   │   ├── Liveness Probe: /health (30s delay, 10s interval)
│   │   └── Readiness Probe: /ready (5s delay, 5s interval)
│   └── 🎯 Endpoints:
│       ├── /health - Application liveness check
│       └── /ready - Database connectivity + readiness
├── 🔄 Replicas: 2
└── 📊 Resources: 100m CPU, 128Mi-256Mi RAM
```

### **🌐 Nginx Static Server**
```
nginx-static-deployment:
├── 🔄 Init Container (static-extractor):
│   ├── 📦 Image: same as guestbook
│   ├── 📁 Copies: /app/static/* → /shared/static/
│   └── 💾 Volume: static-volume (emptyDir)
├── 📦 Main Container (nginx):
│   ├── 📦 Image: nginx:1.25-alpine
│   ├── 🔌 Port: 80
│   ├── 📁 Serves: /usr/share/nginx/html/static
│   ├── ⚙️ Config: nginx-config ConfigMap
│   └── 🏥 Health Check: /health
├── 🔄 Replicas: 2
└── 📊 Resources: 50m CPU, 64Mi-128Mi RAM
```

### **🐘 PostgreSQL Database**
```
postgresql (Helm Chart):
├── 📊 Chart: bitnami/postgresql v12.12.10
├── 🗄️ Database: guestbook
├── 👤 User: guestbook
├── 🔐 Password: guestbook123 (in secret)
├── 💾 Storage: 2Gi PVC
├── 🔌 Port: 5432
├── 🔄 Replicas: 1 (primary)
└── 📊 Resources: 250m CPU, 256Mi-512Mi RAM
```

### **🔒 SSL/TLS Configuration**
```
cert-manager setup:
├── 🏢 Issuer: letsencrypt-guestbook-issuer
│   ├── 📧 Email: your-email@example.com
│   ├── 🔗 ACME Server: Let's Encrypt v2
│   └── 🛡️ Challenge: HTTP-01 via Traefik
├── 📜 Certificate: guestbook-tls-cert
│   ├── 🌐 Domain: gb-heg.duckdns.org
│   ├── 🔄 Auto-renewal: Yes
│   └── 🔐 Secret: guestbook-tls-cert
└── 🌍 Ingress: TLS termination at Traefik
```

## 🔄 Data Flow Explanation

### **📥 Request Flow (User → App)**
```
1. 🌐 User visits https://gb-heg.duckdns.org/
2. 🔒 DNS resolves to Traefik LoadBalancer
3. 🔐 Traefik terminates SSL using guestbook-tls-cert
4. 🛣️ Traefik routes based on path:
   ├── /static/* → nginx-static-service:80
   └── /*        → guestbook-service:80
5. 🐍 Python app processes request
6. 🐘 App queries PostgreSQL if needed
7. 📤 Response sent back to user
```

### **📦 Static Files Flow**
```
1. 🔄 Pod starts with init container
2. 📦 Init container uses guestbook image
3. 📁 Copies /app/static/* to shared volume
4. ✅ Init container completes
5. 🌐 Nginx container starts
6. 📂 Mounts shared volume at /usr/share/nginx/html/static
7. 🚀 Nginx serves files with caching headers
```

### **💾 Database Connection Flow**
```
1. 🐍 Guestbook app starts
2. 🔍 Reads environment variables
3. 🐘 Connects to postgresql:5432
4. 🔐 Uses credentials from postgresql secret
5. 🗄️ Creates/uses guestbook database
6. 💬 Stores guestbook entries
```

## Environment Variables

The guestbook application uses these environment variables to connect to PostgreSQL:

- `DB_HOST` - PostgreSQL service hostname (`postgresql`)
- `DB_PORT` - PostgreSQL port (`5432`)
- `DB_NAME` - Database name (`guestbook`)
- `DB_USER` - Database username (`guestbook`)
- `DB_PASS` - Database password (from PostgreSQL secret)
- `GUESTBOOK_SERVICE` - Backend service type (`postgres`)

## 🎓 GitOps Learning Concepts

### **What is GitOps?**
GitOps is a deployment methodology that uses Git as the single source of truth for infrastructure and application configuration. Key principles:

1. **📝 Declarative**: Everything is described declaratively (YAML manifests)
2. **🔄 Versioned**: All changes are tracked in Git history
3. **🚀 Automated**: Deployments happen automatically when Git changes
4. **🏥 Observable**: Current state vs desired state is always visible

### **ArgoCD Workflow**
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   📱 Developer   │    │  🏭 GitHub      │    │  ☁️ Kubernetes   │
│   Source Code   │    │  Actions        │    │   Cluster       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │ 1. git push src/      │                       │
         ▼                       │                       │
┌─────────────────┐              │                       │
│  📚 Monorepo    │              │                       │
│  ├── src/       │              │                       │
│  ├── apps/      │              │                       │
│  └── argocd/    │              │                       │
└─────────────────┘              │                       │
         │                       │                       │
         │ 2. trigger on src/    │                       │
         ▼                       │                       │
         ├──────────────────────▶│                       │
                                 │ 3. build & push       │
                                 ▼                       │
                        ┌─────────────────┐              │
                        │  📦 GitHub      │              │
                        │  Container      │              │
                        │  Registry       │              │
                        └─────────────────┘              │
                                 │                       │
                                 │ 4. update same repo   │
                                 ▼                       │
                        ┌─────────────────┐              │
                        │  📝 GitOps      │              │
                        │  Config         │              │
                        │  (same repo)    │              │
                        └─────────────────┘              │
                                 │                       │
                                 │ 5. ArgoCD sync        │
                                 ▼                       │
                        ┌─────────────────┐              │
                        │  🔄 ArgoCD      │              │
                        │  Controller     │              │
                        └─────────────────┘              │
                                 │                       │
                                 │ 6. deploy             │
                                 ▼                       │
                                 ├──────────────────────▶│
                                                         │
                                                ┌─────────────────┐
                                                │  🐍 Guestbook   │
                                                │  🌐 Nginx       │
                                                │  🐘 PostgreSQL  │
                                                └─────────────────┘
```

### **Why This Architecture?**

#### **🔄 Separation of Concerns**
- **Static Files**: Served by nginx (fast, cached, CDN-ready)
- **Dynamic Content**: Handled by Python app (business logic)
- **Data Storage**: PostgreSQL (ACID compliance, relationships)

#### **📈 Scalability Benefits**
- **Independent Scaling**: Each component scales based on load
- **Resource Optimization**: nginx uses less resources than Python for static files
- **Caching Strategy**: Static files cached at nginx level

#### **🔐 Security Advantages**
- **Namespace Isolation**: All components in dedicated namespace
- **Secret Management**: Database credentials managed by Kubernetes
- **TLS Termination**: SSL handled at ingress level
- **Least Privilege**: Each component has minimal required permissions

## Setup

### Prerequisites
- ArgoCD installed and running
- cert-manager installed for SSL/TLS certificates
- Traefik ingress controller
- DNS record pointing `gb-heg.duckdns.org` to your cluster

### Deployment Steps

1. **Update Email in cert-issuer.yaml**:
   ```bash
   # Edit apps/guestbook/base/cert-issuer.yaml
   # Change: email: your-email@example.com
   ```

2. **Update Repository Name in kustomization.yaml**:
   ```bash
   # Edit apps/guestbook/base/kustomization.yaml
   # Change: ghcr.io/vuillthe/guestbook-gitops
   ```

3. **Enable GitHub Actions**:
   ```bash
   # Make sure your repository has Actions enabled
   # The workflow will automatically trigger on pushes to src/
   ```

4. **Apply ArgoCD applications**:
   ```bash
   kubectl apply -f argocd/app-of-apps.yaml
   ```

5. **Verify SSL certificate**:
   ```bash
   kubectl get certificate -n guestbook
   kubectl describe certificate guestbook-tls-cert -n guestbook
   ```

ArgoCD will automatically deploy PostgreSQL (via Helm), the guestbook application, and nginx for static files, all in the `guestbook` namespace.

## SSL/TLS Configuration

The setup includes automatic SSL/TLS certificates via Let's Encrypt:

- **Domain**: `gb-heg.duckdns.org`
- **Issuer**: Let's Encrypt ACME HTTP-01 challenge
- **Certificate**: Automatically managed by cert-manager
- **Ingress**: Routes `/static` to nginx, everything else to guestbook app

## PostgreSQL Configuration

PostgreSQL is deployed directly as a Helm chart with these settings:
- **Chart**: `bitnami/postgresql` version `12.12.10`
- **Database**: `guestbook`
- **Username**: `guestbook`
- **Password**: `guestbook` (stored in auto-generated secret)
- **Storage**: 2Gi persistent volume

## Image Updates

Image tags are automatically updated by GitHub Actions from the source repository. The workflow updates the `apps/guestbook/base/kustomization.yaml` file with new image tags based on commit hashes.

### File Organization
The base directory uses prefixed filenames for clarity:
- **`guestbook-*`**: Main Python application resources
- **`nginx-*`**: Static file server resources  
- **`cert-issuer.yaml`**: SSL certificate management
- **`kustomization.yaml`**: Kustomize orchestration

## Security Notes

- Database credentials are managed by the PostgreSQL Helm chart
- The guestbook app reads the password from the `postgresql` secret created by Helm
- All secrets are base64 encoded and stored in Kubernetes

## 🚀 Deployment Commands

```bash
# Deploy everything
kubectl apply -f argocd/app-of-apps.yaml

# Check certificate status
kubectl get certificate guestbook-tls-cert -n guestbook
kubectl describe certificate guestbook-tls-cert -n guestbook

# Check issuer status  
kubectl get issuer letsencrypt-guestbook-issuer -n guestbook

# Check all guestbook resources
kubectl get all -n guestbook
```

## 🔍 Troubleshooting SSL

If the certificate doesn't get issued:
```bash
# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager

# Check certificate request
kubectl get certificaterequests -n guestbook
kubectl describe certificaterequest <name> -n guestbook

# Check challenges
kubectl get challenges -n guestbook
```

## 🎯 Learning Exercises

### **Exercise 1: Understanding Resource Dependencies**
```bash
# 1. Check what ArgoCD creates first
kubectl get applications -n argocd

# 2. Watch resources being created
kubectl get events -n guestbook --sort-by='.lastTimestamp'

# 3. Understand service discovery
kubectl exec -n guestbook deployment/guestbook -- nslookup postgresql
```

### **Exercise 2: Scaling and Load Testing**
```bash
# Scale nginx independently
kubectl scale deployment nginx-static -n guestbook --replicas=3

# Scale guestbook app
kubectl scale deployment guestbook -n guestbook --replicas=4

# Watch resource usage
kubectl top pods -n guestbook
```

### **Exercise 3: SSL Certificate Lifecycle**
```bash
# Check certificate details
kubectl get certificate guestbook-tls-cert -n guestbook -o yaml

# Force certificate renewal (for testing)
kubectl delete certificate guestbook-tls-cert -n guestbook
# ArgoCD will recreate it automatically

# Check certificate expiry
kubectl describe certificate guestbook-tls-cert -n guestbook
```

### **Exercise 4: GitOps Workflow Testing**
```bash
# 1. Make a change to image tag in kustomization.yaml
# 2. Commit and push to Git
# 3. Watch ArgoCD detect and sync the change
kubectl get applications -n argocd -w

# Check sync status
kubectl describe application guestbook -n argocd
```

## 📚 Additional Resources

### **Kubernetes Concepts Used**
- **Deployments**: Rolling updates, replica management
- **Services**: Service discovery, load balancing
- **Ingress**: HTTP routing, TLS termination
- **ConfigMaps**: Configuration management
- **Secrets**: Sensitive data storage
- **Namespaces**: Resource isolation
- **Init Containers**: Setup tasks before main container

### **Tools and Technologies**
- **ArgoCD**: GitOps continuous delivery
- **Kustomize**: Kubernetes configuration management
- **Helm**: Package manager for Kubernetes
- **cert-manager**: Automatic SSL certificate management
- **Traefik**: Modern HTTP reverse proxy and load balancer
- **PostgreSQL**: Relational database
- **nginx**: High-performance web server

### **Best Practices Demonstrated**
- **🏗️ Infrastructure as Code**: Everything defined in YAML
- **🔄 GitOps Workflow**: Git as single source of truth
- **📦 Container Security**: Non-root containers, resource limits
- **🎯 Separation of Concerns**: Static vs dynamic content
- **📈 Observability**: Health checks, monitoring endpoints
- **🔐 Security**: TLS, secrets management, namespace isolation

## 🚀 Next Steps for Students

1. **Implement Health Endpoints**: Add `/health` and `/ready` endpoints to your Flask app (see `src/health_endpoints.py` for reference)
2. **Experiment with the configuration**: Try changing resource limits, replica counts
3. **Add monitoring**: Integrate Prometheus/Grafana for metrics
4. **Implement CI/CD**: Test the complete workflow by modifying code in `src/`
5. **Add more environments**: Create staging/production overlays
6. **Database migrations**: Add init containers for database schema
7. **Backup strategies**: Implement PostgreSQL backup solutions
8. **Monitoring and alerting**: Add health checks and alert rules

## 🏥 Health Check Implementation

Your Flask application should implement these endpoints for proper Kubernetes health checking:

### **Required Endpoints**
```python
@app.route('/health')
def health_check():
    # Liveness probe - is the app running?
    return {"status": "healthy"}, 200

@app.route('/ready') 
def readiness_check():
    # Readiness probe - can the app serve traffic?
    # Should check database connectivity
    return {"status": "ready", "database": "connected"}, 200
```

### **Health Check Flow**
1. **Startup Probe**: Kubernetes waits up to 30 seconds for app to start
2. **Liveness Probe**: Kubernetes restarts container if /health fails
3. **Readiness Probe**: Kubernetes stops sending traffic if /ready fails

See `src/health_endpoints.py` for a complete implementation example.
