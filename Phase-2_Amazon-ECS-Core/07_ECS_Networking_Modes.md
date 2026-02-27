# ECS Networking Modes — Complete Guide

---

## The 3 Networking Modes

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
| `bridge` | Not supported | Supported | EC2 IP | Dynamic NAT |
| `host` | Not supported | Supported | EC2 IP | Can conflict |
| `awsvpc` | Required | Supported | Own ENI | None |

---

## bridge Mode

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
✅ Simple to set up for legacy workloads
❌ Dynamic port mapping → harder to reason about routing
❌ Container IPs are private Docker bridge IPs (not VPC IPs)
❌ Security Group applies to the EC2 instance, not to individual containers
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
      "hostPort": 0  // 0 = dynamic port assignment
    }]
  }]
}
```

Bridge mode was the original ECS networking approach and is still functional for EC2 workloads that need maximum container density. However, the lack of per-container network identity makes it unsuitable for modern microservices architectures that require fine-grained security controls.

---

## host Mode

```
EC2 Instance
├── eth0: 10.0.1.100 (main ENI)
│
├── Container 1: DIRECTLY on eth0!
│   Port 3000 → EC2's port 3000
│
├── Container 2: Would conflict on port 3000!
│   (Cannot run two containers using port 3000 on the same host)
```

### Characteristics:
```
✅ Maximum network performance (no bridge overhead)
✅ Lowest latency (no NAT layers between container and network)
❌ Port conflicts (cannot run same-port containers on same host)
❌ No port isolation between containers on the same host
❌ Container sees all of the host's network interfaces
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

> **Use host mode for:** network-intensive applications (latency-sensitive trading, packet processing), monitoring agents that need to observe host-level network traffic, or packet capture tools. Because of port conflict risk, host mode works best with daemon services (one container per host) rather than replica services.

---

## awsvpc Mode — The Production Standard

```
EC2 Instance
├── eth0: 10.0.1.100 (main ENI — host)
├── eth1: 10.0.1.101 (task 1's own ENI) ← gets its own VPC IP!
└── eth2: 10.0.1.102 (task 2's own ENI) ← different VPC IP!

Each Task has:
  ✅ Its own private IP address in the VPC subnet
  ✅ Its own Security Group (task-level granularity)
  ✅ Its own DNS hostname (from VPC DNS)
  ✅ Direct Layer 3 routing (no NAT)
```

### Why awsvpc is the Production Standard:

#### 1. Task-Level Security Groups:
```
With bridge mode: Security Group is on the EC2 instance → applies to ALL containers on that host
With awsvpc mode: Security Group is on EACH TASK → granular per-service control

Example:
  payment-service task → SG allows: 443 inbound from ALB only, 5432 outbound to payment-RDS
  user-service task    → SG allows: 443 inbound from ALB only, 5432 outbound to user-RDS

  payment-service cannot reach user-RDS (no outbound rule to that SG!)
  This is enforced at the network level, not just application logic.
```

#### 2. VPC Flow Logs per Task:
```
Each ENI generates individual VPC Flow Logs.
When investigating suspicious traffic:
  Flow log entry → ENI ID → ECS task ID → service name → developer team

This traceability is impossible with bridge mode (all traffic appears as the EC2's IP).
```

#### 3. Direct VPC Routing:
```
Container traffic goes directly onto the VPC network.
No Docker NAT translation needed.
No dynamic port remapping.
Routing is identical to any other EC2 instance in the VPC.
Works natively with VPC endpoints, PrivateLink, and on-premises connectivity via Direct Connect.
```

### awsvpc in Fargate (Mandatory):
```
Fargate ONLY supports awsvpc networkMode.
Each Fargate task:
  - Gets its own ENI
  - Gets its own private IP address
  - Gets its own Security Group
  - Runs in YOUR VPC (your subnet, your route tables)
```

---

## ENI Limits — The Hidden Constraint in EC2 Mode

```
EC2 instances have a maximum ENI limit that varies by instance type.

awsvpc mode: 1 ENI per task
EC2 t3.micro: max 2 ENIs total
  1 ENI = host (default)
  1 ENI = max 1 awsvpc task!

  → t3.micro can only run 1 awsvpc task at a time!

EC2 c5.4xlarge: max 8 ENIs
  1 ENI = host
  7 ENIs = 7 awsvpc tasks maximum

This ENI limit is the most commonly overlooked constraint when
using awsvpc mode on EC2 instances.
```

### ENI Trunking — Solution for High Task Density on EC2:
```
Feature: ECS Managed Networking / Network Interface Trunking
What it does: Allows many more tasks per instance by using a "trunk ENI" technique
              where multiple task ENIs are multiplexed over a single physical interface
Result: Up to 120 tasks per instance (far beyond the standard ENI limit)

Requirements:
  - Nitro-based instance types
  - ECS Agent version >= 1.55.0
  - awsvpcTrunking enabled on the cluster

Enable:
aws ecs update-cluster-settings \
  --cluster production \
  --settings name=awsvpcTrunking,value=enabled
```

---

## Analogy — Apartment Building

```
EC2 Instance = Apartment Building

bridge mode:
  Building has ONE main door (the EC2's IP address)
  Each apartment (container) has a room number inside
  The building concierge redirects visitors: "Room 3000? Take the elevator to unit 204"
  → DYNAMIC PORT = "call the concierge for the current room assignments"

host mode:
  The building has one huge open floor plan — no walls between apartments
  → Maximum speed, zero privacy, containers can interfere with each other

awsvpc mode:
  Each apartment has its OWN front door with its OWN street address
  "Payment Service is at 10.0.1.101"
  "User Service is at 10.0.1.102"
  → Direct access, individual security locks (Security Groups), clean isolation
```

---

## Real-World Scenario

### E-commerce Platform Network Design:
```
VPC: 10.0.0.0/16
├── Private Subnet AZ-a: 10.0.1.0/24 (ECS tasks)
├── Private Subnet AZ-b: 10.0.2.0/24 (ECS tasks)
└── Private Subnet AZ-c: 10.0.3.0/24 (ECS tasks)

Task 1 (user-service):  10.0.1.54  (awsvpc ENI)
Task 2 (user-service):  10.0.2.23  (awsvpc ENI)
Task 3 (payment-svc):   10.0.1.89  (awsvpc ENI)

Security Groups:
  sg-user-service:    Inbound: 3000 from ALB SG | Outbound: 5432 to user-RDS SG
  sg-payment-service: Inbound: 3000 from ALB SG | Outbound: 5432 to pay-RDS SG, 443 to payment-gateway

user-service task with sg-user-service:
  ✅ Receives traffic from ALB
  ✅ Can query user-RDS
  ❌ CANNOT reach payment-RDS (no outbound rule)
  ❌ CANNOT reach payment-gateway (no outbound rule)

payment-service task with sg-payment-service:
  ✅ Receives traffic from ALB
  ✅ Can query payment-RDS
  ✅ Can call external payment gateway
  ❌ CANNOT reach user-RDS (no outbound rule)

ZERO CROSS-SERVICE DATABASE BLAST RADIUS — enforced at the network layer!
```

---

## Hands-On Examples

### awsvpc Task Definition:
```json
{
  "family": "myapp",
  "networkMode": "awsvpc",   // Required for Fargate; recommended for EC2
  "containerDefinitions": [{
    "name": "app",
    "portMappings": [{
      "containerPort": 3000   // hostPort is not needed — task has its own IP
    }]
  }]
}

// When creating the service:
aws ecs create-service \
  --cluster production \
  --service-name myapp \
  --task-definition myapp:5 \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-abc123", "subnet-def456"],
      "securityGroups": ["sg-myapp-service"],
      "assignPublicIp": "DISABLED"   // Always DISABLED for production (use NAT Gateway)
    }
  }'
```

### Find Task IP Addresses:
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
# Each ECS task ENI appears as a separate entry here
```

---

## Gotchas & Edge Cases

### 1. awsvpc + EC2 = ENI Limit (Critical)
```
Symptom: Tasks are stuck in PENDING state; no placement errors in the console
OR:      "Task failed to start: no available ENI"

Check the ENI limit for your instance type:
aws ec2 describe-instance-types \
  --instance-types c5.xlarge \
  --query 'InstanceTypes[0].NetworkInfo.MaximumNetworkInterfaces'

Solutions:
  1. Use larger instances (more ENIs available)
  2. Enable ENI Trunking (up to 120 tasks per instance)
  3. Switch to Fargate (no ENI limit concern)
```

### 2. awsvpc Eliminates Port Conflicts That Plague bridge Mode
```
bridge mode: Cannot run two containers using port 80 on the same EC2 (port conflict)
awsvpc mode: Each task gets its own IP — task1:80 and task2:80 coexist without conflict

This allows you to standardize all services on port 80 or 3000 internally,
simplifying task definition management and making services easier to reason about.
```

### 3. Task IPs Change on Every Restart — Service Discovery Is Required
```
awsvpc task IPs change every time a task is replaced or redeployed.
"Where is user-service? It was at 10.0.1.54, now it is at 10.0.2.99"

Solutions:
  1. ALB: Tasks register and deregister automatically (for external L7 traffic)
  2. Service Discovery (Cloud Map):
       Task registrations are published to Route 53 DNS automatically
       user-service.local → current task IPs (round-robin)
  3. ECS Service Connect (recommended for new workloads):
       ECS-native service mesh; HTTP/TLS proxy is injected per task
       Built-in retry logic, circuit breaking, and observability
       Services reference each other by logical name, not IP
```

### 4. assignPublicIp Considerations
```
assignPublicIp: "ENABLED"
  → Task gets a public IP — can reach the internet directly without NAT
  → Security risk: task is directly reachable from the internet (mitigated by SG)
  → Useful for development environments to avoid NAT Gateway cost
  → NOT recommended for production

assignPublicIp: "DISABLED" (production standard)
  → Task uses NAT Gateway or VPC endpoint for outbound internet/AWS API access
  → No inbound internet access; traffic enters via ALB only
  → NAT Gateway costs ~$0.045/hour per AZ plus data processing fees
  → Use VPC endpoints for ECR, Secrets Manager, CloudWatch to reduce NAT traffic
```

### 5. VPC Endpoints Reduce Costs and Improve Security in awsvpc Mode
```
Without VPC endpoints: All ECR pulls, Secrets Manager calls, CloudWatch logs
  go out through NAT Gateway → internet → back into AWS
  → NAT Gateway data processing charges add up significantly at scale

With VPC endpoints (Gateway or Interface endpoints):
  ECR traffic stays within the VPC
  Secrets Manager calls stay within the VPC
  CloudWatch log writes stay within the VPC
  → Reduced NAT costs + traffic never leaves the AWS network

Recommended endpoints for ECS workloads:
  com.amazonaws.<region>.ecr.api
  com.amazonaws.<region>.ecr.dkr
  com.amazonaws.<region>.s3 (for ECR layer storage)
  com.amazonaws.<region>.secretsmanager
  com.amazonaws.<region>.logs
  com.amazonaws.<region>.ssm
```

---

## Interview Questions

**Q: "What are the ECS networking modes? Which one is mandatory for Fargate?"**

> ECS has three networking modes:
> `bridge` — Docker's default bridge network with dynamic port mapping. EC2 only. Legacy; not recommended for new workloads.
> `host` — Container shares the host's network interface directly. EC2 only. Used for maximum network performance or network-level monitoring tools.
> `awsvpc` — Each task receives its own ENI and a dedicated VPC IP address. Supported on both EC2 and Fargate; required for Fargate.
>
> `awsvpc` is the recommended mode for all production workloads because it provides task-level Security Groups, direct VPC routing, individual flow log traceability, and no port conflicts between tasks.

**Q: "Why are task-level Security Groups in awsvpc mode important for microservices security?"**

> In bridge mode, the Security Group is applied at the EC2 instance level, meaning all containers on that instance share the same Security Group rules. You cannot restrict one service from reaching another service's database at the network layer.
>
> In awsvpc mode, each task has its own ENI and its own Security Group. This means payment-service and user-service can have completely separate Security Group rules — even if they run on the same EC2 instance. If payment-service has no outbound rule to user-service's database security group, that network path is blocked at the VPC level regardless of what the application code tries to do.
>
> This network-layer isolation is the foundation of a zero-trust microservices architecture in AWS. It limits the blast radius of a compromised service to only the resources explicitly permitted by its Security Group rules.

---

*Phase 2 Complete! Next: [Phase-3_ECS-Advanced-Architecture →](../Phase-3_ECS-Advanced-Architecture/01_ECS_ALB_Integration.md)*
