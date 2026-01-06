# Section 15: Cloud-Native & DevOps

> "Infrastructure that you can't recreate from code is infrastructure that will fail when you need it most."

---

## The Problem This Solves

Deployment day:
- "It works on my machine"
- SSH into production, edit files manually
- Hope nothing breaks
- Rollback means... what rollback?

This doesn't scale. 50 services, 100 servers, 10 deployments/day - manual operations become impossible and dangerous.

Cloud-native is about: automated, reproducible, scalable operations.

---

## First Principles

### Principle 1: Infrastructure as Code
If it's not in version control, it doesn't exist. Servers, networks, databases - all defined in code.

### Principle 2: Immutable Infrastructure
Don't patch servers. Destroy and recreate from clean images. Every deployment is a fresh start.

### Principle 3: Cattle, Not Pets
Servers are disposable. They have numbers, not names. When one dies, another takes its place automatically.

---

## Core Concepts

### 1. Containers (Docker)

**The Problem:**
```
Developer: "Works on my machine"
Ops: "Production has different Python version, missing libraries, different OS"
```

**The Solution:** Package everything together:

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Application code
COPY . .

# Runtime
EXPOSE 8080
CMD ["python", "main.py"]
```

```bash
# Build image (same everywhere)
docker build -t myapp:v1.0 .

# Run anywhere
docker run -p 8080:8080 myapp:v1.0
```

**What's in the container:**
```
┌─────────────────────────────────────────────┐
│              Container                       │
│  ┌───────────────────────────────────────┐  │
│  │         Application Code               │  │
│  ├───────────────────────────────────────┤  │
│  │         Dependencies (pip packages)    │  │
│  ├───────────────────────────────────────┤  │
│  │         Runtime (Python 3.11)          │  │
│  ├───────────────────────────────────────┤  │
│  │         Base OS (Debian slim)          │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 2. Container Orchestration (Kubernetes)

One container is easy. 100 containers across 50 servers?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES                                     │
│                                                                             │
│  "I want 3 replicas of order-service, always running, with these resources" │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Deployment: order-service                                          │   │
│  │  Replicas: 3                                                        │   │
│  │  Image: order-service:v2.1                                          │   │
│  │  Resources: 512MB RAM, 0.5 CPU                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│                    Kubernetes handles:                                      │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │ Scheduling       │  │ Self-healing     │  │ Scaling          │         │
│  │ Which server?    │  │ Container died?  │  │ More traffic?    │         │
│  │ K8s decides      │  │ Start new one    │  │ Add replicas     │         │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │ Load balancing   │  │ Config/Secrets   │  │ Rolling updates  │         │
│  │ Distribute       │  │ Inject without   │  │ Zero-downtime    │         │
│  │ traffic          │  │ rebuilding       │  │ deployments      │         │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3. Infrastructure as Code

**Terraform Example:**
```hcl
# Define what you want
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345"
  instance_type = "t3.medium"

  tags = {
    Name = "web-server-${count.index}"
  }
}

resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = "db.t3.medium"
  storage        = 100
}
```

```bash
# Apply to create/update
terraform apply

# Destroy everything
terraform destroy

# Same code → Same infrastructure (reproducible)
```

### 4. CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CI/CD PIPELINE                                   │
│                                                                             │
│  Git Push                                                                   │
│     │                                                                       │
│     ▼                                                                       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │    Build     │ →  │    Test      │ →  │   Security   │                  │
│  │              │    │              │    │    Scan      │                  │
│  │ docker build │    │ unit tests   │    │ vuln check   │                  │
│  │              │    │ integration  │    │ SAST/DAST    │                  │
│  └──────────────┘    └──────────────┘    └──────────────┘                  │
│                                                │                            │
│                                                ▼                            │
│                              ┌──────────────────────────────┐              │
│                              │      Deploy to Staging       │              │
│                              │      (automatic)             │              │
│                              └──────────────┬───────────────┘              │
│                                             │                              │
│                                             ▼                              │
│                              ┌──────────────────────────────┐              │
│                              │    Deploy to Production      │              │
│                              │    (manual approval)         │              │
│                              └──────────────────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5. Deployment Strategies

**Rolling Update:**
```
Start:    [v1] [v1] [v1] [v1]
Step 1:   [v2] [v1] [v1] [v1]  ← Replace one at a time
Step 2:   [v2] [v2] [v1] [v1]
Step 3:   [v2] [v2] [v2] [v1]
End:      [v2] [v2] [v2] [v2]

Pro: No extra resources needed
Con: Both versions running during rollout
```

**Blue-Green:**
```
Blue (current):  [v1] [v1] [v1] [v1]  ← Serving traffic
Green (new):     [v2] [v2] [v2] [v2]  ← Ready, not serving

Switch:
Blue (old):      [v1] [v1] [v1] [v1]  ← Standing by for rollback
Green (current): [v2] [v2] [v2] [v2]  ← Now serving traffic

Pro: Instant rollback (just switch back)
Con: 2x resources during deployment
```

**Canary:**
```
Start:    [v1] [v1] [v1] [v1]  ← 100% traffic to v1
Step 1:   [v1] [v1] [v1] [v2]  ← 1% traffic to v2 (canary)
Step 2:   [v1] [v1] [v2] [v2]  ← 10% traffic to v2
Step 3:   [v2] [v2] [v2] [v2]  ← 100% traffic to v2

Pro: Gradual rollout, catch issues early
Con: Complex routing logic
```

### 6. Service Mesh (Istio/Linkerd)

Networking logic out of applications:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WITHOUT SERVICE MESH                                │
│                                                                             │
│  ┌──────────────┐           ┌──────────────┐                               │
│  │ Service A    │           │ Service B    │                               │
│  │              │ ────────→ │              │                               │
│  │ - Retry logic│           │              │                               │
│  │ - Timeout    │           │              │                               │
│  │ - Circuit    │           │              │                               │
│  │   breaker    │           │              │                               │
│  │ - mTLS       │           │              │                               │
│  └──────────────┘           └──────────────┘                               │
│                                                                             │
│  Every service implements networking logic (duplicated)                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          WITH SERVICE MESH                                  │
│                                                                             │
│  ┌────────────────────────┐    ┌────────────────────────┐                  │
│  │  ┌──────────────┐     │    │     ┌──────────────┐   │                  │
│  │  │ Service A    │     │    │     │ Service B    │   │                  │
│  │  │ (just logic) │     │    │     │ (just logic) │   │                  │
│  │  └──────────────┘     │    │     └──────────────┘   │                  │
│  │         │             │    │            ▲           │                  │
│  │         ▼             │    │            │           │                  │
│  │  ┌──────────────┐     │    │     ┌──────────────┐   │                  │
│  │  │ Sidecar Proxy│ ────┼────┼────→│ Sidecar Proxy│   │                  │
│  │  │ - Retry      │     │    │     │ - Retry      │   │                  │
│  │  │ - Timeout    │     │    │     │ - Timeout    │   │                  │
│  │  │ - mTLS       │     │    │     │ - mTLS       │   │                  │
│  │  └──────────────┘     │    │     └──────────────┘   │                  │
│  └────────────────────────┘    └────────────────────────┘                  │
│                                                                             │
│  Networking logic centralized in mesh (consistent, observable)              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 1 (Networking)** | Load balancers, DNS, service mesh |
| **Section 12 (Distributed)** | Container orchestration |
| **Section 14 (Observability)** | Infrastructure monitoring |
| **Section 16 (Quality)** | CI/CD enforces quality |

---

## Real-World Scenarios

### Scenario 1: The Manual Deployment Disaster
**What happened:** Engineer SSH'd to production, edited config manually, typo'd database password. 3 hours downtime.
**Fix:** Config in code, automated deployment, no manual changes.

### Scenario 2: The Friday Deployment
**What happened:** Deployed Friday 5 PM. Bug discovered Saturday. No one knows how to rollback.
**Fix:** Blue-green deployment with instant rollback. Deploy Monday-Thursday only.

### Scenario 3: The Zombie Server
**What happened:** Server running for 2 years, no one knows what's on it. Afraid to turn it off.
**Fix:** Infrastructure as code. Can recreate any server from scratch.

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Write Dockerfiles
- [ ] Deploy to Kubernetes
- [ ] Write Infrastructure as Code
- [ ] Design CI/CD pipelines
- [ ] Choose deployment strategies
- [ ] Understand service meshes

---

## Seniority Challenges

### Junior Level
"What problem does Docker solve?"

### Mid Level
"Design a CI/CD pipeline for a service that: runs tests, checks security, deploys to staging automatically, and requires approval for production. Include rollback strategy."

### Senior Level
"We have 50 services, each deployed manually via SSH. No Infrastructure as Code, no centralized CI/CD, no container orchestration. Design a 12-month migration plan to Kubernetes with GitOps."

---

## Key Takeaways

1. **Infrastructure as Code** - Version control everything
2. **Containers for consistency** - Same image everywhere
3. **Orchestration for scale** - Let K8s manage containers
4. **CI/CD for safety** - Automate, test, deploy
5. **Immutable deploys** - Destroy and recreate, don't patch

---

## Prerequisites
- Section 12: Distributed (microservices context)
- Section 14: Observability (monitoring infrastructure)

## Next Section
→ [Section 16: Code Quality](./16-code-quality.md) - Building systems that last.
