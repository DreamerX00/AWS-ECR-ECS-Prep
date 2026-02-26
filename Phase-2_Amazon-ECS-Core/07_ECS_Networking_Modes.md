# 🌐 ECS Networking Modes — Complete Guide

---

## 📖 The 3 Networking Modes

```
┌──────────────────────────────────────────────────────────────┐
│           ECS NETWORKING MODES                                │
│                                                              │
│  bridge  → Docker's default bridge network (legacy)         │
│  host    → Container shares EC2 host networking directly    │
│  awsvpc  → Each task gets its own dedicated ENI (VPC)       │
└──────────────────────────────────────────────────────────────┘
```

| Mode | Fargate | EC2 | Task IP | Port Conflicts? |
|------|---------|-----|---------|-----------------|
| `bridge` | ❌ | ✅ | EC2 IP | Dynamic NAT |
| `host` | ❌ | ✅ | EC2 IP | Can conflict |
| `awsvpc` | ✅ (required) | ✅ | Own ENI | None |

---

## 🔵 bridge Mode

```
EC2 Instance
├── eth0: 10.0.1.100 (main ENI)
│
├── docker0: 172.17.0.1 (bridge network)
│   ├── Container 1: 172.17.0.2:3000  ← internal
│   ├── Container 2: 172.17.0.3:3000  ← internal
│   └── Container 3: 172.17.0.4:3000  ← internal
│
│   Port mapping: 0.0.0.0:32768→Container1:3000  (dynamic!)
│   Port mapping: 0.0.0.0:32769→Container2:3000
│   Port mapping: 0.0.0.0:32770→Container3:3000
│
└── ALB → EC2:32768 → Container1
         EC2:32769 → Container2
```

### Characteristics:
```
✅ Multiple tasks on same EC2 (different dynamic ports)
✅ Simple to setup
❌ Dynamic port mapping → harder to reason about
❌ Container IPs are private docker IPs (not VPC IPs)
❌ Security Group applies to EC2, not individual containers
❌ Not supported in Fargate
```

```json
// Task definition with bridge mode
{
  "networkMode": "bridge",
  "containerDefinitions": [{
    "name": "app",
    "portMappings": [{
      "containerPort": 3000,
      "hostPort": 0  // 0 = dynamic port assignment!
    }]
  }]
}
```

---

## 🟡 host Mode

```
EC2 Instance
├── eth0: 10.0.1.100 (main ENI)
│
├── Container 1: DIRECTLY on eth0!
│   Port 3000 → EC2's port 3000
│
├── Container 2: Would conflict on port 3000!
│   (Can't run two containers needing port 3000 on same host!)
```

### Characteristics:
```
✅ Maximum network performance (no bridge overhead)
✅ Low latency (no NAT layers)
❌ Port conflicts (can't run same-port containers on same host)
❌ No port isolation between containers
❌ Container sees host's network interfaces
❌ Not supported in Fargate
```

```json
{
  "networkMode": "host",
  "containerDefinitions": [{
    "name": "high-perf-app",
    "portMappings": [{
      "containerPort": 3000,
      "hostPort": 3000  // Must match! No dynamic mapping.
    }]
  }]
}
```

> 💡 **Use host mode** for: network-intensive apps (latency-sensitive), monitoring agents, packet capture tools.

---

## 🟢 awsvpc Mode — The Winner for Production

```
EC2 Instance
├── eth0: 10.0.1.100 (main ENI — host)
├── eth1: 10.0.1.101 (task 1's own ENI!) ← gets its own VPC IP!
└── eth2: 10.0.1.102 (task 2's own ENI!) ← different VPC IP!
     
Each Task has:
  ✅ Its own private IP address in VPC subnet
  ✅ Its own Security Group (task-level granularity!)
  ✅ Its own DNS hostname (from VPC)
  ✅ Direct L3 routing (no NAT)
```

### Why awsvpc is Production-Grade:

#### 1. Task-Level Security Groups:
```
Old bridge mode: Security Group on EC2 → applies to ALL containers on that host
awsvpc mode: Security Group on EACH TASK → granular control!

Example:
  payment-service task → SG allows: 443 from ALB only, 5432 to RDS
  user-service task    → SG allows: 443 from ALB only, 5432 to user-RDS
  
  payment-service cannot reach user-RDS (even on same EC2)!
```

#### 2. VPC Flow Logs per Task:
```
Each ENI → individual flow logs
"Which container made this suspicious connection?"
→ ENI ID → task ID → service name → developer to blame!
```

#### 3. Direct VPC Routing:
```
Container → directly on VPC network
No Docker NAT translation
No port remapping
Easy to understand routing (just like any EC2 instance)
```

### awsvpc in Fargate (MANDATORY):
```
Fargate ONLY supports awsvpc networkMode.
Each Fargate task:
  - Gets its own ENI
  - Gets its own private IP
  - Gets its own Security Group
  - Runs in YOUR VPC (your subnet, your routes)
```

---

## 📊 ENI Limits — The Hidden Constraint in EC2 Mode

```
EC2 instances have MAX ENI limits!

awsvpc mode: 1 ENI per task
EC2 t3.micro: max 2 ENIs total
  1 ENI = host/default
  1 ENI = max 1 task with awsvpc mode!
  
  → t3.micro can only run 1 awsvpc task!

EC2 c5.4xlarge: max 8 ENIs
  1 ENI = host
  7 ENIs = 7 tasks maximum!
  
This is the #1 constraint when using awsvpc mode on EC2!
```

### ENI Trunking (Solution for EC2 High Density):
```
Feature: ECS Managed Networking / Network Interface Trunking
What: Allows more tasks per instance (using "trunk ENI" technique)
Result: Up to 120 tasks per instance (instead of ENI limit!)

Requirements:
  - Specific instance types (nitro-based)
  - ECS Agent version >= 1.55.0
  - awsvpcTrunking capability on cluster

Enable:
aws ecs update-cluster-settings \
  --cluster production \
  --settings name=awsvpcTrunking,value=enabled
```

---

## 🎯 Analogy — Apartment Building 🏢

```
EC2 Instance = Apartment Building

bridge mode:
  Building has ONE main door (EC2's IP)
  Each apartment (container) gets a different room number inside
  Building desk gives directions: "Room 3000? Go to unit 204"
  → DYNAMIC PORT = "call the desk for current room numbers"
  
host mode:
  Building has ONE huge open space, no walls between apartments
  → Maximum speed, zero privacy, conflict-prone
  
awsvpc mode:
  Each apartment gets its OWN front door with its OWN address!
  "Payment Service lives at 10.0.1.101"
  "User Service lives at 10.0.1.102"
  → Direct access, individual security, clean isolation
```

---

## 🌍 Real-World Scenario

### E-commerce Platform Networking Design:
```
VPC: 10.0.0.0/16
├── Private Subnet AZ-a: 10.0.1.0/24 (ECS tasks)
├── Private Subnet AZ-b: 10.0.2.0/24 (ECS tasks)
└── Private Subnet AZ-c: 10.0.3.0/24 (ECS tasks)

Task 1 (user-service):  10.0.1.54  (awsvpc ENI)
Task 2 (user-service):  10.0.2.23  (awsvpc ENI)
Task 3 (payment-svc):   10.0.1.89  (awsvpc ENI)

Security Groups:
  sg-user-service:    Inbound: 3000 from ALB SG | Outbound: 5432 to user-RDS
  sg-payment-service: Inbound: 3000 from ALB SG | Outbound: 5432 to pay-RDS, 443 to payment-gateway

user-service task with sg-user-service:
  ✅ Receives traffic from ALB
  ✅ Can query user-RDS
  ❌ CANNOT reach payment-RDS (no rule!)
  ❌ CANNOT reach payment-gateway (no rule!)
  
payment-service task with sg-payment-service:
  ✅ Receives traffic from ALB  
  ✅ Can query payment-RDS
  ✅ Can call external payment gateway
  ❌ CANNOT reach user-RDS (no rule!)
  
ZERO CROSS-SERVICE DATABASE BLAST RADIUS!
```

---

## ⚙️ Hands-On Examples

### awsvpc Task Definition:
```json
{
  "family": "myapp",
  "networkMode": "awsvpc",   ← Must be awsvpc for Fargate!
  "containerDefinitions": [{
    "name": "app",
    "portMappings": [{
      "containerPort": 3000   ← hostPort not needed! Task has its own IP.
    }]
  }]
}

// When creating service:
aws ecs create-service \
  --cluster production \
  --service-name myapp \
  --task-definition myapp:5 \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-abc123", "subnet-def456"],
      "securityGroups": ["sg-myapp-service"],
      "assignPublicIp": "DISABLED"   ← Always DISABLED for production!
    }
  }'
```

### Find Task IP:
```bash
# Get running task IPs
aws ecs describe-tasks \
  --cluster production \
  --tasks $(aws ecs list-tasks --cluster production --service-name myapp --query 'taskArns' --output text) \
  --query 'tasks[*].{TaskArn:taskArn, IP:containers[0].networkInterfaces[0].privateIpv4Address}'
```

### Check ENI Usage per EC2 Instance:
```bash
# How many ENIs does this EC2 instance have attached?
aws ec2 describe-instances \
  --instance-ids i-abc123 \
  --query 'Reservations[0].Instances[0].NetworkInterfaces[*].{
    ENI:NetworkInterfaceId,
    IP:PrivateIpAddress,
    Description:Description
  }'
# Each ECS task ENI shows up here!
```

---

## 🚨 Gotchas & Edge Cases

### 1. awsvpc + EC2 = ENI Limit (Critical!)
```
"Task failed to start: no available ENI"
Or tasks stuck in PENDING state.

Check EC2 instance ENI limits:
aws ec2 describe-instance-types \
  --instance-types c5.xlarge \
  --query 'InstanceTypes[0].NetworkInfo.MaximumNetworkInterfaces'

Solution: Larger instances OR ENI Trunking (ECS managed networking)
```

### 2. Same Container Port — No More Conflicts in awsvpc
```
bridge mode: Can't run two containers using port 80 on same EC2 (port conflict)
awsvpc mode: Both get own IPs! task1:80 + task2:80 = NO CONFLICT
→ More flexible container density with awsvpc
```

### 3. Service Connect vs Service Discovery
```
awsvpc task IPs change every time task restarts!
"Where is user-service now? It was at 10.0.1.54, now at 10.0.2.99"

Solutions:
  1. ALB: Tasks register/deregister automatically (for external L7 traffic)
  2. Service Discovery (Cloud Map): Task registration via DNS
     user-service.local → current task IPs
  3. Service Connect: ECS-native service mesh (newest, recommended)
     HTTP/TLS proxy per task → built-in observability
```

---

## 🎤 Interview Angle

**Q: "ECS networking modes kya hai? Fargate ke liye kaunsa mandatory hai?"**

> 3 modes hain:
> bridge: Docker default bridge network, dynamic port mapping, EC2 only.
> host: Container shares host network, max performance, EC2 only.
> awsvpc: Each task gets own ENI and VPC IP. Fargate ke liye mandatory.
> 
> awsvpc = best security (task-level SGs), clean IP isolation, direct VPC routing.
> ENI limits on EC2 mode constrain max tasks per instance (ENI Trunking solves this).

**Q: "awsvpc mode mein task-level security groups kyu important hain?"**

> bridge mode mein SG EC2 level pe hota hai — same EC2 pe sab containers same SG share karte hain.
> awsvpc mein har task ka apna ENI hota hai → apna Security Group.
> Matlab payment-service aur user-service alag SGs se — even if on same EC2.
> One-service-to-another blast radius minimize: payment-service simply user-DB access nahi kar sakta (no SG rule!).
> This is fundamental to microservices security architecture.

---

*Phase 2 Complete! Next: [Phase-3_ECS-Advanced-Architecture →](../Phase-3_ECS-Advanced-Architecture/01_ECS_ALB_Integration.md)*
